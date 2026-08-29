# Coolify GitHub Webhook Proxy

Minimal Caddy-based proxy for exposing Coolify GitHub App webhook endpoints through Tailscale Funnel.

## Allowed routes

- `POST /webhooks/source/github/events`
- `GET /webhooks/source/github/install*`
- `GET /webhooks/source/github/redirect`

Everything else returns `404`.

## Protections

- Rate limiting on webhook events
- Request body size limit
- Header size limit
- Exact path and HTTP method filtering

## Build

```bash
docker build -t coolify-github-webhook-proxy .
