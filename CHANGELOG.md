# Changelog — `armature-mcp`

All notable changes to this crate will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this crate adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Earlier changes are recorded in the workspace [`CHANGELOG.md`](../CHANGELOG.md).

## [Unreleased]

## [0.3.1] - 2026-08-29

### Fixed

- **Every JSON-RPC POST failed to parse.** `McpController::handle_request` read the payload from the public `HttpRequest::body` field, but on `armature-core` `0.7` that `Vec<u8>` is *cleared* when the incoming body is stored zero-copy in the private `Bytes` slot (`set_body_bytes`), so any request off the wire reached the parser empty and came back `-32700 "EOF while parsing a value at line 1 column 0"` despite carrying a body. The body is now read through `body_ref()` — the `Bytes` view when set, the legacy `Vec` otherwise — which is correct on every `armature-core` version. The existing tests built their requests with `set_body` (the legacy path) and so never exercised the wire shape; a `post_request_from_wire` helper and regression test now cover it.
- **Breaking:** `handle_request`/`handle_json` return `Option`, and the controller answers 204. The server replied to JSON-RPC notifications, which §4.1 forbids, so a conformant client sending `notifications/initialized` received a `-32601` error.
- `prompts/list` and `prompts/get` are implemented. The server advertised the prompts capability whenever it was enabled and then answered both with method-not-found.
- A malformed tool `input_schema` is surfaced rather than silently replaced with a permissive stand-in.

### Changed — `0.1.5` → `0.1.6`

- Migrated onto `armature-core` `0.8`'s `Bytes`-backed request and response types. No behavior change beyond what that migration implies; see [`armature-core/CHANGELOG.md`](../armature-core/CHANGELOG.md).

### Fixed

- **JSON-RPC notifications are no longer answered.** Per JSON-RPC 2.0 §4.1 a request without an `id` must not receive a response. `notifications/initialized` — which every MCP client sends immediately after `initialize` — previously came back as `-32601 Method not found`. Notifications are now dispatched for their side effects only, and `POST /mcp` answers `204 No Content`. `notifications/initialized` and `notifications/cancelled` are recognised no-ops.
- **The advertised `prompts` capability is now actually served.** `initialize` announced `capabilities.prompts` whenever `enable_prompts` was set, but there were no `prompts/list` / `prompts/get` handlers. Both methods now exist, backed by a new `McpPromptRegistry` (compile-time `register_mcp_prompt!` inventory) plus dynamic `McpPromptProvider`s registered with `McpService::with_prompt_provider`.
- A JSON-RPC batch (top-level JSON array) is rejected with `-32600 Invalid Request` instead of the misleading `-32700 Parse error`. Batching remains unsupported and is now documented as such.
- A malformed `input_schema` on a registered tool is still replaced with a permissive `{"type":"object"}`, but now logs at `error` level and trips a `debug_assert!` rather than silently shipping the bug to clients.

### Changed

- **Breaking (API):** `McpService::handle_request` now returns `Option<JsonRpcResponse>`, and `handle_json` / `handle_json_unauthenticated` return `Option<String>`; `None` means "notification — send no body".
- Added `McpService::handle_bytes`, which parses the JSON-RPC payload directly from a byte slice. `POST /mcp` uses it, removing the per-request body copy and intermediate `String`.
- `GET /mcp` now reports `prompts_count` alongside `tools_count` and `resources_count`.

## [0.3.0] - 2026-08-05

### Changed

- **Requires `armature-core` 0.9 (breaking).** The requirement moved `0.8` →
  `0.9`. `armature-core 0.9.0` itself moves `armature-h1` across a breaking
  0.x boundary; because `armature-core` types appear in this crate's own
  public API, the requirement change is breaking here too and the minor moves
  with it. Under Cargo's 0.x caret rules the 0.8 and 0.9 types are distinct
  and do not unify, so a consumer holding an `armature-core 0.8` type cannot
  pass it to this crate. Part of the `armature-core 0.9.0` release train; see
  `armature-core`'s CHANGELOG for the publish order.
- Requires `armature-jwt` 0.3 (was `0.2`); it moved its minor in the same train for the same reason.
- Requires `armature-proc-macro` 0.4 (was `0.3`); it moved its minor in the same train for the same reason.

## [0.2.1] - 2026-08-04

### Fixed

- Requirements on sibling armature crates name a minor instead of `0`. Under
  Cargo's 0.x rules `version = "0"` matches any release ever made, and edition
  2024 selects the MSRV-aware resolver, so a consumer declaring an older
  `rust-version` was handed the oldest version satisfying it — resolving
  `armature-core = "0"` on Rust 1.89 produced `armature-core 0.2.3` while an
  explicit `armature-core = "0.8"` elsewhere in the same graph pulled 0.8.2.
  Two copies of core, and a build failing on symbols the older one lacks. Each
  0.x minor in this family is a breaking change, so the requirement now names
  one. No API change.
