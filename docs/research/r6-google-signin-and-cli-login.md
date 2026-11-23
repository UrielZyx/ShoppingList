# R6: Google sign-in (OIDC) for a small personal app, and CLI login

Researched 2026-11-23. Issue: UrielZyx/ShoppingList #7.

**Verification status matters.** Several Google pages were blocked by a permission timeout in this run. Claims are tagged:
- VERIFIED = read from the cited page in this run.
- UNVERIFIED = from background knowledge or search-result titles only; re-check before relying on it.

## Fetch status (second pass, 2026-11-23)
Read OK: device-flow, native-app, openid-connect (all "last updated 2026-11-19"), support 15549945 (no date), production-readiness (2026-05-22), support 13464323 (no date), Microsoft OIDC page (2026-06-30), GitHub authorizing-OAuth-apps (no date).
Not read: Apple Sign in with Apple REST page (fetch failed, client_error; retry not made). Note: WebFetch returns model-summarised text, so quotes are from that summary, not raw HTML.

## 1. Publishing status, user caps, token lifetimes
Source: https://developers.google.com/identity/protocols/oauth2/production-readiness/overview (page last updated 2026-05-22). VERIFIED.
- **Testing (External):** only allowlisted test users can sign in, up to 100 test users. The page states an exception for apps that request only basic identity scopes (openid, email, profile). Users see a testing warning.
- **Testing refresh tokens expire after 7 days.** A Workspace admin marking the app "Trusted" overrides this, and the 100 cap, for that org only.
- **Published, unverified (External):** any Google user can sign in. Name and logo are not shown on the consent screen. Sensitive or restricted scopes show a danger warning and the app is capped at 100 users.
- **Published, verified:** verification is required for public apps that request sensitive or restricted scopes. Brand verification is needed for the app name and logo to show.
- **Internal:** Workspace org users only, no verification.

Interpretation (mine): for a handful of users, 100 is no constraint. The practical problem with Testing is the 7-day refresh-token expiry (re-login weekly), if refresh tokens are used at all. Note that the page's "basic scopes exempt in Testing" wording is ambiguous about whether the 7-day expiry also applies to them. Check by testing.

## 2. Production with only openid/email/profile: 7-day expiry and verification
- https://support.google.com/cloud/answer/15549945 (no date). VERIFIED. Testing: "Authorizations by a test user will expire seven days from the time of consent." Exception: if the app requests only a subset of name, email, profile (userinfo.email, userinfo.profile, openid), users need not be on the test list, see no warning, and "their authorizations will not expire after 7 days."
  - This RESOLVES the earlier ambiguity: even in Testing, a basic-scopes-only app is exempt from the 7-day expiry. Caveat: the exemption does not apply once any other scope is added (e.g. Drive).
- The same page states no refresh-token expiry for Production, and says the 100 new-user lifetime cap tied to the unverified-app screen does not apply when only approved sensitive/restricted scopes are requested. It does not explicitly say basic scopes need no verification; it ties the unverified warning to sensitive/restricted scopes. Verification is tied to sensitive/restricted scopes and to showing app name/logo (production-readiness page, 2026-05-22).
- https://support.google.com/cloud/answer/13464323: verification exemptions are listed by audience/mode (personal, limited audience under 100, dev/test, internal), not by scope type; it does not state a rule for basic scopes. UNCONFIRMED as an explicit statement: "Production + basic scopes only needs no verification" is an inference from the pages above, not a quoted rule.
- Note: the production-readiness page's Testing row says basic-scope apps let any user in without the allowlist (access, not token expiry); the 7-day exemption comes from 15549945.
- Practical: for this app (basic scopes, few users) Testing already avoids the 7-day expiry per Google's docs; Production adds no extra need. Still, test once in the console (Google Auth Platform > Audience).

## 3. Cost
https://support.google.com/cloud/answer/13464323 and the production-readiness page mention no fee for OAuth client creation or verification. Neither page states "free" explicitly. UNCONFIRMED as an explicit statement; no cost found in any page read. (OpenID Connect page also has no pricing.)

## 4. How a CLI can authenticate
### Device flow (limited-input device). VERIFIED: https://developers.google.com/identity/protocols/oauth2/limited-input-device (updated 2026-11-19)
- Allowed scopes only: openid, email, profile; drive.appdata, drive.file; youtube, youtube.readonly. So the basic identity scopes are allowed.
- Client type must be "TVs and Limited Input devices" (other types give invalid_client).
- client_secret is required when polling /token; optional on refresh. The page does not say whether the initial device-code request needs it (lists only client_id).
- Server returns `interval` (example 5 s); polling faster gives 403 slow_down. `expires_in` for codes (example 1800 s).
- Quota: exceeding device-code request quota for the client ID gives 403 rate_limit_exceeded (numeric quota not stated). Refresh tokens are capped per client/user and per user across clients; older ones stop working.
### Loopback + PKCE (desktop app). VERIFIED: https://developers.google.com/identity/protocols/oauth2/native-app (updated 2026-11-19)
- Redirect http://127.0.0.1:port or http://[::1]:port (localhost also works but may hit firewall issues); must exactly match a registered URI, else redirect_uri_mismatch. Client type "Desktop app".
- Page says: "Always use PKCE". Verifier 43-128 chars of unreserved set; S256 recommended; if method omitted, defaults to plain.
- Desktop apps are public clients; client_secret is optional in token exchange and refresh per this page (PKCE replaces it).
- OOB copy/paste is no longer supported; custom URI schemes are no longer supported.
- Needs a browser on the same machine as the CLI.
### Via the web app (design option, not a Google doc claim)
Web app signs the user in, then issues its own session/API token or a one-time pairing code the CLI exchanges. Keeps Google out of the CLI; provider-agnostic.

## 5. OIDC details and other providers
- Google OIDC: https://developers.google.com/identity/openid-connect/openid-connect (updated 2026-11-19), VERIFIED. Discovery: https://accounts.google.com/.well-known/openid-configuration (page recommends caching per HTTP headers). Sample lists code_challenge_methods_supported plain and S256. scope must start with openid plus profile and/or email. Use `sub` as the unique user key, not email (emails can change).
- Microsoft Entra ID, VERIFIED: https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc (updated 2026-06-30). Discovery at https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration; discovery lists subject_types_supported ["pairwise"] (sub is per-app, so key by iss+sub within one app registration); device code and refresh tokens supported. PKCE detail is on the separate auth-code-flow page (not read).
- GitHub, VERIFIED: https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps (no date). OAuth 2.0 user login with PKCE (S256) in the web flow and a device flow for CLIs; this page shows no id_token or discovery, so it is not a drop-in OIDC provider (identity via GET /user). Whether GitHub offers user-login OIDC elsewhere is UNCONFIRMED.
- Apple: UNVERIFIED (docs page fetch failed). Auth0, Okta, Keycloak/Authentik, Cloudflare Access, Auth.js/Better Auth: UNVERIFIED general knowledge, not read.
- Design: configure by issuer + client id/secret; key users by (iss, sub), never by email alone; treat GitHub as a separate non-OIDC adapter.

## Conclusions and risks
1. Google fits: a few users is far below caps; basic scopes avoid the 7-day expiry even in Testing (per support 15549945) and the unverified warning.
2. Risk: that exemption disappears if any non-basic scope is added. Verification-free Production with basic scopes is inferred, not quoted.
3. Risk: device flow needs a client secret for polling and has an unquantified quota; loopback needs a local browser.
4. Prefer web-app-mediated CLI login; keep an issuer-configurable OIDC adapter keyed on (iss, sub).
5. Cost: no fee found, but "free" is not stated on pages read.
