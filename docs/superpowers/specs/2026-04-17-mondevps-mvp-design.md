# Mondevps MVP Design

**Status:** Approved for implementation
**Date:** 2026-04-17
**Author:** paul2 (brainstormed with Claude)

## 1. Overview

Mondevps is an open-source AI-driven incident analysis tool for DevOps/SRE teams. A user asks a natural-language question about a live incident ("checkout is slow since 14:20") and a bounded tool-calling LLM agent investigates across Prometheus metrics and Loki logs, returning a structured root-cause report with evidence and suggested actions.

### 1.1 Product goals

- **MVP (B-level):** fully usable for a solo developer or small team with their own observability stack. Self-contained sandbox so "clone and `make demo`" produces a working end-to-end demo in under 5 minutes without requiring any existing infra.
- **Iteration path (C-level):** pluggable LLM backend (Claude + Ollama) and zero-config sandbox make the repo approachable for open-source users; architecture supports later addition of Web UI, Slack/Discord bots, Kubernetes tools, and automation-on-approval.

### 1.2 Non-goals (MVP)

- Real Kubernetes integration (no kind cluster, no K8s tools in MVP — deferred to v0.2)
- Web UI (v0.2; backend HTTP API is designed to support it)
- Slack/Discord bot (v0.3)
- PII/secrets scrubbing in logs (MVP relies on explicit README disclaimer: "do not point this at production data")
- Distributed / multi-tenant deployment (single binary + single machine for MVP)
- Persistent multi-user accounts (MVP is single-user CLI)
- Trace replay provider (deferred to v0.2; MVP integration tests use a hardcoded `FakeProvider`)

## 2. Architecture

### 2.1 Topology

```
┌────────────────┐         ┌──────────────────────────────────────┐
│  mondevps CLI  │ ──HTTP─▶│  mondevps-server (Go, single binary) │
│  (thin client) │   +SSE  │                                      │
└────────────────┘         │  ┌─────────────────────────────────┐ │
                           │  │  Agent Engine                   │ │
                           │  │  ├─ LLM Provider (Claude/Ollama)│ │
                           │  │  ├─ Tool Registry               │ │
                           │  │  ├─ Loop Controller (budget)    │ │
                           │  │  └─ Trace Recorder (JSONL)      │ │
                           │  └──────────────┬──────────────────┘ │
                           │                 │                    │
                           │   ┌─────────────┼──────────────┐     │
                           │   ▼             ▼              ▼     │
                           │  Prom        Loki           Final    │
                           │  Tool        Tool           Report   │
                           └───┬─────────────┬──────────────┬─────┘
                               │             │              │
                   ┌───────────▼─────────────▼──────────────┴───────────┐
                   │  Sandbox (docker-compose, independent subtree)     │
                   │  • Prometheus + Loki + Grafana                     │
                   │  • 4 sample services (order/payment/inventory/...) │
                   │  • Chaos injector (`make inject-<scenario>`)       │
                   └────────────────────────────────────────────────────┘
```

### 2.2 Separation rationale

1. **CLI / server split** — CLI is a thin HTTP + SSE client. Same server can later serve a Web UI, Slack bot, or webhook-triggered investigations without changes to the agent engine.
2. **Agent engine is an in-process package**, not a separate service — no distributed complexity at MVP. Clean internal interface makes future extraction possible.
3. **Sandbox is an independent subtree (`sandbox/`)** — decoupled from the main program. Users run `make sandbox-up` and `make dev` separately. A repo-root `docker-compose.yml` wires both together for one-command demos.

### 2.3 Repo layout

```
AIOps/
├─ cmd/
│  ├─ mondevps/            # CLI entrypoint
│  └─ mondevps-server/     # server entrypoint
├─ internal/
│  ├─ agent/               # agent loop + trace recorder
│  ├─ llm/                 # provider interface + claude.go, ollama.go
│  ├─ tools/               # prometheus.go, loki.go, final_report.go
│  ├─ api/                 # HTTP handlers + SSE
│  └─ config/              # configuration loader
├─ sandbox/
│  ├─ docker-compose.yml
│  ├─ services/            # 4 Go sample services (order/payment/inventory/notify)
│  ├─ chaos/               # chaos templates + injection scripts
│  ├─ prometheus/          # prom config + alert rules
│  └─ loki/
├─ docs/
│  ├─ scenarios/           # one md per demo scenario
│  └─ superpowers/specs/   # design docs (this file)
├─ Makefile
├─ README.md
└─ docker-compose.yml      # one-command: server + sandbox together
```

## 3. Components

### 3.1 LLM Provider interface

```go
// internal/llm
type Provider interface {
    Name() string
    Chat(ctx context.Context, req ChatRequest) (ChatResponse, error)
}

type ChatRequest struct {
    System   string
    Messages []Message        // user / assistant / tool_result
    Tools    []ToolSpec       // JSON schema of available tools
}

type ChatResponse struct {
    StopReason string          // "end_turn" | "tool_use" | "max_tokens"
    Content    []ContentBlock  // text | tool_use
    Usage      TokenUsage
}
```

Two implementations:
- **`claude.go`** — official Anthropic Go SDK, native `tool_use` protocol.
- **`ollama.go`** — Ollama `/api/chat`; uses native tool calling if the model supports it, falls back to JSON-in-text parsing for weaker models. On parse failure, wraps the text as the `final_report` body.

Selection: `MONDEVPS_LLM=claude|ollama` env var, default `claude` if `ANTHROPIC_API_KEY` is set, else `ollama`.

### 3.2 Tool Registry (MVP: 3 tools total)

```go
type Tool interface {
    Spec() ToolSpec                                           // name, description, JSON schema
    Execute(ctx context.Context, args json.RawMessage) (ToolResult, error)
}
```

| Tool | Purpose | Key args |
|---|---|---|
| `query_prometheus` | Execute PromQL; return **server-summarized** time series (min/max/avg/p95 + anomaly points), not raw matrix | `query`, `start`, `end`, `step` |
| `search_logs` | Loki LogQL; return matched lines ranked by severity + time, capped at `limit` | `query`, `start`, `end`, `limit` |
| `final_report` | Terminator — agent emits structured final report | `summary`, `root_cause`, `evidence`, `suggested_actions` |

`list_services` is **not** a tool — the list of available services (name, description, port, dependencies) is injected into the system prompt at investigation start so the agent doesn't waste a turn discovering the environment.

**Critical design point:** `query_prometheus` returns a summarized payload, not raw `matrix` JSON. A 10-minute query with 1500 points compresses to something like: `"avg=120ms, p95=340ms, 2 spikes at 14:22:30 (peaked 1.8s) and 14:24:10 (peaked 2.1s)"`. This keeps each tool result under ~1KB and lets the agent reason efficiently.

### 3.3 Agent Loop Controller

```go
type Budget struct {
    MaxIterations int           // default 10
    MaxCostUSD    float64       // default 0.50 (Claude)
    MaxTokens     int           // default 100_000
    Timeout       time.Duration // default 2m
}

type Loop struct {
    provider llm.Provider
    tools    map[string]Tool
    budget   Budget
    trace    TraceRecorder
}

func (l *Loop) Run(ctx context.Context, query string) (*Report, error) {
    // 1. Compose system prompt (tool docs, service list, analysis guidance)
    // 2. for i := 0; i < budget.MaxIterations; i++:
    //      resp = provider.Chat(...)
    //      trace.Record(step)
    //      if resp is final_report tool_use: return report
    //      if resp is other tool_use: execute, append tool_result, continue
    //      if resp is plain text (no tool): coerce to final_report or stop
    //      check budget → early stop with PartialReport if exceeded
}
```

Termination conditions:
1. `final_report` tool call → normal termination
2. Budget / iteration / timeout hit → one "close-out" LLM call asking it to summarize from evidence gathered, flag as `budget_exceeded: true`
3. LLM returns no tool_use and no final_report → wrap text as `final_report`

### 3.4 Trace Recorder

Each investigation writes a `trace_<uuid>.jsonl` — one event per line:
```json
{"ts":"...","step":1,"type":"llm_request","model":"claude-opus-4-7","prompt_tokens":2341}
{"ts":"...","step":1,"type":"llm_response","stop":"tool_use","tool":"query_prometheus","args":{...}}
{"ts":"...","step":1,"type":"tool_result","tool":"query_prometheus","took_ms":142,"result_summary":"..."}
...
{"ts":"...","type":"final_report","root_cause":"...","total_cost_usd":0.18}
```

Uses: `mondevps traces list|show <id>`, README demo asset (`docs/scenarios/<name>/trace.jsonl`), future replay provider (v0.2) and golden regression tests.

### 3.5 Sandbox

- **4 Go microservices** (`order` → `payment` → `inventory`, plus `notify` as an order side-effect). All `net/http`, in-memory state, each exposes `/metrics` (Prometheus) and writes structured JSON logs to stdout.
- **Chaos injector** — each service exposes a `/chaos` control endpoint: `POST /chaos {"mode":"slow_db","params":{"ms":800}}`. Wrapped in `make inject-<scenario>` for one-command activation.
- **Prometheus + Loki + Grafana** — official images, pre-wired scrape config and datasources.
- **3 preset scenarios** (sufficient for MVP demos and resume story):
  1. `cascade-timeout` — payment slows → order times out → error rate spike. **MVP required.**
  2. `memory-leak` — inventory memory grows slowly → GC pressure → p99 latency spike. **MVP required.**
  3. `noisy-neighbor` — notify emits log flood → Loki pressure. **MVP stretch** (ship if cascade + leak finish on time; otherwise push to v0.2).

## 4. Data flow

### 4.1 End-to-end investigation

```
User:  $ mondevps investigate "checkout is slow since 14:20"

CLI:   POST /api/v1/investigations
       body: {"query":"..."}
       Accept: text/event-stream

Server: creates Investigation{ID, Query, StartedAt}
        opens SSE connection
        runs agent.Loop in goroutine
        streams trace events to CLI

Agent Loop (example):
  step 1  → query_prometheus(error rate)     → "5xx 0.1% → 8% at 14:20 on order-service"
  step 2  → query_prometheus(order p95)      → "80ms → 1.2s at 14:19:50"
  step 3  → search_logs(order errors)        → "147× 'payment timeout after 1000ms'"
  step 4  → query_prometheus(payment p95)    → "50ms → 2.5s at 14:19:30"
  step 5  → final_report{summary, root_cause, evidence[], actions[]}

CLI renders:
  ⏳ step 1: checking error rates...
  ⏳ step 2: checking order latency...
  ⏳ step 3: searching order-service logs...
  ⏳ step 4: checking payment latency...
  ✅ Done in 4 steps · 18s · $0.12

  ═══ Investigation Report ═══
  ... summary, root cause, evidence, suggested actions ...
  Full trace: ~/.mondevps/traces/<uuid>.jsonl
```

### 4.2 SSE event protocol

```
event: step
data: {"step":1,"phase":"llm_request"}

event: step
data: {"step":1,"phase":"tool_use","tool":"query_prometheus","args":{...}}

event: step
data: {"step":1,"phase":"tool_result","took_ms":142}

event: report
data: {"root_cause":"...","evidence":[...],"actions":[...]}

event: done
data: {"total_steps":5,"total_cost_usd":0.12,"duration_ms":18344}
```

The CLI consumes SSE and renders progress locally. A future Web UI consumes the same stream and renders charts.

### 4.3 Prompt strategy

System prompt skeleton:
```
You are an incident investigation agent. You have these tools:
<tool-list rendered from registry>

Available services in this environment:
<service list injected from sandbox config>

Rules:
1. Look at metrics first (error rates, latencies) before diving into logs.
2. Form hypothesis → query → refine. Don't call final_report until you can point to specific evidence.
3. Keep tool args tight. Prefer short time windows (1-5 min around the incident).
4. When you have a confident root cause with evidence, call final_report.

Investigation budget: max 10 tool calls, max $0.50.
Current time: <now>
```

User prompt: raw query + time context.

### 4.4 Persistence

- `~/.mondevps/traces/<uuid>.jsonl` — one file per investigation.
- `~/.mondevps/investigations.db` — SQLite, indexes `(id, query, started_at, summary)` for `mondevps list`.
- `MONDEVPS_TRACE_DIR` env var overrides default location.

No Redis/Postgres — single-binary promise.

## 5. Error handling

### 5.1 Error classification

| Category | Example | Strategy |
|---|---|---|
| LLM transient | 429, 503, timeout | Exponential backoff × 3, then terminate with `PartialReport{reason: "llm_unavailable"}` |
| LLM permanent | 401 auth, 400 bad request | Fail immediately with configuration guidance |
| Tool execution error | Prometheus 5xx, LogQL syntax error | **Do not fail the loop** — wrap as `ToolResult{IsError: true, Content: "prometheus returned 400: ..."}` and let the LLM self-correct |
| Tool timeout | Loki query > 10s | `context.WithTimeout` at tool layer; return `ToolResult{IsError: true, Content: "tool timed out after 10s"}` |
| Tool result too large | 10MB of logs | Truncate to 8KB, append `truncated: true, total_bytes: 10485760` |
| Budget exhausted | 10 turns / $0.50 / 2min hit | One close-out LLM call → partial report marked `budget_exceeded: true` |
| Agent stuck loop | Same tool + same args twice in a row | Inject system message reminding it; third time force `final_report` |
| Sandbox not running | Can't reach server or Prometheus | Friendly error: `Sandbox not reachable. Run \`make sandbox-up\` and try again.` |
| LLM config missing | No `ANTHROPIC_API_KEY` and no Ollama | Fail fast at server start, print configuration options |
| Config conflict | Claude key set but `MONDEVPS_LLM=ollama` | Explicit env var wins, log a warning |

### 5.2 Safety

- **Pre-call budget check** — every LLM request first verifies cumulative cost/tokens against the hard cap; refuses before spending, not after.
- **Tool arg allowlist** — `query_prometheus` and `search_logs` accept read-only query syntax only; admin endpoints are rejected at the tool layer.
- **No automated mutations** — MVP is suggestion-only. Automated apply (e.g. one-click HPA change) is a C-phase feature with a separate approval flow.
- **Trace files** — local disk only, 0600 permissions.
- **PII/secrets** — MVP relies on README disclaimer ("do not point this at production data"). Regex-based scrubber deferred to v0.2.

### 5.3 Degradation

- **Ollama weak tool_use** — provider falls back to JSON-in-text parsing; on parse failure, wraps raw text as `final_report.summary`.
- **Prometheus no data** — `query_prometheus` returns `"result: no data in range"` (not an error) so the agent can adjust window.
- **One tool unreachable, others fine** — investigation continues; final report flagged `"degraded: loki unreachable"`.

### 5.4 Self-observability

- Server exposes `/metrics` in Prometheus format. Sandbox scrapes it. An investigation can query mondevps's own LLM-call latency — the agent can investigate itself. Demo easter egg and useful dogfooding.
- Structured logging (`log/slog`) to stdout.

## 6. Testing

### 6.1 Test pyramid

```
                    ┌───────────────────────┐
                    │  Scenario E2E (3)     │  Real Claude + real sandbox
                    └───────────────────────┘
                ┌───────────────────────────────┐
                │  Integration (10-15)          │  FakeProvider, real Prom/Loki
                └───────────────────────────────┘
            ┌───────────────────────────────────────┐
            │  Unit (50+)                           │  Mock everything external
            └───────────────────────────────────────┘
```

### 6.2 Unit tests (run on every CI)

- **Each tool** — mock HTTP client; cover normal, error, empty, and truncation paths.
- **Agent loop** — with `FakeProvider` (hardcoded tool_use sequences):
  - Normal termination via `final_report`
  - Budget exhaustion (`MaxIterations=2` without final → close-out call)
  - Stuck-loop detection (same tool + args twice → system reminder)
  - Tool error doesn't fail loop (error propagates to agent as tool_result)
- **Trace recorder** — concurrent writes, format compatibility.
- **Budget accounting** — cost calculation for each LLM usage shape.

Target: `go test ./...` completes in under 5 seconds.

### 6.3 Integration tests (run on every CI)

Uses a hardcoded **`FakeProvider`** that returns a scripted sequence of `tool_use` responses. Real Prometheus and Loki run in test containers, real chaos is injected, real tool calls execute — only the LLM is faked.

Example:
```go
func TestInvestigate_CascadeTimeout(t *testing.T) {
    sandbox := startTestSandbox(t)
    sandbox.InjectScenario("cascade-timeout")
    server := newServer(
        WithLLM(NewFakeProvider(cascadeTimeoutScript)),
        WithToolsFor(sandbox),
    )
    report := server.Investigate(ctx, "checkout is slow since 14:20")
    assert.Contains(t, report.RootCause, "payment")
    assert.LessOrEqual(t, report.Steps, 6)
    assert.False(t, report.BudgetExceeded)
}
```

**`TraceReplayProvider`** (replay of real recorded Claude traces) is deferred to v0.2.

### 6.4 Scenario E2E (manual / nightly CI)

- Real Claude + real sandbox; each preset scenario runs once.
- Loose assertions to avoid flake:
  - Final report must mention the **expected service name**.
  - Steps ≤ per-scenario cap (e.g. cascade-timeout ≤ 8).
  - Cost ≤ cap.
- Successful traces are saved to `testdata/traces/` as regression baselines and as README GIF source material.

### 6.5 Ollama compatibility (separate, manual)

- Local `qwen2.5:7b` or `llama3.1:8b` over 3 scenarios.
- Looser assertions: termination without timeout, final report mentions at least one relevant service.
- README acknowledges "Ollama quality may vary — Claude is recommended for best results."

### 6.6 CI structure

```
on PR:
  ✓ lint (golangci-lint)
  ✓ unit tests
  ✓ integration tests (FakeProvider + real Prom/Loki)

nightly / on-demand:
  ✓ scenario E2E (real Claude × 3)
  ✓ Ollama compat × 3 (degraded assertions)
```

E2E jobs use a repo-secret Anthropic key. PRs do not run E2E (cost + key exfiltration risk on external PRs).

### 6.7 Pre-release demo checklist (manual)

- [ ] `docker-compose up` starts cleanly with no warnings
- [ ] 3 scenarios each produce a sensible report on visual inspection
- [ ] README GIFs reproduce
- [ ] `mondevps replay <id>` works offline

## 7. Open decisions for v0.2+

- `TraceReplayProvider` (replay recorded Claude traces in integration tests)
- Web UI (chat + chart visualization, shares SSE stream)
- K8s tools (`describe_pod`, `get_events`) with kind cluster in sandbox
- K8s scaling recommendations (from resume bullet #2)
- PII/secrets regex scrubber
- Slack/Discord bot frontend
- Webhook-triggered investigations (alert-driven auto-RCA)
- Automated action execution (with approval flow)

## 8. Success criteria

MVP is done when all of the following hold:

1. Fresh clone → `make demo` → within 5 minutes a user sees: sandbox running, cascade-timeout scenario injected, agent produces a correct root-cause report pointing at `payment-service`.
2. All 3 preset scenarios produce a sensible report when run with Claude.
3. `go test ./...` passes; CI green on unit + integration.
4. README has at least one recorded GIF demonstrating the agent investigating a scenario end-to-end.
5. Single static binaries: `mondevps` and `mondevps-server`, cross-compiled for linux/amd64, darwin/arm64, windows/amd64.
