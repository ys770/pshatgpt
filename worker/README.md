# PshatGPT Proxy (Cloudflare Worker)

A thin serverless proxy that lets users of the [PshatGPT](https://ys770.github.io/PshatGPT/)
web UI get explanations **without needing their own Anthropic API key**. The
proxy uses the owner's key server-side, caps free use per IP, and streams
responses straight through from Anthropic.

## What it does

- Accepts `POST /v1/messages` from the browser, shaped exactly like
  Anthropic's `/v1/messages` endpoint.
- Rate-limits by IP via Cloudflare KV (default: 10 requests / IP / day, UTC).
- Forces `stream: true` and caps `max_tokens` to 2048.
- Restricts `model` to an allow-list (Sonnet / Haiku 4.5).
- Streams the SSE response back to the browser unchanged, with CORS headers
  for your allowed origins.

Cloudflare's free tier allows 100,000 requests/day — plenty for this use case.

## Setup

Prerequisites: [Node.js](https://nodejs.org) + a free Cloudflare account.

```bash
cd worker
npm install
npx wrangler login            # opens browser, authorizes CLI to your CF account
```

### 1. Create the KV namespace (rate-limit storage)

```bash
npx wrangler kv namespace create RATE_LIMIT
```

Copy the returned `id` into `wrangler.toml` where it says
`PUT_YOUR_KV_NAMESPACE_ID_HERE`.

### 2. Set your Anthropic API key as a secret

```bash
npx wrangler secret put ANTHROPIC_API_KEY
# paste your sk-ant-... when prompted
```

### 3. Deploy

```bash
npx wrangler deploy
```

You'll get a URL like `https://pshatgpt-proxy.<your-subdomain>.workers.dev`.

### 4. Wire the frontend

Open `../docs/app.js` and set the proxy URL constant at the top:

```js
const PROXY_URL = "https://pshatgpt-proxy.<your-subdomain>.workers.dev";
```

Commit + push — GitHub Pages redeploys automatically.

## Configuring limits and origins

Edit `wrangler.toml`:

- `FREE_TIER_DAILY` — per-IP daily request cap (default `"10"`)
- `ALLOWED_ORIGINS` — comma-separated list of origins allowed to call the
  proxy. Use `"*"` for open access, or lock down to your deployed URL.

After editing, redeploy: `npx wrangler deploy`.

## Local dev

```bash
npx wrangler dev
```

Runs at `http://localhost:8787`. Test with:

```bash
curl http://localhost:8787/health
```

## Disabling / deleting

```bash
npx wrangler delete                  # removes the worker
npx wrangler kv namespace delete --namespace-id <id>   # removes rate-limit data
```

## Cost

- Cloudflare Workers: **free** under 100K req/day.
- Anthropic API: **you pay for every call**. For Sonnet 4.5, explanations
  typically cost $0.01–$0.03 each. At 10 free requests × ~50 users/day that's
  ~$5–15/month. Tune `FREE_TIER_DAILY` to control your budget.

## Abuse prevention

Built-in:
- Model allow-list (can't burn through Opus pricing)
- Token cap (2048 max output)
- Per-IP daily cap
- Forced streaming (no batch abuse)

Cloudflare DDoS protection is automatic. For extra safety:
- Add Cloudflare Turnstile in front of the proxy
- Lower `FREE_TIER_DAILY` if you see abuse
- Restrict `ALLOWED_ORIGINS` to just your production URL
