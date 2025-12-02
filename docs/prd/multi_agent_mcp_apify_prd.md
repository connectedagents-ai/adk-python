# Multi-Agent MCP Integration PRD

## Objective
Deliver a production-ready integration that lets the ADK-powered multi-agent
framework ("LitigationEnforcer/LUT Force") call Anthropic Model Context
Protocol (MCP) tools backed by Apify actors. The goal is to enable agents to
autonomously select and run Apify-powered web data workflows (e.g., discovery,
enrichment, monitoring) while preserving compliance, observability, and safe
execution controls.

## Background
- MCP standardizes how LLM clients invoke external tools via a client/server
  model. Claude Desktop (client) can call an Apify MCP server to run Actors and
  return results.
- Apify provides 5,000+ actors (scrapers, automations, enrichment utilities).
  The Apify MCP server exposes selected actors through MCP and handles
  authentication via API tokens.
- Current pain points: inconsistent tool access across agents, manual actor
  selection, weak safety controls, and limited visibility into cross-tool
  workflows.

## Goals
- Allow ADK agents to dynamically choose MCP tools (Apify actors) as part of
  task planning and execution.
- Provide configurable allow/deny lists of actors per agent or workspace to
  reduce hallucinated or unsafe tool use.
- Standardize input/output handling so agents can request structured data
  tables or files directly from actor runs.
- Preserve auditability with run-level provenance (actor, inputs, outputs,
  timestamps, user/agent identity).
- Keep latency reasonable for common tasks (<30s p95 for default actors) with
  timeout and retry policies.

## Non-Goals
- Replacing existing built-in tools (search/code execution) unless explicitly
  routed via planner.
- Full catalog auto-exposure of all Apify actors (scope will be curated and
  configurable).
- Building a custom UI; focus is SDK-level and CLI-level integration first.

## User Stories
1. **Research Agent**: As an analyst agent, I can request "Find 20 pizza
   places in Prague with website + phone" and the planner selects a configured
   Apify actor (e.g., Google Maps extractor) via MCP, returning a table.
2. **Ops Lead**: As an admin, I can configure which actors are available to a
   team and set per-actor timeouts, quotas, and concurrency limits.
3. **Developer**: As an SDK user, I can add MCP tool access to a custom agent
   by providing the Apify token and actor allow-list without editing core
   code.
4. **Compliance Officer**: As a reviewer, I can see logged evidence of which
   agent invoked which actor, with inputs/outputs stored for auditing.

## Architecture & Workflow
- **Components**
  - MCP client: ADK agent runtime (and optionally Claude Desktop for local
    testing).
  - MCP server: Apify MCP server with curated actor list.
  - Control plane: configuration store for allow/deny lists, quotas, and
    timeouts (env vars + YAML/JSON profiles).
  - Observability: structured logging, metrics, and trace exporters.
- **Data Flow**
  1. Planner scores tools (Apify actors + native tools) using task context,
     cost/latency hints, and allow/deny constraints.
  2. Executor issues MCP `callTool` with actor name, inputs, timeout, and
     expected output shape.
  3. MCP server launches Apify actor run; emits progress events; final payload
     returned to agent.
  4. Post-processor normalizes payload into `ToolResult`, attaches provenance,
     and triggers optional exports (CSV/Parquet, signed URLs).
  5. Telemetry pipeline logs metrics and traces correlated to agent/session.
- **Lifecycle Hooks**
  - Pre-flight validation: schema check, input size limit, domain safelist.
  - Circuit breakers: actor health scores and backoff when failure rate
    exceeds threshold.
  - Fallbacks: alternate actor suggestion or built-in search when MCP path
    fails.

## Functional Requirements
- **Tool Registration**
  - Add MCP tool source for Apify actors with metadata (name, description,
    input schema, output schema when available).
  - Support configuration via YAML/JSON and environment variables: `APIFY_TOKEN`,
    `APIFY_MCP_ENDPOINT`, `MCP_ACTOR_ALLOWLIST`.
- **Tool Discovery & Selection**
  - Agents can enumerate allowed actors and ask for recommendations based on
    task description.
  - Planner enforces allow/deny list and respects per-actor cost/latency
    hints.
- **Invocation Pipeline**
  - Standard request contract: actor name, input payload, expected output
    format (JSON/table/file), timeout, retries.
  - Stream progress events (submitted, running, completed, failed) for
    observability.
  - Support long-running actors with polling and partial results where
    available.
- **Data Handling**
  - Normalize actor outputs into ADK `ToolResult` shape with schema
    validation; attach raw payload for traceability.
  - Optional post-processing: table rendering, CSV/Parquet export, and signed
    URL return when files are produced.
- **Security & Compliance**
  - Do not log raw secrets; redact tokens in traces.
  - Enforce per-actor rate limits and maximum input sizes.
  - Add configurable domain/URL safelist for scraping-centric actors when
    required.
- **Observability**
  - Emit structured logs and metrics: actor name, duration, status, retries,
    bytes returned.
  - Correlate MCP runs with agent/session IDs for end-to-end traceability.
- **Failure Handling**
  - Clear error surfaces for common cases: auth failure, missing actor,
    timeout, schema mismatch.
  - Planner fallback path: suggest alternate actor or degrade to built-in
    search when Apify is unavailable.

## Architecture Decisions & Acceptance Criteria
- MCP tool invocation must reuse the ADK tool abstraction rather than a custom
  executor path to preserve planner compatibility.
- All tool metadata (schemas, cost/latency hints) must be cacheable to avoid
  repeated MCP discovery calls at runtime.
- Config changes (allow-lists, quotas) must be hot-reloadable without agent
  process restart where possible; otherwise, surface a clear restart warning.
- Acceptance: a demo scenario (pizza joints in Prague) runs end-to-end with
  structured table output, provenance, and telemetry; automated tests cover
  success, timeout, auth failure, schema mismatch, and fallback behaviors.

## Configuration & Deployment
- **Local (Claude Desktop)**: Provide sample MCP config snippet with
  placeholders for `APIFY_TOKEN`, endpoint, and actor list; document restart
  requirement after edits. Include a minimal `.adk` profile for local agents
  using the same variables.
- **Server/CI**: Environment-variable driven configuration for headless
  agents; secrets injected via vault/secret manager. Provide Helm/compose
  values examples for MCP endpoint and timeouts.
- **Profiles**: Support multiple profiles (dev/stage/prod) with different
  allow-lists and quotas. Expose dry-run mode to validate profiles without
  invoking actors.

## Operational Playbooks
- **Incident Response**:
  - Triage checklist: verify MCP endpoint health, token validity, actor
    availability, and rate-limit status.
  - Runbook for mass-failure: disable MCP tool source via feature flag and
    fall back to built-in tools; page on-call if MTTR >30 minutes.
- **Change Management**:
  - Add/remove actors via PR to config profiles with owner sign-off and
    schema validation gate.
  - Schedule load tests for new high-volume actors; monitor p95 latency and
    failure rate for 24 hours post-change.
- **Compliance**:
  - Quarterly review of actor allow-lists against domain safelists and data
    retention policies.

## Non-Functional Requirements
- **Performance**: Default actor invocations p95 <30s; planner should avoid
  actors with historical p95 >60s unless explicitly requested.
- **Reliability**: Retry transient failures (network/timeouts) with exponential
  backoff; mark actors unhealthy after configurable failure threshold.
- **Compatibility**: Works on Linux/macOS; supports Claude Desktop config
  schema for local MCP testing.
- **Maintainability**: Configuration-driven; minimal code changes required to
  onboard new actors.

## Testing & Validation
- Unit tests for MCP tool adapter: request/response normalization, schema
  validation errors, redaction behavior.
- Integration tests against mocked MCP server covering auth errors, timeouts,
  retries, and progress events.
- Soak test plan: 500 sequential runs of top 5 actors to measure p95 latency
  and failure rates.
- UAT checklist for sample workflows (pizza search, lead enrichment, site
  monitoring) with screenshots or saved artifacts.

## Success Metrics
- 90% of eligible agent tasks use MCP actors without manual intervention
  within two weeks of rollout.
- <2% tool-call failures attributable to configuration errors after initial
  setup.
- Mean planning latency increase <300ms when MCP toolset is enabled.
- Audit completeness: 100% of MCP calls include actor name, inputs
  (redacted), outputs, and timestamps in traces.

## Risks & Mitigations
- **Actor Variability**: Schemas differ across actors. Mitigation: maintain
  curated allow-list with validated schemas and adapters.
- **Cost Spikes**: Large outputs increase token + actor charges. Mitigation:
  enforce output size caps and sampling for previews.
- **Timeouts on Free Plans**: Apify/Claude limits may cause failures.
  Mitigation: document plan requirements and expose configurable timeouts.
- **Data Sensitivity**: Scraping PII or restricted domains. Mitigation: domain
  safelist, compliance review gates, and encrypted storage of outputs.

## Rollout Plan
1. **Alpha**: Wire MCP client with 1–2 vetted actors (e.g.,
   `apify/actor-google-maps-scraper`) behind feature flag; manual monitoring;
   complete demo runbook and regression tests.
2. **Beta**: Expand allow-list to 10–20 actors; enable planner-based
   selection; add dashboards for success/failure rates; run soak tests and
   finalize schema adapters.
3. **GA**: Default-on for eligible agents; publish cookbook examples; finalize
   SLOs and escalation playbooks; enable automated config validation in CI.

## Open Questions
- Should we auto-learn latency/cost profiles per actor to inform planner
  scoring?
- Do we need per-tenant quotas distinct from per-actor limits?
- What is the required retention period for MCP call logs and outputs?
- Should we support per-actor privacy labels (e.g., disallow PII extraction)
  that influence planner routing and audit requirements?
