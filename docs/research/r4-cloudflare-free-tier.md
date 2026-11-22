# R4: Does Cloudflare's free plan cover this app's edge needs?

Date of research: 2026-11-23. Status: INCOMPLETE. No primary source was read.

## What happened
All five WebFetch calls to Cloudflare pages were refused by the permission layer
("user did not answer an earlier permission request in time"). Per the tool's
instruction I did not retry. A WebSearch returned only page titles and URLs, no
content. So **nothing below is verified against today's documentation**.

Blocked URLs (read these next):
- https://developers.cloudflare.com/fundamentals/reference/connection-limits/
- https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-524/
- https://www.cloudflare.com/plans/
- https://developers.cloudflare.com/workers/platform/limits/
- https://www.cloudflare.com/products/registrar/
Also surfaced by search, unread: https://developers.cloudflare.com/dns/proxy-status/

## Unverified recollection (training knowledge, not a source; confirm each)
| Topic | Recollection | Confirm at |
|---|---|---|
| DNS, proxy (orange cloud), universal TLS on Free | Included, no bandwidth charge | plans page |
| Proxy response timeout | About 100 s without response bytes gives 524; not adjustable on Free | error-524 page |
| SSE through proxy | Works if the origin sends data (comments or heartbeats) more often than the idle limit; response buffering/caching must be off | connection-limits page |
| Request body limit | About 100 MB on Free | connection-limits page |
| Static hosting | Workers static assets / Pages exist with a free tier; Pages is being steered toward Workers | workers limits page, Pages docs |
| Registrar | At-cost wholesale pricing, no markup | registrar page |

## Runtime coupling (design reasoning, not sourced)
- Proxied mode makes Cloudflare a runtime dependency: its outage or a 524 hits users, and the origin sees CF-Connecting-IP, not the client IP.
- DNS-only mode (grey cloud) removes proxy coupling but also removes the proxy and its TLS; the origin would then need its own certificate.
- Keep the app portable: heartbeat SSE every ~15-30 s, no CF-specific headers or Workers APIs in the backend, SPA deployable to any static host.

## Conclusion (provisional)
Likely sufficient for one container, a static SPA and a few users, but the SSE timeout and current free-tier limits must be confirmed from the blocked pages before this is relied on.
