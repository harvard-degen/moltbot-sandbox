# OpenClaw on Cloudflare Workers with Moonshot Kimi K2.5

Step-by-step guide to deploy OpenClaw (Moltbot) on Cloudflare Workers using Moonshot's Kimi K2.5 as the AI provider. This fork adds OpenAI-compatible provider support (Kimi, Ollama, etc.) alongside the original Anthropic support.

## Prerequisites

- **Cloudflare account** with [Workers Paid plan](https://www.cloudflare.com/plans/developer-platform/) ($5/month)
- **Moonshot API key** from [Moonshot AI Platform](https://platform.moonshot.cn/)
- **Docker Desktop** installed and running (required for building container images)
- **Node.js** v18+ and npm
- **Cloudflare API token** with Workers permissions (create at [API Tokens](https://dash.cloudflare.com/profile/api-tokens) using the "Edit Cloudflare Workers" template)

## Step 1: Clone and Install

```bash
git clone https://github.com/harvard-degen/moltbot-sandbox.git
cd moltbot-sandbox
npm install
```

## Step 2: Set Up Cloudflare Zero Trust (Access)

This protects your deployment with email-based authentication.

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Choose a **team name** (e.g., `yourteam`) if prompted to create one
3. Navigate to **Access** > **Applications** > **Add an application**
4. Select **Self-hosted**, name it (e.g., `moltworker`)
5. Set the **Application domain** to your worker URL: `your-worker-name.your-subdomain.workers.dev`
6. Create an **Access Policy**: Allow emails matching your team's addresses
7. Save and copy the **Application Audience (AUD) tag** from the application's overview page

## Step 3: Set Secrets

Use your Cloudflare API token to authenticate wrangler. Replace all placeholder values below.

```bash
export CLOUDFLARE_API_TOKEN="your-cloudflare-api-token"

# --- Required: Cloudflare Access ---
echo "yourteam.cloudflareaccess.com" | npx wrangler secret put CF_ACCESS_TEAM_DOMAIN
echo "your-aud-tag-from-step-2" | npx wrangler secret put CF_ACCESS_AUD

# --- Required: Gateway Token ---
# Generate a random token or use any secure string. Save this - you'll need it to access the UI.
export MOLTBOT_GATEWAY_TOKEN=$(openssl rand -hex 32)
echo "Your gateway token: $MOLTBOT_GATEWAY_TOKEN"
echo "$MOLTBOT_GATEWAY_TOKEN" | npx wrangler secret put MOLTBOT_GATEWAY_TOKEN

# --- Required: Kimi K2.5 AI Provider ---
echo "your-moonshot-api-key" | npx wrangler secret put OPENAI_API_KEY
echo "https://api.moonshot.ai/v1/openai" | npx wrangler secret put AI_GATEWAY_BASE_URL

# --- Recommended: R2 Persistent Storage ---
# Without R2, chat history and device pairings are lost on container restart.
# Create an R2 API token at: R2 > Overview > Manage R2 API Tokens
# Permissions: Object Read & Write, scoped to the moltbot-data bucket
echo "your-r2-access-key-id" | npx wrangler secret put R2_ACCESS_KEY_ID
echo "your-r2-secret-access-key" | npx wrangler secret put R2_SECRET_ACCESS_KEY
echo "your-cloudflare-account-id" | npx wrangler secret put CF_ACCOUNT_ID

# --- Optional: Debug routes (useful for troubleshooting) ---
echo "true" | npx wrangler secret put DEBUG_ROUTES
```

**Finding your Cloudflare Account ID:** Dashboard home page > click three-dot menu next to your account > "Copy Account ID".

## Step 4: Deploy

Make sure Docker Desktop is running, then:

```bash
npx wrangler deploy
```

First deploy takes 3-5 minutes (downloads base image, installs clawdbot in container, pushes to Cloudflare registry). Subsequent deploys are faster due to Docker layer caching.

## Step 5: Access the UI

1. Visit `https://your-worker-name.your-subdomain.workers.dev/?token=YOUR_GATEWAY_TOKEN`
2. Authenticate via Cloudflare Access (email OTP)
3. Wait for the loading page (container cold start takes 1-2 minutes on first visit)
4. You should see the OpenClaw chat UI

### Device Pairing (first time only)

After the gateway starts, you need to pair your browser:

1. Visit `/_admin/` (authenticated via Cloudflare Access)
2. Approve your pending device
3. Return to the main chat UI and refresh

## How It Works

The key modification in this fork is in `start-moltbot.sh`. When `AI_GATEWAY_BASE_URL` ends in `/openai`, it:

1. Strips the `/openai` suffix to get the actual API endpoint (`https://api.moonshot.ai/v1`)
2. Configures the OpenAI provider with `api: 'openai-completions'` (Chat Completions format)
3. Registers `kimi-k2.5` as the available model (262K context window)
4. Injects the `OPENAI_API_KEY` into the provider config
5. Sets `openai/kimi-k2.5` as the primary model

The `src/index.ts` validation is also updated to accept `OPENAI_API_KEY` as an alternative to `ANTHROPIC_API_KEY`.

## Using a Different OpenAI-Compatible Provider

To use a different provider (e.g., DeepSeek, Qwen, local Ollama), modify `start-moltbot.sh` lines 225-230:

```javascript
const openaiProviderConfig = {
    baseUrl: openaiBaseUrl,
    api: 'openai-completions',  // use 'openai-responses' for OpenAI's newer API
    models: [
        { id: 'your-model-id', name: 'Display Name', contextWindow: 128000 },
    ]
};
```

Then update lines 238-239 to match:
```javascript
config.agents.defaults.models['openai/your-model-id'] = { alias: 'Display Name' };
config.agents.defaults.model.primary = 'openai/your-model-id';
```

Set the appropriate `OPENAI_API_KEY` and `AI_GATEWAY_BASE_URL` secrets.

### Supported `api` values

| Value | Format | Use for |
|-------|--------|---------|
| `openai-completions` | Chat Completions (`/v1/chat/completions`) | Moonshot Kimi, DeepSeek, Qwen, Ollama, most OpenAI-compatible APIs |
| `openai-responses` | Responses API (`/v1/responses`) | OpenAI direct, Azure OpenAI |
| `anthropic-messages` | Messages API | Anthropic Claude direct |
| `google-generative-ai` | Generative AI | Google Gemini |

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `Configuration Required` page | Missing secrets. Check `npx wrangler secret list` |
| `Unauthorized - Cloudflare Access session invalid` | `CF_ACCESS_TEAM_DOMAIN` must include `.cloudflareaccess.com` (e.g., `yourteam.cloudflareaccess.com`) |
| `Invalid or missing token` | Append `?token=YOUR_GATEWAY_TOKEN` to the URL |
| Gateway fails to start with config error | Check `npx wrangler tail your-worker-name` for the specific validation error |
| Chat sends but no response | Check your Moonshot API key has credits. Test with: `curl https://api.moonshot.ai/v1/models -H "Authorization: Bearer YOUR_KEY"` |
| Docker not found during deploy | Install Docker Desktop and ensure the daemon is running (`docker info`) |
| `Durable Object reset` in logs | Normal after deploy. The container restarts with the new image. |

## All Secrets Quick Reference

| Secret | Required | Value |
|--------|----------|-------|
| `CF_ACCESS_TEAM_DOMAIN` | Yes | `yourteam.cloudflareaccess.com` |
| `CF_ACCESS_AUD` | Yes | AUD tag from your Access application |
| `MOLTBOT_GATEWAY_TOKEN` | Yes | Random hex string (`openssl rand -hex 32`) |
| `OPENAI_API_KEY` | Yes* | Your Moonshot API key |
| `AI_GATEWAY_BASE_URL` | Yes* | `https://api.moonshot.ai/v1/openai` |
| `R2_ACCESS_KEY_ID` | Recommended | From R2 API token |
| `R2_SECRET_ACCESS_KEY` | Recommended | From R2 API token |
| `CF_ACCOUNT_ID` | Recommended | Your Cloudflare account ID |
| `DEBUG_ROUTES` | Optional | `true` to enable debug endpoints |

*Or use `ANTHROPIC_API_KEY` instead for Claude.
