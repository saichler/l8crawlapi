# PRD: l8crawlapi — AI-Driven Website API Discovery CLI

## 1. Summary

`l8crawlapi` is a command-line tool that logs into a target website with user-supplied credentials, uses an AI agent to autonomously navigate the site and observe its network traffic, discovers the undocumented HTTP APIs the web UI calls, probes those endpoints to infer request/response shapes, and emits a `.proto` file (plus optional Go bindings) that describes those APIs as Layer 8 services for later integration.

## 2. Goals

- Given only `--url`, `--user`, `--password`, produce a usable `discovered.proto` describing the site's backend API surface.
- Require zero prior knowledge of the target site's API.
- Output must be deterministic enough to diff across runs (stable naming, sorted fields).
- Generated `.proto` must compile via `make-bindings.sh` and conform to Layer 8 protobuf conventions (enum zero values, list types, prime object rules).

## 3. Non-Goals

- Not a general-purpose scraper or content extractor — only API shape discovery.
- Not a load tester — probes are rate-limited and minimal.
- Not a bypass tool for MFA, CAPTCHA, WAF, or anti-bot protections beyond what the user has legitimate credentials for.
- No support for non-HTTP protocols (WebSocket framing, gRPC-web, GraphQL subscriptions) in v1 — flagged as future work.

## 4. Authorization & Safety

The tool is dual-use. It MUST:
- Require an explicit `--i-own-this-site` or `--authorized` flag before running, with text the user must type (e.g., "I am authorized to test <host>").
- Refuse to run against a hard-coded denylist of well-known third-party services unless `--force` is set.
- Respect `robots.txt` by default (override with `--ignore-robots`).
- Rate-limit requests (default: 2 req/sec, configurable).
- Never retry failed auth more than N times (default 3) to avoid account lockout.
- Log every outbound request to an audit log file alongside the output.

## 5. CLI Surface

```
l8crawlapi discover \
  --url https://app.example.com \
  --user alice@example.com \
  --password-env TARGET_PASSWORD \
  --out ./out/example.proto \
  --authorized "I am authorized to test app.example.com" \
  [--max-duration 10m] \
  [--max-requests 500] \
  [--rate 2] \
  [--headless=true] \
  [--ai-model claude-opus-4-6] \
  [--ai-api-key-env ANTHROPIC_API_KEY] \
  [--service-area 100] \
  [--package-name example] \
  [--audit-log ./out/audit.jsonl]
```

- Password is read from env var (`--password-env`) or stdin prompt. Never a positional arg.
- `--out` directory receives: `discovered.proto`, `audit.jsonl`, `samples/` (captured req/resp pairs), `report.md` (summary).

## 6. Architecture

Four components, running in a single Go process:

### 6.1 Browser Driver
- Headless Chromium via `chromedp` (vendored).
- Loads the site, performs login using AI-assisted form detection (see 6.2).
- Captures ALL network traffic via CDP (`Network.requestWillBeSent`, `Network.responseReceived`, `Network.getResponseBody`).
- Navigates autonomously: AI agent issues high-level intents ("click primary nav items", "open detail view of first row", "submit visible forms with valid-looking data"), driver translates to CDP input events.

### 6.2 AI Agent (Planner + Interpreter)
- Uses the Claude API (Anthropic SDK, per `claude-api` skill) with prompt caching on the site's DOM snapshots.
- Responsibilities:
  1. **Login**: given the login page DOM, identify username/password fields and submit button; plan and execute the login action.
  2. **Exploration plan**: given the post-login DOM, produce a ranked list of UI actions most likely to exercise distinct API endpoints.
  3. **API classification**: given a captured request/response pair, classify:
     - Is this an API call (vs asset/tracking)?
     - What entity/operation does it represent? (CRUD + action verb)
     - Are two endpoints variants of the same entity?
  4. **Schema synthesis**: from multiple samples of the same endpoint, infer a stable JSON schema (field names, types, optional/required, enums).
- Model default: `claude-opus-4-6`. Cacheable system prompt with the classification rubric.

### 6.3 Traffic Analyzer
- Filters captured traffic: drops static assets, analytics beacons, third-party domains (unless same-origin API via CORS).
- Groups requests by `(method, path-template)` where `path-template` is the URL with numeric/UUID segments replaced by `{id}`.
- For each group: aggregates samples, merges inferred schemas, records auth requirements (headers, cookies), pagination hints (`page`, `limit`, cursor fields).

### 6.4 Proto Generator
- Emits one `.proto` file following Layer 8 conventions:
  - Each discovered entity → a `message` with fields sorted by frequency-of-presence.
  - Each discovered endpoint → a service method with request/response messages.
  - Enums use `*_UNSPECIFIED = 0` per `proto-enum-zero-value`.
  - List wrappers use `repeated X list = 1; l8api.L8MetaData metadata = 2;` per `proto-list-convention`.
  - References between entities use string IDs, not nested structs, per `prime-object-references`.
- Emits a companion `discovered.yaml` that maps each proto method back to the real HTTP endpoint (method, path template, auth).
- Does NOT auto-register types or emit Go code in v1 — the user runs `make-bindings.sh` manually.

## 7. Discovery Flow

1. Validate authorization flag and target URL.
2. Launch headless browser, load `--url`.
3. AI identifies and executes login; verify success heuristic (URL changed, auth cookie set, etc.).
4. Snapshot post-login DOM; AI produces exploration plan.
5. Loop (bounded by `--max-duration`, `--max-requests`):
   - Pop next planned action; execute; capture new traffic.
   - After each action, feed new DOM + new captured requests to AI for replanning.
   - AI may schedule follow-up probes: replay a discovered endpoint with mutated params to confirm required vs optional fields (bounded, read-only operations preferred; writes only with `--allow-writes`).
6. When exploration budget is exhausted or AI reports "no new endpoints in last N actions":
   - Run analyzer → generate proto → write outputs.
7. Print summary to stdout: N endpoints, M entities, output path.

## 8. Output Artifacts

```
out/
├── discovered.proto       # Protobuf definitions (Layer 8 compliant)
├── discovered.yaml        # Proto method → HTTP endpoint mapping
├── audit.jsonl            # Every outbound request (timestamp, method, url, status)
├── samples/               # Raw request/response pairs (one file per endpoint group)
│   └── <method>_<path>.json
└── report.md              # Human-readable summary: endpoints found, confidence, gaps
```

## 9. Layer 8 Compliance

Per `prd-compliance.md`:

- **Project structure**: CLI lives at `go/crawl/main/` with `main.go`, `build.sh`, `Dockerfile` (per `deployment-artifacts.md`).
- **Protobuf**: generator emits compliant enums (`UNSPECIFIED = 0`), list types (`repeated X list = 1`), no cross-prime struct refs.
- **Service design**: discovered services get `ServiceName` ≤10 chars, ServiceArea from `--service-area` flag.
- **No generics**: per `no-go-generics.md`.
- **Vendored deps**: `chromedp`, `anthropic-sdk-go` under `go/vendor/`.
- **Build verification**: `go build ./...` (per `cleanup-test-binaries.md`).
- **Tests**: live under `go/tests/`, exercise the CLI via its command interface, not internal functions (per `test-location-and-approach.md`).

## 10. Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Login flow uses SSO / MFA / CAPTCHA | Detect and abort with actionable error; support `--cookie-jar` to import a pre-authenticated session |
| AI hallucinates endpoints that weren't observed | Generator only emits endpoints present in audit log; AI is used for classification, not invention |
| Writes cause data corruption on target | `--allow-writes` off by default; when off, skip POST/PUT/DELETE probes |
| Cost blowout on AI calls | Prompt caching; per-run budget cap (`--ai-budget-usd`); fail closed when exceeded |
| Schema drift across samples | Require ≥2 samples before emitting an optional field; unify via AI only when ambiguous |
| Target site terms of service | Authorization gate + audit log; user is responsible |

## 11. Out of Scope (v1)

- WebSocket / SSE / gRPC-web payload decoding.
- GraphQL schema introspection (detect and warn only).
- Authenticated-session replay against a staging environment.
- Generating a full Layer 8 service implementation — only the `.proto` contract.
- Multi-tenant crawl (one site per invocation).

## 12. Traceability Matrix

| # | Requirement | Component | Section |
|---|------------|-----------|---------|
| 1 | CLI flags and authorization gate | CLI entrypoint | 5, 4 |
| 2 | Headless login | Browser Driver + AI Agent | 6.1, 6.2 |
| 3 | Traffic capture | Browser Driver (CDP) | 6.1 |
| 4 | Endpoint grouping and schema inference | Traffic Analyzer | 6.3 |
| 5 | AI classification and planning | AI Agent | 6.2 |
| 6 | Proto emission, Layer 8 compliant | Proto Generator | 6.4, 9 |
| 7 | Audit log, rate limiting, robots.txt | Browser Driver | 4 |
| 8 | Safe defaults (no writes) | Browser Driver + Flags | 4, 10 |
| 9 | Deployment artifacts | build.sh, Dockerfile | 9 |
| 10 | Tests via CLI interface | go/tests/ | 9 |

## 13. Implementation Phases

- **Phase 0**: Project skeleton under `go/crawl/main/`, vendoring, CLI flag parsing, authorization gate.
- **Phase 1**: Browser driver with CDP traffic capture; unauthenticated crawl of a fixture site; audit log.
- **Phase 2**: AI Agent login + exploration planner against a fixture site with known credentials.
- **Phase 3**: Traffic Analyzer — grouping, path templating, schema merging.
- **Phase 4**: Proto Generator — emit Layer 8 compliant `.proto` + mapping YAML.
- **Phase 5**: Safety features — robots.txt, rate limit, denylist, `--allow-writes` gate, budget caps.
- **Phase 6**: Deployment artifacts (build.sh, Dockerfile); integration tests under `go/tests/` driving the CLI end-to-end against a local fixture server.
- **Phase 7**: End-to-end verification against 2–3 real sites the author owns (personal dashboards, dev instances). Validate that emitted `.proto` compiles via `make-bindings.sh` and that the mapping YAML is sufficient to replay a call.

## 14. Open Questions

1. Should the tool support a `--resume` mode that imports a prior `samples/` directory and re-synthesizes the proto without re-crawling?
2. How should it handle sites that require role-switching to exercise admin endpoints? (v1: one role per run.)
3. Is there value in emitting OpenAPI alongside protobuf for interop? (Leaning yes, as a flag in v1.1.)
