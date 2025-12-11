# Agent operations playbook for personal multi-role assistants

This guide distills best practices for designing, orchestrating, and
running agents that cover your life domains (business owner, lender,
developer, real estate developer, coder, husband, friend, etc.). It is
written for beginners who learn by doing but want structure to scale
reliably over time.

### Quick-start blueprint
1. **Pin the stack**: choose the model/SDK version, enable telemetry, and
   set secrets via environment variables.
2. **Define roles**: capture a 150–200 word persona for each life domain
   with goals, tone, do/do-not rules, and escalation criteria.
3. **Stand up memory**: create a long-term store (facts, preferences,
   identifiers) plus a short-term scratchpad; seed a few starter facts to
   validate recall.
4. **Attach tools**: map each role to a minimal set of trusted tools with
   schemas and dry-run switches before adding more.
5. **Add a router**: instruct a coordinator agent to pick experts, merge
   their answers, and surface citations to memory or tools used.
6. **Test weekly**: run a small scenario suite per role, review logs, and
   adjust prompts or tools based on drift.

## 1. Environment and tooling
- Use version control for agent prompts, configs, and tool schemas; prefer
  feature branches plus pull requests for traceability.
- Pin model and tool versions (Gemini model name, SDK version) in a single
  env file so agents are reproducible across machines.
- Keep secrets out of configs by wiring environment variables and using a
  secrets manager when possible.
- Run linting/formatting and unit tests before deploys; automate with CI to
  catch prompt or tool regressions.

## 2. Role modeling and instructions
- Create a short persona per role: goal, scope, tone, and decision guardrails
  (e.g., "optimize for cash flow" vs. "optimize for learning"). Include a
  confidence/uncertainty behavior (e.g., "ask before spending"), and an
  escalation trigger (e.g., "hand off if tax/legal ambiguity").
- Keep per-role instructions under ~200 words with explicit do/do-not lists
  to avoid drift.
- Standardize the final-answer format (summary + next steps) so outputs stay
  predictable across domains.
- For sensitive areas (finance, lending, personal relationships), add
  explicit consent and double-check steps before irreversible actions.

## 3. Memory and recall
- Separate long-term facts (preferences, accounts, recurring tasks) from
  short-term session notes; load only what each role needs.
- Store memory with timestamps and provenance (who said it, when) so the
  agent can resolve conflicts.
- Provide retrieval prompts that summarize retrieved memories before
  answering; include a freshness preference (prefer last 30 days unless
  overridden).
- Periodically prune or archive stale memories to keep context windows
  focused.
- Add a "must remember" registry for critical facts (e.g., lenders,
  entities, family preferences) that is always loaded for relevant roles;
  require citations to this registry in answers that rely on it.
- When a response changes state (new commitment, updated contact), write a
  short memory entry immediately with who/what/when so recall stays fresh.

## 4. Tooling strategy
- Map tools to roles: business (finance APIs, CRM), lending (rate sheets,
  calculators), development (repo access, linters), social (calendar, maps).
- Validate tool inputs with explicit schemas and sanity checks before calls;
  fail fast with helpful error messages for missing required fields.
- Add a "dry-run" mode for tools that cause side effects (emails, calendar,
  transactions) so you can review before execution.
- Log all tool calls with inputs/outputs for auditability; surface concise
  tool summaries in the agent's response when decisions depend on them.

## 5. Orchestration patterns
- Use a coordinator agent to route requests to domain experts; include a
  short router prompt that lists roles, triggers, and fallbacks.
- Keep cross-role handoffs explicit: each expert returns a decision and a
  short rationale; the coordinator stitches a final answer with sources
  (tool results, memories) noted.
- Prefer parallel tool calls for independent data fetches, but serialize
  when actions depend on prior validation (e.g., credit checks).
- Add circuit breakers: max tool calls per turn and a fallback message when
  thresholds are hit.

## 6. Planning and execution discipline
- For any task beyond a quick lookup, require a lightweight plan (2–5
  milestones) before tool calls; keep exactly one item in progress.
- Update the plan after each major step and close items with outcomes, not
  just "done".
- Encourage agents to proceed by default when intent is clear; only pause
  for confirmation on irreversible actions or missing safety details.

## 7. Evaluation and safety
- Maintain scenario suites per role (e.g., loan scenarios, code refactors,
  property underwriting) and run them after prompt or tool updates.
- Include adversarial cases (ambiguous asks, conflicting memories, partial
  data) to test routing and safety fallbacks.
- Track regressions with a simple dashboard: pass/fail counts, latency, and
  tool-call volume per scenario.
- For finance/legal outputs, require citations and a "not financial/legal
  advice" reminder unless a human reviewed the response.

## 8. Operational cadence
- Review logs weekly to tune prompts, prune memories, and refresh tool
  credentials.
- Rotate secrets regularly and monitor rate limits to avoid outages.
- Keep a rollback plan: previous prompt versions, prior tool schemas, and
  a known-good dependency set.
- Document playbooks for common workflows (loan application prep, deal
  evaluation, code shipping) and keep them under version control.

## 9. Growing over time
- Add new roles as separate modules with their own test sets before merging
  into the coordinator.
- Revisit personas quarterly to align with evolving goals (impact, fun,
  learning) and retire roles that no longer add value.
- Instrument user feedback loops (thumbs, short surveys) to capture drift
  early and prioritize fixes.
