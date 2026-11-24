# R9: Hand-written OpenAPI vs Protocol Buffers (gRPC-Web / Connect) for the API contract

Researched 2026-11-24. Every claim below was read from the cited URL in this session (page dates given where the page showed them). Items marked UNVERIFIED were not read from a primary source. Microsoft Learn pages carry conflicting "latest version" banners (.NET 10 vs .NET 11) and `updated_at` metadata later than their `ms.date`; I quote `ms.date` / "Last updated".

## Summary table

| # | Criterion | Winner | Note |
|---|-----------|--------|------|
| 1 | ASP.NET Core server | Protobuf (gRPC-Web) narrowly | gRPC/gRPC-Web are first-party and documented; no Connect-protocol server for .NET found; contract-first OpenAPI server stubs exist only via third-party tools (NSwag, openapi-generator) |
| 2 | Browser client | Protobuf (Connect-ES) | Typed, small, server streaming works; OpenAPI TS generators are also strong and very popular |
| 3 | Rust client | Protobuf | prost/tonic are mature; Connect Rust is pre-1.0; progenitor supports OpenAPI 3.0.x only |
| 4 | Live updates | OpenAPI + SSE (tie at best) | SSE is the simpler path through Caddy; gRPC-Web streaming needs flush config and has no auto-reconnect |
| 5 | Photo uploads | OpenAPI | Plain HTTP multipart/raw body; protobuf needs chunking or a side HTTP endpoint |
| 6 | Breaking-change CI | Protobuf (`buf breaking`) | Built in, four strictness levels, free CLI; oasdiff is good but a separate tool |
| 7 | Debuggability | OpenAPI | curl, browser devtools and plain JSON; protobuf over gRPC-Web is binary-framed |
| 8 | Licensing / hosted | Tie | Buf CLI, oasdiff, NSwag, openapi-generator all permissive and local; BSR not required |
| 9 | Usage evidence | Tie | Both have large npm/crates numbers; OpenAPI is the more universal convention (UNVERIFIED as a proxy for agent friendliness) |

## 1. ASP.NET Core server

**Protobuf**
- gRPC is first-party. The comparison page says gRPC needs a `.proto` file and generates service base classes, messages and typed clients; browsers cannot call gRPC directly, so ASP.NET Core offers gRPC-Web or JSON transcoding. https://learn.microsoft.com/en-us/aspnet/core/grpc/comparison?view=aspnetcore-10.0 (original 2019-12-05, last updated 2024-07-31).
- gRPC-Web: package `Grpc.AspNetCore.Web`, `app.UseGrpcWeb()` after routing and `EnableGrpcWeb()` per service; CORS must expose `Grpc-Status`, `Grpc-Message` etc. Browser support: unary and server streaming only (server streaming needs `GrpcWebText` for the .NET client). Client and bidi streaming are not supported. Microsoft recommends only unary and server streaming with gRPC-Web. https://learn.microsoft.com/en-us/aspnet/core/grpc/grpcweb?view=aspnetcore-10.0 (ms.date 2025-07-10).
- JSON transcoding (`Microsoft.AspNetCore.Grpc.JsonTranscoding`): annotate each rpc with `google.api.http`; runs in-process, works over HTTP/1.1 and HTTP/2; only server streaming is supported, delivered as line-delimited JSON. https://learn.microsoft.com/en-us/aspnet/core/grpc/json-transcoding?view=aspnetcore-10.0 (2026-10-26).
- The OpenAPI-for-transcoding package `Microsoft.AspNetCore.Grpc.Swagger` is marked deprecated: "experimental ... no direct replacement" (see issue dotnet/aspnetcore#67134, not read). https://learn.microsoft.com/en-us/aspnet/core/grpc/json-transcoding-openapi?view=aspnetcore-10.0 (ms.date 2026-08-16). This weakens the hybrid route (section 10).
- Connect-protocol server for ASP.NET Core: NOT FOUND. The Connect introduction lists Go, TypeScript/JavaScript, Swift, Kotlin, Python, Rust and Dart and does not mention C#/.NET (https://connectrpc.com/docs/introduction, undated). The docs landing page redirects without a language list. A web search for a .NET Connect server returned nothing relevant. Treat as "no first-party or documented Connect server for .NET"; a web client would therefore use `createGrpcWebTransport` against ASP.NET Core (https://connectrpc.com/docs/web/choosing-a-protocol: gRPC-web transport works with ASP.NET Core middleware).

**OpenAPI contract-first**
- Microsoft's `Microsoft.AspNetCore.OpenApi` is code-first only: it generates a document from endpoints (optionally at build time). The page does not cover server stubs or request validation against a spec. https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/overview?view=aspnetcore-10.0 (ms.date 2026-08-28). So contract-first needs third-party tools.
- NSwag: `CSharpControllerGenerator` creates ASP.NET Core controllers from an OpenAPI spec ("contract first/schema first"); MIT; actively committed (4,316 commits), but I could not read a latest release/date. https://github.com/RicoSuter/NSwag (page undated).
- openapi-generator: lists C# ASP.NET Core server stubs; latest stable 7.25.0 dated 24.08.2026; Apache-2.0. https://github.com/OpenAPITools/openapi-generator.
- Request validation against a spec at runtime: no primary source read. UNVERIFIED. (Hand-rolled validation in handlers, or generated controllers with model validation, is the likely fallback.)

## 2. Browser client

**Protobuf**
- protobuf-es (`@bufbuild/protobuf`): claims 100% conformance with the official suite, "tiny bundle size" (no figures on the README), Apache-2.0, targets Baseline browsers of the last 2.5 years. https://github.com/bufbuild/protobuf-es. Weekly npm downloads 44,695,005 (2026-11-17 to 2026-11-23) https://api.npmjs.org/downloads/point/last-week/@bufbuild/protobuf.
- Connect-ES: `createConnectTransport` and `createGrpcWebTransport` from `@connectrpc/connect-web`; uses fetch; browsers can only do server streaming because vendors have not agreed on streaming request bodies. https://connectrpc.com/docs/web/getting-started and https://connectrpc.com/docs/web/choosing-a-protocol. FAQ: generated code affects bundle size; ELIZA client about 13 KiB compressed (vendor example, not our schema). https://connectrpc.com/docs/faq. (My attempt to read the connect-es GitHub README was rate limited twice: UNVERIFIED for its version/licence; `@connectrpc/connect` had 9,588,011 weekly downloads, 2026-11-17 to 2026-11-23.)
- Server streaming in a browser: fetch-streamed response with enveloped messages (Connect streaming content types `application/connect+json` / `+proto`; each message framed as 1 flag byte + 4-byte length). https://connectrpc.com/docs/protocol.
- Code generation at build time: `buf generate` with local plugins (`protoc-gen-es`); fits "generated, not committed". https://github.com/bufbuild/buf.

**OpenAPI**
- @hey-api/openapi-ts: MIT, generates SDKs, Zod schemas, TanStack Query hooks, Fetch/Axios clients; "millions of weekly downloads" claim is self-reported; npm shows 8,314,694 (2026-11-16 to 2026-11-22). https://github.com/hey-api/openapi-ts. openapi-typescript: 17,848,617 weekly (2026-11-12 to 2026-11-18). Licence/version of openapi-typescript not read.
- NSwag also emits TS (Fetch) clients (MIT).
- Kiota's TypeScript generation is labelled Preview (C# Stable). https://github.com/microsoft/kiota.

## 3. Rust client

**Protobuf**
- prost: Apache-2.0, MSRV 1.85, README says "passively-maintained" (bug/security fixes, no new features). Requires `protoc` installed since prost-build 0.11. https://github.com/tokio-rs/prost. crates.io: v0.14.6, updated 2026-11-12, 736.5M total, 184M recent downloads. https://crates.io/api/v1/crates/prost
- tonic: MIT, MSRV 1.88, 0.14.x line released; crates.io v0.14.9, updated 2026-10-29, 489.6M total downloads. https://github.com/hyperium/tonic, https://crates.io/api/v1/crates/tonic. It is a gRPC (HTTP/2) client; as a CLI client talking to ASP.NET Core over HTTP/2 gRPC this works without gRPC-Web (the CLI is not a browser). Interop with the ASP.NET Core server was not read: UNVERIFIED.
- connectrpc (Rust): v0.9.1, 2026-09-21, Apache-2.0, pre-1.0 ("API surface is settling but may shift in 0.x"), MSRV 1.88; speaks Connect, gRPC and gRPC-Web; passes the Connect conformance suite per its README; uses buffa (pure-Rust protobuf), not prost; tonic interop stated. https://docs.rs/crate/connectrpc/latest, https://connectrpc.com/docs/rust/getting-started. Downloads 10.1M total (4.1M recent). https://crates.io/api/v1/crates/connectrpc.

**OpenAPI**
- progenitor (Oxide): MPL-2.0; README says OpenAPI 3.0.x only, "may fail for some OpenAPI documents", no header handling in generated code; docs.rs shows 0.15.1 (2026-10-09) while crates.io shows max_version 0.17.0 (2026-11-19), a discrepancy I did not resolve. Downloads 7.06M total. https://github.com/oxidecomputer/progenitor, https://docs.rs/crate/progenitor/latest, https://crates.io/api/v1/crates/progenitor. Risk: a hand-written 3.1 document may not work; stay on 3.0.x if chosen.
- openapi-generator lists Rust clients (hyper, reqwest, rust-server) with no maturity statement on the page I read. https://github.com/OpenAPITools/openapi-generator. Kiota does not list Rust.

## 4. Live updates (SSE vs streaming)

- Caddy `reverse_proxy`: for `Content-Type: text/event-stream` responses `flush_interval` is ignored and responses flush immediately. Otherwise "no periodic flushing" by default and the proxy "partially buffers" responses; `flush_interval -1` disables buffering. https://caddyserver.com/docs/caddyfile/directives/reverse_proxy (undated). Inference (not in the page): gRPC-Web / Connect streaming responses use other content types, so they would need `flush_interval -1`.
- Caddy h2c upstream is documented as a non-standard feature; the page does not mention gRPC. Native gRPC to Kestrel through Caddy: UNVERIFIED.
- Cloudflare: default Proxy Read Timeout is 125 seconds (confirmed), only Enterprise can raise it; page does not mention heartbeats. https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-524/ (updated 2026-07-23). Either choice therefore needs an application heartbeat under 125 s.
- Cloudflare gRPC: available on all plans, needs port 443, HTTP/2, `application/grpc` content type; no mention of gRPC-Web or streaming behaviour; Access does not support gRPC. https://developers.cloudflare.com/network/grpc-connections/
- Reconnection: browser `EventSource` auto-reconnect and `Last-Event-ID` are standard browser behaviour but I did not read the spec this session: UNVERIFIED. For Connect/gRPC-Web streams, the Connect docs describe no automatic reconnect; the client code would have to re-open the stream and resume (inference).
- Microsoft says gRPC has no broadcast concept (each call streams individually) and recommends SignalR for chat-room style push. https://learn.microsoft.com/en-us/aspnet/core/grpc/comparison?view=aspnetcore-10.0
- SSE is not describable in OpenAPI 3.1.1 as a first-class streaming type (I did not find it in the sections read): UNVERIFIED. A text/event-stream response can be declared, but event payload types usually live outside the generated client. Keeping SSE beside either contract is viable; a protobuf contract could instead replace SSE with a server-streaming RPC.

## 5. Photo uploads

- OpenAPI 3.1.1 (24 Oct 2024): raw binary body or multipart part is expressed with `contentMediaType` (3.0 used `format: binary`); base64 inside JSON uses `contentEncoding`. https://spec.openapis.org/oas/v3.1.1.html (only first 100k of 327k characters read).
- gRPC in ASP.NET Core: default `MaxReceiveMessageSize` 4 MB (server and client), send unlimited; docs advise raising limits only per service, and (as general guidance, not stated on that page) streaming for very large data. https://learn.microsoft.com/en-us/aspnet/core/grpc/configuration?view=aspnetcore-10.0
- Browsers cannot do client streaming in Connect/gRPC-Web, so a browser photo upload must be a single unary message (bytes field, subject to the size limit) or a separate plain HTTP endpoint. https://learn.microsoft.com/en-us/aspnet/core/grpc/grpcweb?view=aspnetcore-10.0
- Conclusion: OpenAPI handles photos natively; protobuf almost certainly needs a second plain-HTTP upload route (derived from the sources above, no source states this as a recommendation).

## 6. Breaking-change detection

- `buf breaking`: compares against Git, tarball, BSR or image; categories FILE (default), PACKAGE, WIRE_JSON, WIRE; catches e.g. field type change on a tag; has GitHub Action. https://buf.build/docs/breaking/. buf CLI is Apache-2.0 and "core CLI features work without a BSR account." https://github.com/bufbuild/buf. The WIRE_JSON level matches our case: old clients stay working on the wire and JSON.
- oasdiff: Apache-2.0; `breaking`, `changelog`, `diff` commands; GitHub Action and pre-commit hook; supports OpenAPI 3.1 and 3.2 (3.0 upgrade path); a hosted PR-review workflow exists at oasdiff.com (optional). Number of rules and latest release not stated on the page. https://github.com/oasdiff/oasdiff
- Not checked: whether oasdiff catches semantic changes (e.g. enum value additions in responses) and how its rules compare in depth to buf's. UNVERIFIED. Protobuf has an inherent advantage: field numbers and `reserved` give well-defined wire compatibility rules; OpenAPI compatibility depends on client strictness.

## 7. Debuggability

- Connect unary calls are plain HTTP POST with JSON (`Content-Type: application/json`), curl-able; side-effect-free methods can use GET; streaming is harder by hand because of envelope framing. https://connectrpc.com/docs/protocol. But no .NET Connect server exists (section 1), so against ASP.NET Core you would use gRPC-Web (binary default; base64 text mode for streaming) or JSON transcoding (which gives curl-friendly REST + line-delimited JSON streaming but cannot do client streaming; its OpenAPI package is deprecated).
- ASP.NET Core gRPC logging uses category `Grpc`; tracing creates Activities with tags `grpc.method` and `grpc.status_code`; tracing covers the call, not individual streaming messages; metrics via EventCounters; OpenTelemetry consumes these Activities; the page does not mention `rpc.*` semantic-convention attributes. https://learn.microsoft.com/en-us/aspnet/core/grpc/diagnostics?view=aspnetcore-10.0 (undated on fetch).
- OpenAPI/REST: standard ASP.NET Core HTTP instrumentation applies (curl, Swagger-like UIs, readable JSON logs). I did not fetch the OpenTelemetry ASP.NET Core instrumentation page: UNVERIFIED.

## 8. Licensing and hosted components

- buf CLI Apache-2.0; no BSR needed for lint/breaking/generate with local plugins. https://github.com/bufbuild/buf
- BSR pricing: Community free = one private repository (max 100 types) and unlimited public; Teams $0.50 per type/month; Pro $5 per type/month with $3,000/month minimum. https://buf.build/pricing (undated). Whether the free tier includes generated SDKs was unreadable. We do not need BSR if we generate locally.
- protobuf-es Apache-2.0 (varint code BSD-3); prost Apache-2.0; tonic MIT; connectrpc-rs Apache-2.0; protoc must be installed for prost-build (prost README); protoc-gen-openapiv2 BSD-3.
- OpenAPI side: oasdiff Apache-2.0, NSwag MIT, openapi-generator Apache-2.0 (generated code not covered by the licence), hey-api MIT, Kiota MIT, progenitor MPL-2.0 (file-level copyleft on the crate itself, generated output not examined: UNVERIFIED).
- Hosted dependency: none required for either path. The only optional hosted services are BSR and oasdiff.com.

## 9. Usage evidence (primary registry numbers)

Weekly npm downloads: @bufbuild/protobuf 44.7M; @connectrpc/connect 9.6M; @grpc/grpc-js 88.4M; @hey-api/openapi-ts 8.3M; openapi-typescript 17.8M (periods 2026-11-12 to 11-23; see section 2). crates.io: prost 736M total, tonic 490M, progenitor 7M, connectrpc 10M. Both approaches are widely used. Download counts include transitive installs and do not measure coding-agent friendliness; no primary source for agent friendliness was found. UNVERIFIED.

## 10. Hybrid options

- gRPC JSON transcoding (proto + `google.api.http`) is first-party in ASP.NET Core, but its OpenAPI generator is deprecated with no replacement. https://learn.microsoft.com/en-us/aspnet/core/grpc/json-transcoding-openapi?view=aspnetcore-10.0
- protoc-gen-openapiv2 (grpc-gateway) generates OpenAPI v2 (Swagger) from `google.api.http`; described as production-stable; the OpenAPI 3.1 plugin (`protoc-gen-openapiv3`) is alpha. BSD-3. https://github.com/grpc-ecosystem/grpc-gateway
- gnostic (`protoc-gen-openapi`) is in the google/gnostic repo, pre-1.0 "prerelease software"; whether it emits v3 from http annotations was not confirmed by the page. https://github.com/google/gnostic
- So hybrid is documented, but the maintained path produces Swagger 2.0, which in turn feeds generators whose 3.x support varies; it adds a second artifact to keep in sync.

## Biggest risks

- Protobuf: no Connect server for ASP.NET Core, so browser access is gRPC-Web with base64/binary framing and weaker debuggability; no client streaming from browsers; photo upload needs a side route; SSE replacement requires custom reconnect; proxy flush config (Caddy) must be handled; connectrpc-rs is pre-1.0; prost is passively maintained.
- OpenAPI: server stubs and request validation depend on third-party tools (NSwag/openapi-generator) of unverified recency; Rust generation is the weakest link (progenitor is 3.0.x only and "may fail"); drift between the spec and handlers unless generated stubs are used; breaking-change rules are less formal.

## Verified vs unverified

Verified from fetched primary pages: sections 1 (gRPC/gRPC-Web/transcoding, NSwag, openapi-generator), 2 (Connect-Web docs, protobuf-es README), 3 (prost, tonic, connectrpc, progenitor), 4 (Caddy, Cloudflare timeout), 5, 6 (buf, oasdiff), 8 (pricing), 9 (registry counts). UNVERIFIED or not fetched: connect-es README (rate limited twice), OpenAPI runtime request-validation tooling, EventSource reconnect spec, OpenTelemetry ASP.NET Core HTTP instrumentation page, Kestrel/Caddy native gRPC proxying, tonic-to-ASP.NET interop, agent friendliness, deprecation rationale in dotnet/aspnetcore#67134.
