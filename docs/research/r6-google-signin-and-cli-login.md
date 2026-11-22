# R6: Google sign-in (OIDC) for a small personal app, and CLI login

Researched 2026-11-23. Issue: UrielZyx/ShoppingList #7.

**Verification status matters.** Several Google pages were blocked by a permission timeout in this run. Claims are tagged:
- VERIFIED = read from the cited page in this run.
- UNVERIFIED = from background knowledge or search-result titles only; re-check before relying on it.

## Blocked pages (not read)
- https://developers.google.com/identity/protocols/oauth2/limited-input-device (device flow)
- https://developers.google.com/identity/protocols/oauth2/native-app (loopback / installed apps)
- https://support.google.com/cloud/answer/15549945 (Manage App Audience; only seen as a search result title)
- https://developers.google.com/identity/openid-connect/openid-connect

## 1. Publishing status, user caps, token lifetimes
Source: https://developers.google.com/identity/protocols/oauth2/production-readiness/overview (page last updated 2026-05-22). VERIFIED.
- **Testing (External):** only allowlisted test users can sign in, up to 100 test users. The page states an exception for apps that request only basic identity scopes (openid, email, profile). Users see a testing warning.
- **Testing refresh tokens expire after 7 days.** A Workspace admin marking the app "Trusted" overrides this, and the 100 cap, for that org only.
- **Published, unverified (External):** any Google user can sign in. Name and logo are not shown on the consent screen. Sensitive or restricted scopes show a danger warning and the app is capped at 100 users.
- **Published, verified:** verification is required for public apps that request sensitive or restricted scopes. Brand verification is needed for the app name and logo to show.
- **Internal:** Workspace org users only, no verification.

Interpretation (mine): for a handful of users, 100 is no constraint. The practical problem with Testing is the 7-day refresh-token expiry (re-login weekly), if refresh tokens are used at all. Note that the page's "basic scopes exempt in Testing" wording is ambiguous about whether the 7-day expiry also applies to them. Check by testing.

## 2. Is verification needed for openid/email/profile?
- Source: https://support.google.com/cloud/answer/13464323 (no date on page). VERIFIED. Verification is not needed for "Limited Audience" apps (fewer than 100 users), personal-mode apps, dev/test apps, and internal apps. This page does not mention the basic scopes. It says the 100-user cap applies to unverified apps in testing/dev modes.
- Combined with the production-readiness page above, which says verification is required for sensitive and restricted scopes: openid, email and profile are non-sensitive scopes (UNVERIFIED classification; the Google scope list was not read). So a published app using only those should not need verification. Publishing to Production with only basic scopes is the usual way to avoid the 7-day expiry (UNVERIFIED; confirm in the Cloud console, Google Auth Platform > Audience).

## 3. Cost
UNVERIFIED (no page read on this). Google Sign-In / OAuth client creation is free in Google Cloud; no billing account is needed for an OAuth client. Verify on the Google Auth Platform pages before committing.

## 4. How a CLI can authenticate
All UNVERIFIED (pages blocked); recollection of Google docs:
- **Device flow (limited-input device):** CLI shows a code and URL, user approves in a browser elsewhere. Google restricts scopes to a small list: openid, email, profile, plus a few like Drive file and YouTube. The client type is "TVs and Limited Input devices". Quotas apply to device-code requests. openid/email/profile are on the allowed list. Needs a client secret (not truly confidential for a distributed CLI).
- **Loopback redirect (installed/desktop app):** CLI starts http://127.0.0.1:<port>, opens the browser, uses Authorization Code + PKCE. Google no longer supports the out-of-band (OOB) copy-paste flow. Desktop clients still get a client secret that Google says is not treated as secret. Works with any scope. Needs a local browser on the same machine.
- **Via the web app:** the web app signs the user in, then issues its own session/API token (or a short-lived one-time code the CLI exchanges, e.g. `cli login` prints a URL, web page shows a pairing code). This keeps Google out of the CLI entirely and is provider-agnostic. Recommended for the "interface not baked to Google" goal.

## 5. Other OIDC providers behind the same interface
Standard OIDC means: issuer URL, discovery at /.well-known/openid-configuration, authorization-code + PKCE, ID-token validation via JWKS. Providers (UNVERIFIED, general knowledge; none read in this run): Microsoft Entra ID, Apple, GitHub (OAuth2, not full OIDC for user login, though Actions has OIDC), Auth0, Okta, Keycloak/Authentik (self-hosted), Cloudflare Access, Auth.js/Better Auth wrappers. Design: configure by issuer + client id/secret; key users by (iss, sub), never by email alone.

## Conclusions and risks
1. Google fits: a few users is far below caps, and basic scopes should not need verification.
2. Risk: Testing status gives 7-day refresh tokens; publish to Production or avoid long-lived Google refresh tokens by issuing the app's own session tokens.
3. Risk: device-flow/loopback specifics and cost are unverified here.
4. Prefer web-app-mediated CLI login; keep an issuer-configurable OIDC adapter.
