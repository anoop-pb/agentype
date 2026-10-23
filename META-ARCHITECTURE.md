# Meta Architecture for Agent-First Business Apps

Version 1.5 · 23 October 2026

---

## 1. Purpose

This document sets the architecture rules for agent-first business apps. In these apps, users work through the runtime agent (D3).

Each app gets its own design document, based on this one. The app design carries forward every rule here, adapted to the app, and may change any of them in conversation with the human at design time. Once the app design exists, this document is removed from the app's repo. The app design is then the only source of rules.

Based on the app design, the core must be developed first, as its own phase, before any domain code. That first phase contains the adapter, tenant context, RLS, confirmation, outbox, and checks. It is verified on its own before anything is built on it.

---

## 2. Core principle

**The agent decides what to do. Code guarantees how it is done.**

- The agent's flexibility goes into intent, exploration, judgment, and UI.
- Writes, business rules, and business-critical numbers are deterministic code.
- New capabilities are added at build time, never at runtime.

---

## 3. Terms

| Term | Meaning |
|---|---|
| Tenant | One customer company using the app. (In case of a multi-tenant app) |
| Business area | One part of the business with one responsible owner, its own records, and its own rules. Others ask it to do things. They never change its records directly. Each area is one folder under `domain/`. |
| Operation | One action the app can perform, such as `approve_refund`. Holds one workflow. Defined once, with its schema and settings. |
| Helper | An internal business function inside an area. Never exposed as a tool. |
| Adapter | One generic piece of code that registers operations as MCP tools and runs every tool call. |
| System tool | A tool that is not a business operation, such as `run_query`, `describe_views`, and `file_change_request`. Lives in `platform/`, not `domain/`. |
| Tool | An operation or system tool that is registered with MCP. |
| Curated view | A DB view that the runtime agent may query. The read contract. |
| Lifecycle | The declared list of allowed state changes for an area's records. |
| Template | A screen-level UI layout in A2UI, built at build time, filled with data at runtime. |
| Runtime agent | The agent that app users talk to. It may use sub-agents (D9). |
| Coding agent | The agent that writes the app's code. |
| Verification agent | An independent agent that checks code against the app design. |

---

## 4. Layers

```
              User (web / mobile)
                    │  A2UI
                    ▼
              Runtime agent
                    │  MCP tool calls
                    ▼
┌────────────────────────────────────────┐
│ Adapter (one, generic)                 │
│ identity, tenant, permission, confirm, │
│ validation, logging, call operation    │
└───────────────────┬────────────────────┘
                    ▼
┌─ domain/ ──────────────────────────────┐
│  <area>/                               │
│    Operations (opt-in tools)           │
│      one workflow each                 │
│    Helpers (internal)                  │
│    Lifecycle (allowed state changes)   │
│    Models (schemas)                    │
└──────┬────────────────────────┬────────┘
       ▼                        ▼
┌──────────────────┐   ┌──────────────────┐
│ Postgres         │   │ Outbox           │
│ constraints, RLS │   │   ▼              │
│ curated views    │   │ Worker           │
│ audit triggers   │   │   ▼              │
└──────────────────┘   │ integrations/    │
                       └──────────────────┘
```

System tools live in `platform/`, beside `domain/`. They go through the same adapter, with the same checks.

Batch jobs and inbound webhooks enter through MCP or fixed operations, with their own service identity.

---

## 5. Stack requirements

Most stack choices depend on the app's domain and purpose.
Some are fixed, because the decisions below depend on them.

**Fixed**
- Postgres, or a database with the same features: row-level security, `security_invoker` views, and transaction-level settings.
- An MCP server SDK.
- An A2UI renderer for each client.
- A language with schema-first models that produce JSON Schema. Examples: TypeScript with Zod, Python with Pydantic.
- A key management service that supports envelope encryption.

**Chosen per app**
- Language and frameworks.
- Hosting.
- Model provider and models.
- Client platforms.
- Integrations.
- A durable workflow engine, only if needed (D5).

---

## 6. Repository layout

File names are examples. The structure is the rule.

```
adapter/              generic MCP adapter
domain/
  <area>/
    models            input schemas, types, sensitive-column marks
    operations        opt-in tools, one workflow each
    rules             internal helpers
    lifecycle         allowed state changes (if the area has states)
platform/             system tools, not business operations
                      run_query, describe_views, file_change_request
integrations/
  <service>/
    client            standard client contract
    fake              fake client for tests
worker/               outbox worker
jobs/                 batch jobs (MCP clients)
ui/templates/         A2UI screen templates
db/
  migrations/
  views/              curated views with comments
checks/               automated architecture checks
evals/                runtime agent eval suite
tests/
```

---

## 7. Operation definition

Every operation is defined once with these fields.
The adapter reads them. Automated checks verify them.

System tools use the same definition format and the same fields. Their `area` is `platform`. They follow every rule that applies to operations, including `permission` and `confirm`.


| Field | Meaning |
|---|---|
| `name` | The tool name the agent sees, such as `approve_refund`. |
| `description` | What the operation does, in user terms. The agent reads this to choose tools. |
| `area` | The business area that owns the operation (D6). |
| `input` | The input schema. The same schema drives forms, validation, and the tool call (D2). |
| `function` | The code that runs the workflow (D4, D5). |
| `kind` | `read` if it only reads data. `write` if it changes data. |
| `exposed` | `true` registers the operation as a tool. `false` keeps it internal. Default is `false`. |
| `permission` | The permission a user's role must have to call this operation, such as `refunds.approve`. Roles are mapped to permissions in the roles model (D10). The adapter checks it before running. If this field is missing, the operation is not registered (default deny). |
| `confirm` | Whether the user must approve before the operation runs (D9). `none`: runs at once. `user`: the adapter does not run it. It shows the user the exact payload on a system confirmation screen. It runs only after the user clicks confirm. The agent cannot confirm on the user's behalf. |
| `admin` | `true` marks an admin operation. It is hidden unless the user has turned on admin mode in the UI (D9). Its `permission` must be one held only by admin roles. |
| `outbound` | `true` if the operation causes an action outside the app, such as sending an email (D11). Forces `confirm: user`. |
| `display` | Optional. Suggests the answer form: `text`, `table`, `chart`, or `dashboard` (D15). Recommended for business-critical reads. |

---

## 8. Decisions

Each decision lists what is decided, why, and how it is enforced.
Enforcement is one of:
- **Check:** automated, fails the build.
- **Verify:** reviewed by the verification agent.
- **Eval:** tested in the runtime agent eval suite.

### A. Structure

#### D4. Operations, adapter, and domain package
- One generic MCP adapter for all tools.
- One `domain/` folder, organized by business area only.
- Each area has models, operations, helpers, and a lifecycle if it has states.
- Each operation is defined once (section 7).
- The operation holds the workflow and calls internal helpers.
- Only opted-in operations become tools.
- System tools live in `platform/` and use the same definition format (section 7).

*Rationale:* A single definition removes hand-written mapping, and rules stay testable without MCP. System tools in the same format follow the same rules, without posing as a business area.

*Enforcement:* Check: every tool maps to one definition, either an operation in `domain/` or a system tool in `platform/`. Check: no MCP handlers exist outside the adapter.

#### D5. Workflows and lifecycles
- Workflows are code, inside operations.
- Areas with states declare allowed transitions in a lifecycle file.
- One shared function checks every transition.
- Human approvals are lifecycle states stored in the DB.
- Constants live in code.
- Config files and a durable workflow engine are added only when needed. A durable engine is needed only for timers, retries across steps, or multi-step rollback.

*Rationale:* One list of allowed state changes per area means every operation checks states the same way. Config files and a workflow engine are added only when needed, to keep the design simple.

*Enforcement:* Verify: every state change goes through the shared transition function.

#### D6. Business areas
- A business area is one part of the business with one responsible owner, its own records, and its own rules.
- Others ask it to do things. They never change its records directly.
- Each area lives in one folder under `domain/`.

Engineering checks:
1. **Each table belongs to exactly one area.** Only code in that area's folder may write to it. If no single owner can be named, the boundaries are wrong.
2. **Cross-area changes go through the owning area's functions.** Example: `refunds` calls `ledger.post_entry()`. It never inserts into the ledger table. If most operations need many cross-area calls, the two areas should probably be one.
3. **One meaning per term inside an area.** "Account" can mean a login in `customers` and a money account in `ledger`. Within one folder it means one thing, and its model matches that meaning.

*Rationale:* Owners, engineers, and every AI run draw the same boundaries.

*Enforcement:* Check: every table is listed as owned by one area. Check: no writes to tables owned by another area. Verify: area boundaries and terms.

### B. Data and enforcement

#### D1. Database vs code
- Data invariants live in the DB: NOT NULL, foreign keys, CHECK, unique constraints.
- RLS is a backstop for every table.
- Business rules and workflows live in code.
- Triggers are used only for audit logging.
- The agent never holds DB credentials that bypass the code.

*Rationale:* Code is easier to test and review than stored procedures, but the DB must still hold when that code is wrong.

*Enforcement:* Check: RLS is enabled on every table. Check: no triggers other than audit triggers. Check: no stored procedures hold business logic. Check: the agent's DB role has no write grants.

#### D12. Schema coupling
- The runtime agent sees only tool schemas and curated views.
- It learns them at runtime, through MCP discovery and a `describe_views` tool. Nothing about the schema is in prompts.
- Views are the read contract. View changes are reviewed like API changes.
- Migrations use expand, migrate, contract.
- Form payloads built from an older schema are rejected with a clear error, and the agent re-renders the form.

*Rationale:* Internal tables can change freely without breaking the agent.

*Enforcement:* Check: every curated view has a comment. Check: prompts contain no table names. Verify: migrations follow expand, migrate, contract.

#### D21. Concurrency
- Prefer append-only rows for activities, logs, and history.
- Where state is updated in place, use a version check per record.
- Example: a deal's current stage uses a version check, and each stage change is also appended to a history table.

*Rationale:* Append-only rows avoid conflicts, and version checks catch the rest.

*Enforcement:* Check: tables updated in place have a version column. Verify: append-only is used where suitable.

### C. Agent interface

#### D2. Forms and input
- Each form is bound to one MCP tool.
- The tool's input schema drives rendering, validation, and submission. There is no separate payload type.
- Submitted values pass to the tool unchanged.
- If the agent wants to change a value, it shows the user first.
- Simple inputs of 1–3 fields can be gathered in conversation, validated against the same schema.
- When input is incomplete, from conversation or a suggested action, the agent shows the tool's form. The form is prefilled with known values. Missing required fields are highlighted.

*Rationale:* One schema means one place to maintain. Pass-through stops the agent from altering what the user entered. Prefilled forms finish the task in one step.

*Enforcement:* Check: forms are rendered only from tool schemas. Check: the adapter validates every payload against the tool schema. Eval: form values pass through unchanged.

#### D3. Single entry point and batch jobs
- MCP is the single entry point for users and batch jobs. Inbound webhooks call fixed operations directly (D11).
- There are no non-agent clients.
- Batch jobs are code that calls MCP tools, with their own narrowly scoped service identity.
- Agents are used in batch jobs only for steps that need judgment.

*Rationale:* One entry point avoids rule drift, and plain code is cheaper and deterministic for routine jobs.

*Enforcement:* Check: batch jobs use only the MCP client. Verify: service identities have only the permissions their jobs need.

#### D7. Fixed operations and ad hoc reads
- All writes are fixed operations, generated at build time. This includes CRUD for simple areas.
- All frequent and business-critical reads are fixed operations.
- The product owner defines which reads are business-critical. Rule of thumb: any number people compare over time, report upward, or use for money or decisions.
- Frequent reads are identified from logs against a set threshold.
- Other reads may be ad hoc, through one `run_query` tool:
  - read-only SQL on curated views only,
  - under RLS, running as the end user,
  - every curated view is created at build time with `security_invoker`, so RLS applies to the end user,
  - with statement timeouts and row limits.
- Every tool call is logged in one format and can be replayed.
- The runtime agent never creates operations.

*Rationale:* Numbers people compare over time must be deterministic, and the agent's flexibility goes into intent, exploration, and judgment.

*Enforcement:* Check: `run_query` uses a read-only role with timeouts and row limits. Check: every curated view uses `security_invoker`. Verify: every business-critical read in the app design is a fixed operation.

### D. Security

#### D9. Agent permissions and prompt injection
- The agent has no privileges of its own. Every tool call runs as the end user.
- Each app has one runtime agent. Any sub-agent it uses runs as the same end user, calls tools only through the adapter, and shares the session's trace ID and budget. A sub-agent has no memory of its own.
- Each operation declares `confirm: none | user`.
- The adapter enforces confirmation. It returns "confirmation required" instead of running. Only a user action in the UI completes it. The agent cannot confirm on the user's behalf.
- Outbound operations always need confirmation.
- Admin operations are hidden unless the user turns on admin mode in the UI. Only users with an admin role can turn it on. Admin mode grants no permissions. It only reveals admin operations the user is already permitted to run. Admin mode expires after a set time. The agent cannot turn it on.
- Secrets never enter the agent's context.

*Rationale:* Injection cannot be fully prevented, so its reach is limited to what the user could do, would approve, and has explicitly elevated to.

*Enforcement:* Check: every operation declares `confirm`. Check: every `outbound` operation has `confirm: user`. Check: every operation with `admin: true` has a `permission` held only by admin roles. Eval: injection attempts from user input, DB text, and inbound email. Verify: sub-agents follow the same identity, adapter, trace, and budget rules.

#### D10. Authorization
- Row visibility is RLS, based on tenant, ownership, team, and role.
- There is no per-record sharing.
- Action permissions are declared per operation and checked by the adapter. Default deny.
- Data-dependent rules live in helpers. Example: discounts over 20% need a manager.
- RLS and code share one roles model: users, roles, and teams in one set of tables.
- Allow/deny tests are generated per operation and role.

*Rationale:* Declared, centrally checked permissions cannot be skipped, and generated tests catch gaps in AI-written code.

*Enforcement:* Check: every exposed operation declares `permission`. Check: generated allow/deny tests pass.

#### D20. Multi-tenancy
- Shared tables, with `tenant_id` on every table and RLS.
- The tenant is part of every RLS policy, every unique key, and every foreign key.
- Tenant context is set per transaction, never per connection.
- All shared components carry the tenant: outbox, worker, batch jobs, briefings, logs, memory, and caches.
- Integrations use per-tenant credentials.
- Budgets apply per tenant.
- Each user belongs to one tenant.

*Rationale:* Simple to run, with isolation enforced by the DB and tested.

*Enforcement:* Check: `tenant_id` on every table, in every RLS policy, unique key, and foreign key. Check: cross-tenant isolation tests pass.

#### D22. Per-tenant encryption
D22 also applies to single-tenant apps. There, the app has one key for its data.

- Sensitive columns are marked in models.
- Marked columns are always encrypted with a per-tenant key, using envelope encryption.
- Keys are generated and managed by the app, one per tenant.
- Tenant keys live in a managed key service, with deletion protection turned on.
- The app can only encrypt and decrypt. It cannot delete keys.
- Deleting a key needs two people and a waiting period.
- An alert fires when a key stops working.
- Encryption is column-level.
- Encrypted columns are excluded from curated views and ad hoc reads. They are available only through fixed operations.
- Keyed hashes allow exact-match lookup where needed.
- Decrypted values never appear in logs, traces, memory, or briefings.
- Audit values for these columns stay encrypted.
- Keys rotate on a schedule.
- Deleting a tenant's key is the deletion method for its data.
- Support for Tenant-supplied keys only if needed, as confirmed at design time.

*Rationale:* One code path for all tenants, strong isolation of sensitive data, and clean tenant deletion.

*Enforcement:* Check: every marked column is encrypted. Check: no marked column appears in a curated view. Check: the app's role has no right to delete keys.

### E. External systems

#### D11. Side effects and integrations
- Operations never call external services directly.
- An operation writes external actions to an outbox in the same transaction as the business change.
- One worker sends them using a standard client contract: action, payload, and idempotency key in; success, retryable failure, or permanent failure out.
- The worker retries with backoff, then marks failures visibly to users and the agent.
- Each service has its own folder under `integrations/`, with a fake client and tests.
- Rate limits are set per integration.
- Inbound webhooks verify the signature, store the raw event, and call a fixed operation with a service identity.
- Polling and syncing are batch jobs.
- The app DB is the source of truth for its own records.
- Inbound text is untrusted data.

*Rationale:* A single contract lets integrations be added without touching the core, while the DB and external systems stay consistent.

*Enforcement:* Check: domain code does not import integration clients. Check: every integration has a fake client and tests.

### F. User experience

#### D15. Generative UI
- UI uses A2UI directly, with no abstraction layer.
- UI is rendered from a fixed component catalog. The model never writes HTML or JavaScript.
- Screen-level templates are built at build time and filled with data by the agent.
- Dynamic composition from the catalog is allowed only for ad hoc exploration.
- Table and chart data is bound from tool results. The model never types numbers into the UI.
- Values inside sentences are also bound from tool results. The agent writes the words, not the values.
- The agent chooses the answer form by judgment, in all cases. Examples, which app designs may add to or change:
  - a sentence for a single fact,
  - a table for lists,
  - a chart for trends and comparisons,
  - a mini dashboard for an overview. This is a template with stat cards, a chart, and a short table.
- Operations may declare an optional `display` hint (section 7). It is recommended for business-critical reads, so numbers people compare always look the same.
- The user can ask for a different form, such as "show that as a table".
- Chart components use safe defaults, such as bars always starting at zero.
- Forms come from tool schemas (D2).
- Confirmation screens are system components built from the real payload. The agent cannot compose or alter them.
- Users can pin views. A pin stores a template and its tool calls, and re-runs without the model.
- New components and templates are added only through the build process.
- MCP Apps is an option for later, under the same rules: HTML written at build time, confirmations enforced by the adapter.
- Accepted risk: A2UI changes before version 1.0 require migrating templates directly.

*Rationale:* A standard spec and fixed templates give safe, consistent, cheap UI, with no extra layer to maintain. Form choice does not change data, so leaving it to judgment is low risk, and hints keep compared numbers consistent.

*Enforcement:* Check: agent UI output validates against the A2UI catalog. Check: chart, table, and text components take values only by binding. Eval: numbers shown match tool results. Eval: answer forms suit the result.

#### D16. Proactive briefing
- Each user visit to the app opens with a briefing.
- The opening screen always shows:
  - actions that failed (D11),
  - approvals and confirmations waiting on the user (D5, D9).
- Other parts are examples, which app designs may add to or replace:
  - what needs attention,
  - current status,
  - next steps,
  - what changed since the last visit,
  - upcoming items, such as meetings and tasks due,
  - items waiting on others,
  - pinned views (D15).
- Numbers come from a fixed read operation per role.
- The agent writes the summary and ranks suggested actions as "now" or "later".
- Each suggested action links to an operation. Nothing runs without the user, and D9 confirmation applies.
- A scheduled job prepares briefings ahead of time. If data has changed, the briefing refreshes in the background.

*Rationale:* Proactive help without slow load times or unverified numbers. Items the architecture depends on stay visible, and each app shapes the rest.

*Enforcement:* Verify: each role has a briefing operation. Check: every briefing template includes failed actions and pending approvals. Eval: briefing quality and ranking.

#### D19. Conversation history and memory
- Full transcripts are stored for audit, for a retention period set in the app design.
- Future sessions get a short memory per user: preferences and open threads. Not raw transcripts.
- Business facts come from the DB, not memory.
- The user can view and edit their memory.

*Rationale:* Context carries forward without replaying injected text.

*Enforcement:* Verify: no raw transcripts are loaded into new sessions.

### G. Observability and cost

#### D13. Audit and debugging
- A permanent business audit log in the DB records who changed what, when, before and after values, and whether it was confirmed. Audit triggers write it.
- Technical traces follow one trace ID through user message, tool call, operation, DB, outbox, and integration. They are kept for a set retention period.
- Access to both is role-restricted.

*Rationale:* Business history and technical debugging have different users and lifetimes.

*Enforcement:* Check: every business table has an audit trigger. Check: every log entry carries a trace ID, tenant, and area.

#### D14. Cost and latency
- Tools are grouped by business area and loaded on demand.
- Prompts and tool definitions are cached.
- Cost and latency are recorded per conversation, per tenant, and per area. Per area covers tool calls and the tokens of their results. Model reasoning is counted per conversation and per tenant only.
- Budgets apply per user, session, and tenant.

*Rationale:* Tool growth is the main long-term cost and accuracy risk, so it is limited by design.

*Enforcement:* Verify: tool loading by area. Runtime metrics: cost and latency per conversation, per tenant, and per area.

### H. Build and change

#### D8. Change requests from the runtime
- The runtime agent has one tool, `file_change_request`. Whenever a user conversationally asks for something no operation supports, the runtime agent says so and files the need, in its own words and without customer data, plus an example, to a table in the app DB.
- The runtime has no project access and no outbound path to any dev tool.
- The change request table can be exported as CSV or Markdown by an admin only. Export is a fixed operation. Each export is logged.
- Nothing else leaves the app. Any further export path is decided outside this architecture. Such a path must not carry customer data to external dev tools unless a human reviews the data before it leaves the app.

*Rationale:* Use drives the build, but the app's job ends at recording the need safely.

*Enforcement:* Check: the runtime has no credentials for dev tools. Check: export requires the admin role. Eval: the agent files a request when no operation fits, without customer data.

#### D17. Verifying generated code
- Mechanical architecture rules are automated checks that fail the build.
- An independent verification agent checks judgment rules against the app design written based on this document.
- Indexes for the coding and verification agents, such as a tool catalog and a table-ownership map, are generated from the source in CI. They are never edited by hand.

*Rationale:* Scripts are certain and cheap for mechanical rules, and an independent reviewer catches what scripts cannot.

#### D18. Runtime agent evals
- An eval suite tests runtime agent behavior before each release.
- It covers tool choice, form pass-through, briefings, and injection attempts.
- It uses scripted conversations with expected tool calls.

*Rationale:* Code checks do not cover what the agent does.

---

## 9. What each app design must specify

Besides carrying forward the rules above, an app design must define:

1. Purpose, users, and roles.
2. Stack choices (section 5).
3. Business areas, each with its owner and what it covers in business terms, such as deals or invoices. Tables are left to the coding agent.
4. Operations per area: name, purpose, `permission`, `confirm`, `admin`, and `outbound`. Input schemas and code are left to the coding agent.
5. Lifecycles for areas with states, including approval states.
6. Business-critical reads.
7. What data the agent may query ad hoc. Views and their columns are left to the coding agent.
8. Which data is sensitive. The columns that hold it are left to the coding agent.
9. Integrations, with actions and inbound events.
10. Batch jobs and their service identities.
11. UI templates, and any answer forms beyond the examples in D15.
12. Opening screen parts and briefing content per role.
13. Retention periods for transcripts, traces, and memory.
14. Budgets per user, session, and tenant.
15. Eval cases.
16. Any additional rules for the domain.
17. Whether ad hoc reads need a minimum group size, to stop totals over very small groups revealing individual values.

This is a minimum, not an outline. The app design may cover more, in any structure that suits the domain.

© 2026 Anoop P B · CC BY 4.0 · https://github.com/anoop-pb/agentype