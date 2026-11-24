# R10: CLI language options (C#, Rust, Go, TypeScript) for the shopping-list CLI

Researched 2026-11-24 (issue #24). Every claim below was read from the cited page in this session; page dates are as shown by the page. Items marked UNVERIFIED were not read from a primary source. Nothing is from memory.

## 1. Client generation from OpenAPI 3.0.x

| Generator | Evidence (source, date) | Notes |
|---|---|---|
| Kiota (C#, Go stable; TypeScript preview) | https://github.com/microsoft/kiota (table: C# stable, Go stable, TS/JS preview); releases page https://github.com/microsoft/kiota/releases shows v1.35.0 (Sep 2026) as Latest, preview 10 Sep | Active. Docs https://learn.microsoft.com/en-us/openapi/kiota/overview (2024-09-16) do not name OpenAPI versions or AOT/trimming. 3.0.x acceptance: UNVERIFIED (release notes mention 3.1 features, so 3.0 is likely but not stated). Generated via CLI. |
| NSwag (C#) | https://github.com/RicoSuter/NSwag: "Swagger/OpenAPI 2.0 and 3.0 toolchain", generates C# or TS clients; NSwag.MSBuild runs the tool in an MSBuild target. Releases: v14.7.1, "20 April" (year not shown on page) | Build-time generation fits "do not commit generated code". Release year and activity UNVERIFIED. |
| progenitor (Rust) | https://github.com/oxidecomputer/progenitor: targets "OpenAPI 3.0.x"; `generate_api!` macro or `build.rs` into OUT_DIR; "may fail for some OpenAPI documents"; no request header handling (set on reqwest client); no mention of multipart or binary bodies (photo upload risk). crates.io API: v0.17.0, updated 2026-11-19 (https://crates.io/api/v1/crates/progenitor) | Opinionated, Dropshot-first. Active. Multipart/file upload support UNVERIFIED. |
| oapi-codegen (Go) | https://github.com/oapi-codegen/oapi-codegen: OpenAPI 3.0 and 3.1; `go:generate`; Go 1.25+ to build the tool; no defined release cadence. Releases page: v2.8.2 "Security fixes for code generation", 31 October (year not shown) | Fits build-time generation via `go generate`. |
| openapi-generator | https://github.com/OpenAPITools/openapi-generator: OpenAPI 2.0 and 3.0, 3.1 beta; stable 7.25.0; needs Java 11+ | Covers Rust/Go/C#/TS but needs a JVM in CI. Per-generator stability labels not read. |
| hey-api openapi-ts | https://github.com/hey-api/openapi-ts: "accepts any OpenAPI specification", Node 22+. https://heyapi.dev/openapi-ts/get-started: "in initial development", pin an exact version. npm registry: 0.99.0 (engines node >=22.18.0). Clients: Fetch, Axios, Angular, Next.js, Nuxt, more | Pre-1.0. 3.0.x explicit support and SSE support UNVERIFIED (fetch client page does not mention SSE). |
| openapi-typescript | https://openapi-ts.dev/introduction (v7.x docs, no date): supports 3.0 and 3.1; types only, runtime-free; pair with openapi-fetch | Types only, so fewer moving parts, but you hand-write the client calls. |

Sharing:
- C# generated client could be shared with the C# E2E harness (same language and generator). That is an inference; no source states it.
- TypeScript client could be shared with the React app. Same inference.
- A Rust or Go CLI would need a third client generated just for the CLI.

## 2. Cross-building from GitHub-hosted runners

- Runner labels: `ubuntu-24.04-arm`, `ubuntu-22.04-arm`, `ubuntu-26.04-arm`, `windows-11-arm`, `windows-11-vs2026-arm`. Free and unlimited on public repos; private repos draw from included minutes then bill per minute. https://docs.github.com/en/actions/reference/runners/github-hosted-runners (no page date; footer 2026). So a native arm64 Linux runner is available.
- C# self-contained single-file: publish once per RID (`-r win-x64`, `linux-x64`, `linux-arm64`); apps are OS- and arch-specific. https://learn.microsoft.com/en-us/dotnet/core/deploying/single-file/overview (2026-03-25). The page does not state that one runner can build all three RIDs with plain `dotnet publish`; it is likely, but UNVERIFIED.
- C# NativeAOT: "does not support cross-OS compilation". Cross-architecture (x64 to arm64) is supported if the native toolchain is installed; on Linux that needs a cross-linker (clang), target objcopy/strip, target C runtime and zlib objects, and the Ubuntu example is "not documented or guaranteed". https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/cross-compile (2025-05-27). Needs a runner per OS (Windows runner for win-x64, Linux runner for Linux); linux-arm64 is natively built on `ubuntu-24.04-arm`. Windows needs VS C++ workload (https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/, 2026-11-05).
- Go: you are "always cross-compiling"; `linux/amd64`, `linux/arm64`, `windows/amd64` are supported targets. https://go.dev/doc/install/source (no date). One runner builds all three (with pure-Go, no CGO; GoReleaser's page only shows per-target C compilers for CGO). https://goreleaser.com/customization/builds/go/ (2026-09-06).
- Rust: cargo-zigbuild builds `aarch64-unknown-linux-gnu` from Linux with zig; page says only Linux and macOS targets are supported, so Windows MSVC cross-build is not covered (https://github.com/rust-cross/cargo-zigbuild, no date). dist (cargo-dist) "spins up machines for each platform you support", i.e. a runner per target. https://github.com/axodotdev/cargo-dist. Practical plan: one runner per target (windows-latest, ubuntu x64, ubuntu arm).
- Bun: `--target=bun-windows-x64 | bun-linux-x64 | bun-linux-arm64` from one host. Cross-compiling disables most Windows metadata flags. https://bun.com/docs/bundler/executables (no date).
- Deno: `--target` for `x86_64-pc-windows-msvc`, `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu` from any host; downloads denort per target. https://docs.deno.com/runtime/reference/cli/compile/ (2026-08-06).
- Node SEA: Stability 1.1 (active development); `--build-sea` since v25.5.0; cross-platform builds possible by pointing `executable` at a target Node binary, with `useCodeCache` and `useSnapshot` false; the docs do not say whether you must build on each target. Linux arm64 builds with postject in Docker may crash on `process.dlopen()`. Linux CI tested except Alpine; Windows tested. https://nodejs.org/api/single-executable-applications.html (Node v26.13.1 docs).

## 3. Size, startup, runtime requirements

- C# single-file: can be framework-dependent or self-contained; self-contained is large (bundles runtime). Trimming and ReadyToRun can be combined; compression trades startup for size. Native runtime libraries remain separate unless `IncludeNativeLibrariesForSelfExtract` (then extracted to `$HOME/.net` or `%TEMP%/.net` at start). No byte figures given. (single-file page, 2026-03-25)
- C# NativeAOT: "faster startup" and "smaller memory", no numbers; apps are larger than framework-dependent ones; no runtime install needed; limits: no dynamic loading/Reflection.Emit, trimming required, partial ASP.NET support. (NativeAOT page, 2026-11-05). Trimming page (2026-04-21) does not mention System.Text.Json reflection, so generated-client JSON compatibility with AOT is UNVERIFIED for Kiota and NSwag.
- Go and Rust: no size or startup figures read from primary sources. UNVERIFIED.
- Bun: embeds the runtime; no Node/Bun install needed; page says binary is "still too large", bytecode reportedly ~2x faster `tsc` startup. No size numbers. Deno: no sizes; options to shrink (`--bundle`, `--engine quickjs` experimental). Node SEA: no sizes read.

## 4. SSE and loopback OAuth/PKCE

- C#: `System.Net.ServerSentEvents` `SseParser.Create(Stream)`; the page documents package v11.0.0 / .NET 11 (ms.date 2025-07-01, updated 2026-05-27). https://learn.microsoft.com/en-us/dotnet/api/system.net.serversentevents.sseparser. Availability on earlier .NET versions UNVERIFIED. Duende OidcClient: repo exists (https://github.com/DuendeSoftware/foss) but loopback/PKCE support and .NET versions UNVERIFIED. Loopback listener via HttpListener: UNVERIFIED (not fetched).
- Rust: reqwest-eventsource wraps reqwest, retries (https://github.com/jpopesculian/reqwest-eventsource; 19 commits, no maintenance statement, no releases shown). oauth2 crate: described as extensible OAuth2 implementation, PKCE not confirmed on page (https://github.com/ramosbugs/oauth2-rs). Loopback server: UNVERIFIED.
- Go: golang.org/x/oauth2 v0.38.0 (published Nov 6, 2026) has `GenerateVerifier`, `S256ChallengeOption`, `VerifierOption` (https://pkg.go.dev/golang.org/x/oauth2). Loopback via net/http is standard library (general knowledge, UNVERIFIED). SSE: r3labs/sse fetch was rate limited twice, UNVERIFIED.
- TypeScript: Bun/Deno/Node SEA all have fetch and http built in (not read from sources, UNVERIFIED). hey-api SSE UNVERIFIED.

## 5. Self-update

- Rust: `self_update` crate: GitHub releases default backend, `checksums` feature (SHA-256/512, can read a `SHA256SUMS` asset, GitHub per-asset digest), zipsign signatures, default y/n confirmation prompt. https://github.com/jaemk/self_update (412 commits; activity dates not shown). dist also has an "updater" installer chapter (not read).
- Go: `creativeprojects/go-selfupdate`: GitHub/Gitea/GitLab sources, replaces running exe with rollback, verifies SHA256 via goreleaser-style checksums file, ECDSA. https://github.com/creativeprojects/go-selfupdate (152 stars). Prompt must be hand-written. Windows running-exe handling is not explicitly stated.
- C#: Velopack: "installer and automatic update framework for cross-platform desktop applications"; Windows, OSX, Linux; Linux arm64, GitHub Releases source and CLI suitability not stated. https://github.com/velopack/velopack. No small CLI-specific updater found in this research; a hand-rolled GitHub Releases download plus SHA-256 check plus exe swap is the likely pattern (UNVERIFIED as a pattern).
- TypeScript: no library researched; hand-rolled. UNVERIFIED.

## 6. Packaging and release tooling

- Rust: dist (formerly cargo-dist) generates `release.yml` (plan, build, host, publish, announce), tarballs, installers, GitHub Release creation. https://github.com/axodotdev/cargo-dist. Checksums and target list not read.
- Go: GoReleaser, latest v2.18.2 (17 Sep 2026; its docs page is dated 2026-09-06 and says v2.20, an inconsistency left unexplained); matrix of goos/goarch, checksums on a separate page not read.
- C#, TypeScript: no dedicated tool researched; a plain GitHub Actions matrix with `sha256sum` and `gh release` is implied. UNVERIFIED.

## Verdict per criterion

1. Generation: C# (Kiota, NSwag) and Go (oapi-codegen) are mature; progenitor is Dropshot-centric with unverified multipart; hey-api is pre-1.0.
2. Cross-build: Go is clearly best (one runner). Bun and Deno also from one host. C# single-file likely one runner. NativeAOT needs a runner per OS. Rust effectively one runner per target.
3. Size/startup: no comparable numbers in primary sources; unresolved.
4. SSE/PKCE: Go and C# viable; others partly unverified.
5. Self-update: Rust and Go have ready libraries with checksum verification; C# and TS appear hand-rolled.
6. Release tooling: Rust (dist) and Go (GoReleaser) have first-party tooling.

## Biggest risks

- C# single-file: large binary; NativeAOT with generated clients and JSON reflection; self-update is DIY.
- Rust: progenitor limits, learning cost, runner per target.
- Go: new language for the user, a third generated client.
- TypeScript: Node SEA unstable; hey-api pre-1.0; large binaries.

## Evidence-based lean

C# self-contained single-file (not NativeAOT) fits this user best: language fluency, Kiota or NSwag stable, shared with the E2E harness, no AOT risk. The cost is DIY self-update and size. Go is the strongest alternative on toolchain (cross-build, GoReleaser, go-selfupdate). Neither choice is proven on binary size from primary sources.

## Coverage

Verified from primary pages: about 60 percent of claims. UNVERIFIED: Kiota/NSwag/hey-api 3.0.x specifics, binary sizes/startup for Go/Rust, C# loopback OAuth and SSE on pre-.NET 11, dist targets/checksums, r3labs/sse, NSwag release year.
