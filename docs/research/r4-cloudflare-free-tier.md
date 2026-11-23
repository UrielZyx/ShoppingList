# R4: Does Cloudflare's free plan cover this app's edge needs?

Date of research: 2026-11-23. Status: MOSTLY VERIFIED against Cloudflare primary docs fetched today. Items still unconfirmed are marked UNVERIFIED.
Page dates: only the connection-limits page showed one (last updated 2026-11-13). The other pages showed no date; they were read live on 2026-11-23.

## Verified facts

| Topic | Verified fact | Source |
|---|---|---|
| Free plan scope | $0/month, "for personal or hobby projects that aren't business-critical". Includes DNS, CDN, unmetered DDoS protection, Universal SSL, free managed WAF ruleset. No uptime SLA. | https://www.cloudflare.com/plans/ |
| Bandwidth cap | The plans page lists no bandwidth cap for Free (absence on the page, not an explicit "unlimited" statement). | https://www.cloudflare.com/plans/ |
| Proxy (orange cloud) | Proxied: DNS returns Cloudflare anycast IPs, HTTP(S) goes through Cloudflare, origin IP hidden. DNS-only (grey): DNS returns origin IP, no HTTP(S) routing through Cloudflare, origin IP exposed. If any A/AAAA on a name is proxied, all are treated as proxied. | https://developers.cloudflare.com/dns/proxy-status/ |
| Universal SSL | Available on Free; DV certs, issuance, renewal and deployment automatic once the domain is active on Cloudflare. HTTP/2 and HTTP/3 need a Cloudflare-served cert. | https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/ , proxy-status page above |
| Proxy read timeout (524) | **125 s** default (my earlier recollection of ~100 s was wrong). Only Enterprise zones can change it (up to 6000 s). So NOT adjustable on Free. Write timeout 30 s, not adjustable. | https://developers.cloudflare.com/fundamentals/reference/connection-limits/ ; https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-524/ |
| Other proxy-to-origin limits | TCP connect 19 s (522); TCP request ack 90 s (522); TCP keep-alive interval 30 s (520); **Proxy Idle Timeout 900 s (520), not configurable**; HTTP/2 connection idle 900 s. | connection-limits page |
| Client-side | HTTP/1.1 keep-alive and HTTP/2 idle: 400 s, not configurable. | connection-limits page |
| Header/URL limits | URL 16 KB; request headers 128 KB total; response headers 128 KB total. | connection-limits page |
| Request body | Max upload 100 MB on Free (Pro 100 MB, Business 200 MB, Enterprise up to 5 GB); can be lowered in zone settings. | https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/ |
| 524 workarounds | Poll for status instead of one long request; move long requests to a DNS-only subdomain. The page does not mention heartbeats. | error-524 page |
| Static hosting: Workers static assets | Requests to static assets are free and unlimited. Free limits: 20,000 files per Worker version, 25 MiB per file. Range requests return 200 with the full file (no 206) unless routed through a Worker. Worker invocations (including `run_worker_first`) count against Free quota: 100,000 requests/day, 10 ms CPU, 50 subrequests; over quota gives Error 1027 / 429. | https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/ ; https://developers.cloudflare.com/workers/platform/limits/ |
| Static hosting: Pages | Free: 20,000 files per site, 25 MiB per file, 500 builds/month, 1 concurrent build, 20 min build timeout, 100 projects per account. Page shows no deprecation notice but links a "Migrate to Workers" guide and recommends Workers Static Assets for scale. | https://developers.cloudflare.com/pages/platform/limits/ |
| Registrar pricing | At-cost: "does not mark up domain prices at all", no add-on fees, renewals at-cost. 390+ TLDs. Page gives no example prices and no transfer fee/wait rules. | https://www.cloudflare.com/products/registrar/ |
| Terms, CDN clause | Free/Pro/Business may use the CDN to cache and serve web pages/sites. Serving video or a disproportionate share of large files (pictures, audio, other large files) requires a paid service; Cloudflare may restrict access. Page says nothing about SSE, WebSockets, streaming or API use. | https://www.cloudflare.com/service-specific-terms-application-services/ |

## SSE through the proxy (derived from the verified limits)
- The 125 s read timeout is a wait for response bytes from the origin; the 900 s Proxy Idle Timeout is also fixed. An SSE stream that sends an event or a comment line (`:ping`) every 15-30 s stays far below both. Neither can be raised on Free, so heartbeats are required for quiet streams.
- Cloudflare's docs read today do not say explicitly how an indefinitely open streaming response is treated, so "long-lived SSE works with heartbeats" is a reasoned inference. UNVERIFIED by a primary source: that response buffering is off for `text/event-stream`, and whether Cloudflare ever closes long-lived streams during edge restarts/deploys (clients need auto-reconnect with `Last-Event-ID` regardless).
- The terms' "large files" clause targets media, not SSE; low-volume event text should not trigger it, but this is interpretation.

## Backend container on a VM, CLI client
- Free proxy handles any origin reachable on the proxied HTTP(S) ports; the container on a VM is just an origin. The CLI is an ordinary HTTPS client and has no Cloudflare dependency, but it will also be subject to the 125 s/900 s/100 MB limits and Cloudflare bot/WAF challenges (UNVERIFIED for this app: whether the free managed WAF ever challenges a CLI user agent; test it).
- Origin-side TLS (Full/Strict mode, origin certificates) was not fetched. UNVERIFIED.

## Runtime coupling (design reasoning plus verified background)
- Proxied mode makes Cloudflare a runtime dependency: outage, 52x errors, or a 524 reach users. The origin sees Cloudflare IPs; real client IP comes via the CF-Connecting-IP header (UNVERIFIED this session; from memory).
- DNS-only mode removes the proxy, HTTP/2 and HTTP/3 via Cloudflare certs, DDoS shielding and origin-IP hiding (all verified above); the origin would need its own certificate.
- Static assets via Workers: no runtime coupling beyond hosting if no Worker code is used; avoid `run_worker_first` so the 100k/day Worker quota is not on the request path.
- Keep portable: SSE heartbeat every 15-30 s, no Workers APIs or CF-specific headers in the backend, SPA deployable to any static host.
- Registrar: domains bought there are tied to Cloudflare DNS nameservers by default (UNVERIFIED; not on the fetched page). Transfer-out terms unverified.

## Conclusion
Yes for a personal app: Free covers DNS, proxy, Universal TLS, a static SPA/PWA (free unlimited static asset requests, 25 MiB per file), a proxied container origin, SSE with heartbeats, and a CLI client. Key risks: fixed 125 s read / 900 s idle timeouts on Free (heartbeats mandatory); 100 MB upload cap; Free is "hobby, not business-critical" with no SLA; Pages-to-Workers steering; Range requests on static assets lack 206; edge-proxy coupling at runtime.

## Still unverified
Registrar example prices and transfer rules; SSE buffering behaviour; origin TLS modes; CF-Connecting-IP; WAF behaviour toward a CLI; explicit Free bandwidth statement; Registrar nameserver lock-in.
