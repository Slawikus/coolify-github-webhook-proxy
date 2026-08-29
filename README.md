# Coolify GitHub Webhook Proxy

Minimal Caddy-based proxy for exposing only the Coolify GitHub App endpoints that must be reachable through Tailscale Funnel. The Coolify dashboard and other endpoints can remain private.

## Current setup

Tailscale Funnel provides the public HTTPS endpoint and forwards requests to this proxy over HTTP on port 80. Caddy then forwards accepted requests to `coolify:8080`, so both containers must share a Docker network on which `coolify` resolves to the Coolify container.

The custom image uses:

- Caddy `2.11.4`
- [`mholt/caddy-ratelimit`](https://github.com/mholt/caddy-ratelimit) pinned to commit `5625512f24f6f59d6f64fb3aafe5eecff0b286db`

Build the image with:

```bash
docker build -t coolify-github-webhook-proxy .
```

## Allowed routes

- `POST /webhooks/source/github/events`
- `GET /webhooks/source/github/install`
- `GET /webhooks/source/github/redirect`

Everything else returns `404 Not Found` and is not forwarded to Coolify.

## Security model

The proxy applies several narrow controls before a request reaches Coolify:

- Only the routes and HTTP methods listed above are forwarded.
- Webhook event requests are limited to 30 requests per one-minute window, keyed by Caddy's `{remote_host}` value.
- Webhook event request bodies are limited to 5 MB.
- Request headers are limited to 64 KB for the server.
- All unmatched requests receive a `404` response.

This proxy is an additional exposure-reduction layer, not the webhook authentication boundary. Coolify must still be kept current and configured with a strong GitHub webhook secret so it can verify webhook signatures. TLS is expected to terminate at Tailscale Funnel; Caddy intentionally listens on plain HTTP port 80 behind it. Ensure that port 80 is not exposed directly to the public internet.

## Possible extensions

### Restrict webhook events to GitHub hook IP ranges

The event route could also be restricted to the CIDRs in the `hooks` field returned by GitHub's [`GET /meta`](https://api.github.com/meta) endpoint. Those ranges can change and would need to be kept current.

Verify the actual client address visible to Caddy before enabling this restriction. Tailscale Funnel may obscure the original source IP, in which case Caddy may see a Funnel relay or proxy address instead of a GitHub address. An IP allowlist would then require correctly configured trusted-proxy and client-IP handling, using only headers supplied by a proxy that Caddy explicitly trusts. Do not trust client-supplied forwarding headers indiscriminately.

### Revisit the rate-limit key behind Funnel

The current rate limit uses `key {remote_host}`. Behind Tailscale Funnel, that value may be a Funnel relay or proxy address rather than the original GitHub source IP. If all webhook deliveries appear to come from the same address, the 30-requests-per-minute limit is effectively global, not per GitHub source IP, and legitimate bursts could be throttled together.

Before changing the key, confirm what address and trusted forwarding metadata reach Caddy. Any forwarded client-IP value should be accepted only from an explicitly trusted proxy path.
