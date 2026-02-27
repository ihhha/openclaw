# OpenClaw Rust Rewrite Plan (Multi-Agent Execution)

## Goal

Rewrite OpenClaw from TypeScript to Rust incrementally while keeping the product shippable at all times and enabling several agents to work in parallel with low merge risk.

## Repository strategy

Use the **same repository** during migration.

- Add Rust as an in-repo workspace (for example `crates/*`).
- Keep TypeScript and Rust implementations side by side until parity is proven.
- Keep one release pipeline and one contract-test pipeline.
- Split into another repo only if coupling remains low after stable cutover.

## Parallel-first architecture boundaries

Create crates with narrow ownership so each agent can own one lane with minimal overlap:

- `openclaw-contracts`: shared protocol/config schemas and generated bindings.
- `openclaw-core`: routing, policy evaluation, domain rules.
- `openclaw-gateway`: WS/HTTP server + orchestration.
- `openclaw-cli`: command parsing and output behavior.
- `openclaw-channels-<name>`: one crate per channel.
- `openclaw-media`: media pipeline logic.
- `openclaw-plugin-host`: plugin runtime boundary.
- `openclaw-observability`: tracing/metrics/health.
- `openclaw-compat`: config/session/wire compatibility helpers.

## Agent operating model (how several agents run in parallel)

### 1) Workstreams (one owner agent per stream)

- **A Contracts**: protocol schemas, fixtures, generated types.
- **B Core**: routing engine + policy decisions.
- **C Gateway**: transport/runtime/state fan-out.
- **D CLI**: command/flag parity and UX output.
- **E Channels**: channel adapters (split by channel family).
- **F Plugins**: runtime and TS plugin compatibility.
- **G Observability/Release**: telemetry, CI gates, packaging.

### 2) Dependency graph to avoid blocking

- Contracts unblock all streams.
- Core depends on Contracts.
- Gateway and CLI depend on Contracts + Core.
- Channels depend on Contracts + Gateway interfaces.
- Plugin host depends on Contracts + Gateway hooks.
- Observability runs in parallel from day one.

### 3) File ownership map (reduce merge conflicts)

- `crates/openclaw-contracts/**` -> Stream A only.
- `crates/openclaw-core/**` -> Stream B only.
- `crates/openclaw-gateway/**` -> Stream C only.
- `crates/openclaw-cli/**` -> Stream D only.
- `crates/openclaw-channels-*/**` -> Stream E only.
- `crates/openclaw-plugin-host/**` -> Stream F only.
- `crates/openclaw-observability/**`, CI wiring -> Stream G only.

Cross-stream edits require a short contract change note first (schema/version/changelog impact).

### 4) Integration cadence

- Daily: rebase/integrate each stream branch onto latest mainline.
- Daily: run parity contract suite against TS and Rust implementations.
- Twice weekly: merge train for streams that pass required gates.
- Weekly: shadow traffic report and rollback readiness review.

### 5) Definition of done per stream

A stream can merge when all are true:

- Contract tests pass.
- No regression in golden fixtures.
- Feature flag/fallback path exists.
- Observability hooks are present.
- Migration note is added for operators.

## Migration phases (parallelized)

## Phase 0: Contract freeze and baseline capture (Week 1-2)

**Primary:** Stream A
**Parallel:** Stream G (test harness infra)

- Freeze CLI/gateway wire contracts.
- Capture golden fixtures:
  - CLI help text, flags, exit codes.
  - WS events and request/response payloads.
  - Routing outcomes for DM/group/channel contexts.
- Build parity harness that can execute TS and Rust backends side-by-side.

## Phase 1: Workspace bootstrap (Week 2)

**Primary:** Streams A + G
**Parallel:** B/C/D skeletons

- Add Rust workspace and crate skeletons.
- Add CI jobs for `fmt`, `clippy`, unit tests, contract tests.
- Generate shared types from canonical schemas.

## Phase 2: Core engine implementation (Week 3-5)

**Primary:** Stream B
**Parallel:** C/D build against stable interfaces

- Port config normalization and routing policy logic.
- Keep TS gateway in front, calling Rust core via FFI or sidecar RPC.
- Validate deterministic parity with fixture replay.

## Phase 3: Gateway runtime (Week 5-8)

**Primary:** Stream C
**Parallel:** G observability hardening, E channel scaffolding

- Implement Rust WS/HTTP runtime.
- Preserve current auth/session/pairing semantics.
- Run mirror/shadow mode before enabling writes.

## Phase 4: CLI parity (Week 6-9)

**Primary:** Stream D
**Parallel:** B/C bugfixes from parity gaps

- Rebuild CLI in Rust with command and flag parity.
- Preserve output contracts (tables/status/progress/error codes).
- Keep TS command fallback for unported paths.

## Phase 5: Channel migration (Week 8+ rolling)

**Primary:** Stream E with sub-agents per channel

Suggested order:

1. Telegram
2. Discord
3. Slack
4. Signal
5. iMessage/Web
6. Extension channels (Matrix, Teams, Zalo, etc.)

Each channel follows identical checklist:

- adapter contract tests,
- send/receive parity,
- typing/presence/chunking/retry parity,
- soak test,
- feature-flag default flip.

## Phase 6: Plugin runtime transition (parallel)

**Primary:** Stream F

- Choose WASM-first or subprocess protocol model.
- Keep TypeScript plugin SDK compatibility during migration.
- Provide compatibility bridge so existing extensions keep working.

## Phase 7: Cutover and deprecation (final)

**Primary:** Streams C + D + G

- Flip defaults to Rust for core gateway and CLI.
- Keep TS fallback flags for one to two release cycles.
- Remove TS legacy paths only after telemetry + support stability.

## Technical choices

- Async runtime: `tokio`.
- Web stack: `axum` + websocket stack.
- Serialization: `serde` with explicit schema versioning.
- Keep current config shape and session storage format wherever possible.
- Plugin strategy: WASM preferred, subprocess fallback.

## Validation gates (must pass continuously)

- Contract parity tests (TS vs Rust).
- Property tests for routing and policy invariants.
- Integration tests per channel with mocks.
- E2E smoke tests for onboarding, send/receive, restart recovery.
- Performance SLOs: startup, p50/p95 latency, memory ceiling.
- Reliability SLOs: reconnect behavior, retry idempotency, crash recovery.

## Risk controls for multi-agent delivery

- **Schema churn conflicts** -> only Stream A edits canonical schema files.
- **Cross-lane merge conflicts** -> strict crate ownership and interface-first changes.
- **Behavior drift** -> golden fixtures + shadow mode diff reports.
- **Slow integration** -> scheduled merge train and required gate dashboard.
- **Plugin breakage** -> TS compatibility bridge until stable replacement.

## First 10 execution tasks (parallel-ready)

1. Create `crates/` workspace and empty crate layout.
2. Introduce `openclaw-contracts` with versioned schema package.
3. Build fixture capture scripts for CLI and gateway payloads.
4. Add parity test runner invoking TS and Rust backends.
5. Scaffold `openclaw-core` with routing test corpus.
6. Add Rust CI lane (`fmt`, `clippy`, `test`, parity suite).
7. Scaffold `openclaw-gateway` with health endpoint and auth stub.
8. Scaffold `openclaw-cli` with top-level command map.
9. Define channel adapter trait and Telegram pilot crate.
10. Add feature flags for TS fallback + shadow mode toggles.
