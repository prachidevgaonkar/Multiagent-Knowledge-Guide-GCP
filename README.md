<img width="1280" height="640" alt="image" src="https://github.com/user-attachments/assets/63e8eb4f-e2b5-4ba6-a20a-652b92d78c96" />


# Multi-Agent Knowledge Guide System

> **Confidentiality note:** This is a generalized, independently written technical case study. Names, examples, schemas, timings, identifiers, and operating details are illustrative or intentionally abstracted. It contains no customer data, source code, credentials, internal URLs, unpublished product information, or employer-specific implementation details. Public Google Cloud product names are used only to explain a reference architecture.
>
> **What this is:** A proof-of-concept and technical learning journal. It documents the design approach, lessons, limitations, and possible improvements for a multi-agent knowledge system without representing any specific organization's production environment.

---

## Table of Contents

- [The Idea](#the-idea)
- [Who This Is For](#who-this-is-for)
- [What I Learned Building It](#what-I-learned-building-it)
- [Architecture — How I Got Here](#architecture--how-I-got-here)
- [Data Sources — The Three Pillars](#data-sources--the-three-pillars)
- [Agent Design — What I Built and Why](#agent-design--what-I-built-and-why)
- [Google Chat Integration — The Hard Part](#google-chat-integration--the-hard-part)
- [Active Issues — Still Learning](#active-issues--still-learning)
- [Security & IAM](#security--iam)
- [No-Code / Low-Code Context — Why This Matters Now](#no-code--low-code-context--why-this-matters-now)
- [Next Steps](#next-steps)
- [Reusing This Reference](#reusing-this-reference)

---

## The Idea

Every engineering program accumulates a mountain of knowledge — design documents, onboarding guides, runbooks, issue histories, source code. That knowledge lives in five different systems and is effectively inaccessible unless you know exactly where to look or who to ask.

The hypothesis was simple: **what if your team could ask questions in plain English, in Google Chat, and get answers drawn from all of those sources at once?**

This POC tests that hypothesis using Google Cloud's Vertex AI Agent Platform to orchestrate multiple specialised AI agents — one for documents, one for issue tracking data, one for source code — all reachable through a single conversational interface.

It is not finished. It works, imperfectly, and that's the point. This document is as much about what I learned designing and building it as it is a technical spec.

---

## Who This Is For

This POC was built to serve four groups — and the design decisions at every layer were shaped by their different needs:

| Audience | Primary use | Example question |
|---|---|---|
| **Technical team** (SWE / SRE) | Code understanding, issue triage, runbook lookup | *"What does the `DataPipeline` class do?"* |
| **Product team** (PM / TPM) | Feature status, sprint health, decision history | *"Summarise open P1s in the Payments component"* |
| **Business / leadership** | Program status, KPIs, risk summary | *"What are the current blockers for the release?"* |
| **New joiners** | Onboarding, team structure, tooling setup | *"How do I set up my dev environment?"* |

**Early learning:** I initially designed primarily for the technical team. Query patterns across audiences turned out to be completely different — PMs want summaries and status, not raw data. The routing logic in the Orchestrator Agent was redesigned to account for this.

---

## What I Learned Building It

This is the section I wish I had found before starting. A summary of the non-obvious lessons, with pointers to where each is documented in detail.

### On agent design

- **Start with one agent, not many.** I built three sub-agents on day one. I should have started with one and split only when a single agent demonstrably could not handle the context switching. Multi-agent overhead is real — routing errors compound.
- **The system prompt is the product.** The quality of agent responses is almost entirely determined by how precisely the system prompt describes the agent's role, data format, and constraints. Treat it like code — version it, review it, test it.
- **Temperature matters for structured data.** NL-to-SQL with temperature > 0 produces non-deterministic SQL. The same query can return different results across sessions. Setting `temperature=0` for the issue tracker sub-agent was the single highest-impact fix I made.

### On data integration

- **Batch issue data has a freshness problem.** When an enterprise issue tracker exports to BigQuery on a schedule, recent updates may not appear immediately. This limits time-sensitive status queries (see [Issue #3](#issue-3--data-and-code-freshness)).
- **Code-as-text works better than expected.** Storing source files as `.txt` in Cloud Storage felt like a compromise, but Vertex AI Search indexes them well and semantic search over code is surprisingly effective for "what does X do?" queries. The limitation shows up on "how does X interact with Y?" — cross-file reasoning is weak.
- **Document freshness is an invisible problem.** If a document is updated in Drive and not re-uploaded to the bucket, the agent answers from the old version with full confidence. I needed to make this visible.

### On the Google Chat integration

- **Synchronous chat-response windows are a major design constraint.** Agent calls that query multiple data sources may exceed the interaction platform's acknowledgement window. This requires durable asynchronous processing rather than long-running work in the webhook request. See [Issue #2](#issue-2--delayed-or-missing-chat-responses).
- **"Thinking..." is not enough UX.** During testing, a generic progress message made longer requests appear stalled. Briefly describing the current lookup improved perceived reliability.

---

## Architecture — How I Got Here

### Current architecture

The system has four layers. This is not the architecture I designed on day one — it evolved through three significant restructuring events documented below.

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1 — PRESENTATION                                     │
│  Google Chat  (natural language queries via chat space)     │
└───────────────────────┬─────────────────────────────────────┘
                        │ HTTP POST webhook
┌───────────────────────▼─────────────────────────────────────┐
│  LAYER 2 — MIDDLEWARE                                       │
│  Serverless webhook middleware                              │
│  validation · session management · durable task dispatch   │
└───────────────────────┬─────────────────────────────────────┘
                        │ authorized agent invocation
┌───────────────────────▼─────────────────────────────────────┐
│  LAYER 3 — AGENT ORCHESTRATION (Vertex AI Agent Platform)    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Orchestrator Agent                                  │   │
│  │  intent classification · routing · aggregation       │   │
│  └──────────┬──────────────────┬────────────┬───────────┘   │
│             │                  │            │               │
│    ┌────────▼──────┐  ┌────────▼──────┐  ┌─▼────────────┐ │
│    │  Doc agent    │  │  Issue agent  │  │  Code agent  │ │
│    │ (documents)   │  │ (issue data)   │  │ (.txt files) │ │
│    └───────────────┘  └───────────────┘  └──────────────┘ │
└──────────┬─────────────────┬──────────────────┬────────────┘
           │                 │                  │
  ┌────────▼──────┐  ┌───────▼────────┐  ┌─────▼──────────┐
  │  Cloud        │  │   BigQuery     │  │  Cloud Storage │
  │  Storage      │  │ (issue export)│  │  (code .txt    │
  │  (docs)       │  │               │  │   snapshots)   │
  └───────────────┘  └────────────────┘  └────────────────┘
```

### How the architecture evolved

**V1 — Single agent, all data stores**
Started with one Vertex AI agent connected to all three data stores. Context window quickly became a bottleneck — the agent confused document content with issue data on ambiguous queries. Response quality was inconsistent.

**V2 — Sub-agents, synchronous middleware**
Introduced the Orchestrator + three sub-agents pattern. Synchronous middleware then exposed the mismatch between chat acknowledgement windows and longer analytical queries.

**V3 — Async middleware (current)**
The preferred design acknowledges the chat request immediately and dispatches agent work through a managed task queue. This separates user interaction from longer-running processing (see [Issue #2](#issue-2--delayed-or-missing-chat-responses)).

### Technology stack

| Layer | Technology | Purpose |
|---|---|---|
| Agent platform | Vertex AI Agent Platform | Hosts orchestrator and sub-agents |
| Orchestration | Vertex AI Agent Engine | Routes queries, aggregates responses |
| Document data store | Vertex AI Search on Cloud Storage | Semantic search over program docs |
| Issue tracker data store | BigQuery + Vertex AI BigQuery connector | Structured query on issue exports |
| Code data store | Vertex AI Search on Cloud Storage (.txt) | Semantic search over code files |
| Middleware | Serverless functions + managed task queue | HTTP bridge and durable async orchestration |
| Chat interface | Google Chat App | User-facing conversational UI |
| Auth | Google Service Accounts + Workload Identity | Secure agent-to-data-store access |
| Observability | Cloud Logging, Cloud Trace, BigQuery audit logs | Tracing, latency, error tracking |

---

## Data Sources — The Three Pillars

### Pillar 1 (Doc agent) — Program documents (Cloud Storage)

Program documentation — design docs, onboarding guides, architecture decisions, runbooks, SLAs — is stored in a Cloud Storage bucket and indexed by Vertex AI Search.

**Bucket layout:**
```
gs://[PROJECT]-program-docs/
  ├── onboarding/
  │   ├── new-joiner-guide.pdf
  │   └── team-structure.docx
  ├── architecture/
  │   ├── system-design-v3.pdf
  │   └── api-contracts.md
  ├── product/
  │   └── prd-v2.pdf
  └── runbooks/
      ├── incident-response.md
      └── deployment-checklist.md
```

**Index configuration:**
- Unstructured document index with semantic chunking tuned through evaluation
- Managed embedding model selected according to platform support
- Scheduled refresh with event-driven updates where appropriate

**What works well:** Summarisation, policy lookup, onboarding Q&A, finding the right document when the user does not know its name.

**What does not work well:** Cross-document synthesis, version awareness (the agent does not know if it is reading an outdated version), very large PDFs where relevant content is buried in a single deeply nested paragraph.

---

### Pillar 2 (Issue agent) — Issue tracker data (BigQuery)

#### What is the issue data source?

This reference architecture assumes a generic enterprise issue and feature lifecycle management system. The source platform may be Jira, another commercial tracker, or a proprietary system.

A typical issue tracker:
- Tracks issues through defined lifecycle states: Open → Assigned → Fixed → Verified → Closed
- Classifies work by priority (P0 critical → P4 low) and severity (S0 emergency → S3 minor)
- Organises issues by component and product area
- Assigns ownership to engineers and teams
- Links issues to sprints, milestones, and releases
- Supports structured querying, reporting, and dashboarding

The pattern is platform-neutral: export authorized issue metadata to BigQuery through a supported API or connector, then provide the read-only agent with a curated analytical schema.

| Capability | Typical tracker | Illustrative POC |
|---|---|---|
| Real-time API | REST + webhooks | Batch export to BigQuery |
| Native query language | Platform-specific | Natural language to SQL |
| Agent integration | Vendor-dependent | Custom read-only sub-agent |
| Data freshness | Near real-time | Periodic, non-real-time refresh |
| AI query layer | Vendor-dependent | Custom orchestration layer |

#### Integration approach

For the illustrative POC, authorized issue data is exported to BigQuery periodically and queried there. The Issue agent uses NL-to-SQL to translate natural-language questions into read-only analytical queries.

**BigQuery schema — issues table:**

| Column | Type | Description |
|---|---|---|
| `issue_id` | STRING | Unique issue identifier |
| `title` | STRING | Issue title / summary |
| `status` | STRING | OPEN, ASSIGNED, FIXED, VERIFIED, CLOSED |
| `priority` | STRING | P0 (critical) → P4 (low) |
| `severity` | STRING | S0 (emergency) → S3 (minor) |
| `component` | STRING | Product component or module |
| `assignee_group` | STRING | Owning team or anonymized assignee key |
| `reporter_group` | STRING | Reporting team or anonymized reporter key |
| `sprint_id` | STRING | Associated sprint identifier |
| `created_at` | TIMESTAMP | Issue creation time |
| `updated_at` | TIMESTAMP | Last update time |
| `resolved_at` | TIMESTAMP | Resolution time (null if open) |

**Example NL-to-SQL — what the Issue agent does:**

User query:
```
"How many P0 bugs are open in the Authentication component?"
```

Generated SQL (at `temperature=0`):
```sql
SELECT COUNT(*) AS open_p0_count
FROM `project.dataset.issues`
WHERE status = 'OPEN'
  AND priority = 'P0'
  AND component = 'Authentication';
```

**Pipeline configuration:**
- Export frequency: periodic and environment-dependent (a known limitation)
- Mode: `WRITE_TRUNCATE` for small tables; incremental `MERGE` for large tables
- Deduplication: `issue_id + updated_at`
- Partitioning: `issues` table by `created_at` DATE
- Clustering: `(priority, status, component)` — matches common query filter patterns

---

### Pillar 3 (Code agent) — Source code (Cloud Storage / .txt files)

Source files are exported from the code repository as `.txt` files via CI/CD and indexed by Vertex AI Search.

**Why .txt and not direct repo integration?**

Direct repository integration was the original plan. I moved away from it for the POC because: Vertex AI Search indexes unstructured text rather than code natively; real-time repo webhooks add significant middleware complexity; and `.txt` export via Cloud Build gives me control over exactly which files are indexed (no generated files, no binaries). This is a deliberate short-term compromise — see [Next Steps](#next-steps) for where this goes.

**CI/CD export step (Cloud Build):**
```yaml
- name: 'gcr.io/cloud-builders/gsutil'
  entrypoint: 'bash'
  args:
    - '-c'
    - |
      find ./src -name '*.py' -o -name '*.go' -o -name '*.java' | while read f; do
        dest="gs://${PROJECT}-code-snapshots/${f}.txt"
        gsutil cp "$f" "$dest"
      done
```

**What works well:** "What does function X do?", "Explain this module", "What parameters does API Y accept?"

**What does not work well:** Cross-file reasoning, recently merged code that has not yet been re-indexed, and generated or minified files that slip past the exclusion filter.

---

## Agent Design — What I Built and Why

### The Orchestrator Agent

The Orchestrator is the entry point for all queries. It classifies intent and routes to the appropriate sub-agent. For cross-domain queries it calls multiple sub-agents in parallel and aggregates the responses.

**Routing table:**

| User intent | Target | Trigger signals |
|---|---|---|
| Document / policy | Doc agent | onboarding, architecture, SLA, PRD, design doc, spec, runbook |
| Issue / bug / feature | Issue agent | bug, issue, priority, sprint, blocker, ticket, defect, feature request |
| Code / implementation | Code agent | function, class, module, file, implementation, source, code, API |
| Cross-domain | All agents (parallel) | status, summary, program health, tell me about |

**Learning from routing failures:**

Early versions frequently mis-routed "how does authentication work?" to the Code agent because of the word "how does", when the user actually wanted the architecture document. Adding explicit disambiguation examples to the system prompt significantly improved routing accuracy. The examples in the system prompt matter as much as the routing rules.

### The Issue Agent

**Design constraints:**
- Must generate only `SELECT` statements — no DML under any circumstances
- Must include the generated SQL in its response for transparency
- Must handle empty result sets gracefully with helpful guidance on query refinement
- Must limit result sets to 50 rows by default with pagination support

**System prompt structure (simplified):**
```
You are an issue tracker specialist with read-only access to an approved BigQuery
dataset containing authorized issue metadata.

When a user asks about bugs, issues, feature requests, sprint health, or component
status, generate a safe SELECT query using the schema below.

Always respond with:
1. A plain-English summary of what you found
2. The SQL query you used (in a code block)
3. Key numbers or counts highlighted

Schema: [full BigQuery schema appended — kept in sync manually, automation planned]

Never generate INSERT, UPDATE, DELETE, or DROP statements.
Temperature is 0 — do not attempt to be creative with SQL generation.
```

---

## Google Chat Integration — The Hard Part

### The response-window problem

Chat platforms expect webhooks to acknowledge events quickly, while multi-source agent calls can take longer. The shareable design therefore separates acknowledgement from processing.

### Recommended interaction flow

1. Google Chat sends an authenticated event to serverless middleware.
2. Middleware validates the request and immediately acknowledges receipt.
3. A managed task queue receives a minimal, authorized work item.
4. A worker invokes the orchestrator and permitted data tools.
5. The worker posts the final response through the Chat API.
6. Structured logs capture correlation IDs, timing bands, and sanitized errors.

The important lesson is architectural: durable queues are more reliable than unmanaged background threads in an ephemeral serverless instance. Deployment identifiers, service-account addresses, endpoint URLs, queue names, and timeout values are intentionally omitted.

---

## Active Issues — Still Learning

These are not bugs to fix before launch. They are the active learning surface of the POC — open questions about how to build AI agent systems well. Each one is documented with what was observed, what was learned, what was changed, and what remains to be done.

---

### Issue #1 — Inconsistent responses on issue data

> **Status: Active** — The same question about issue data can produce different answers in different sessions, even when the underlying data has not changed.

**Observed:**
- Equivalent questions can return different counts even when the source snapshot is unchanged
- Some valid queries return empty or incomplete results
- Component name misspellings cause total query failure with no helpful error message
- Multi-table JOIN queries are more prone to inconsistency than single-table queries

**Learned:**
The root cause is NL-to-SQL non-determinism. At `temperature > 0`, the LLM generating SQL from natural language produces structurally different queries for the same input. A query like "open P0s in Authentication" might generate `component = 'Authentication'` in one session and `component LIKE '%auth%'` in another — returning different result sets.

A secondary problem: the BigQuery schema is embedded in the agent system prompt as static text. When the schema evolves, the prompt is not updated automatically, and the LLM generates SQL referencing columns that no longer exist.

**Changed:**
- Set `temperature=0` for the Issue agent — the single highest-impact change
- Added explicit schema documentation with column descriptions and valid enum values to the system prompt
- Added curated BigQuery views for common query patterns and exposed them as named tools

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| Completed | Deterministic generation settings for the Issue agent | Validated in POC |
| Completed | Curated views for common queries | Validated in POC |
| Next | SQL validation layer in middleware | Planned |
| Next | Automated schema synchronization | Planned |
| Next | Short-lived query-result cache | Planned |
| Production consideration | Constrained query builder with parameterized templates | Future |

---

### Issue #2 — Delayed or missing chat responses

> **Status: Active** — POC testing found that some longer analytical requests may not produce a final chat response.

**Observed:**
- User sends message, sees "Thinking...", waits, sees nothing
- Middleware acknowledges the message, but downstream processing may not complete
- Complex analytical queries are more likely to be affected
- Cold starts and transient delivery failures can increase risk

**Learned:**
The interaction platform's acknowledgement window can be shorter than a multi-source agent request. A background-thread experiment reduced blocking but introduced a new failure mode when an ephemeral function instance ended before work completed.

A second failure mode: the Chat API POST delivering the real response can fail due to an expired OAuth token or rate limit — and these failures were not being logged. Users saw "Thinking..." with no follow-up because the response was computed but never delivered.

**Changed:**
- Error handling ensures the user receives a sanitized failure response
- Worker execution settings adjusted within supported platform limits
- Structured logging added for all Chat API post failures

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| Completed | Immediate acknowledgement before agent work | Validated in POC |
| Completed | User-visible failure callback | Validated in POC |
| Completed | Structured logging for delivery failures | Validated in POC |
| Next | Managed task queue replacing background threads | Planned |
| Next | Progress update for longer requests | Planned |
| Production consideration | Fully decoupled event-driven architecture | Future |

---

### Issue #3 — Data and code freshness

> **Status: Known limitation** — Periodic exports and indexing introduce a delay between source updates and searchable content.

**Observed:**
- Recently created or updated issues may not appear immediately
- Recently merged code may not be reflected until export and indexing complete
- The agent may report a status from an older snapshot
- Time-sensitive metrics can drift between refresh cycles

**Learned:**
This is a fundamental constraint of batch export and asynchronous indexing. Even when snapshots are generated promptly, the search index may take additional time to make updates queryable.

Truly real-time code and issue search is therefore outside the scope of this POC architecture.

**Planned:**

- Process only records changed since the previous export.
- Trigger targeted re-indexing when authorized source objects change.
- Surface source timestamps and freshness indicators in agent responses.

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| Next | Incremental BigQuery updates | Planned |
| Next | Event-driven targeted re-indexing | Planned |
| Production consideration | Authorized change feed into the analytical store | Future |
| Production consideration | Supported direct issue-tracker connector | Future |
| Production consideration | Supported source-repository connector | Future |

---

## Security & IAM

### Service identities

Separate service identities should be used for agent execution, webhook middleware, data export, and code snapshot publishing. Each identity should receive only the minimum permissions needed for its function. Exact names, role bindings, project boundaries, and deployment policies are environment-specific and intentionally omitted.

### Recommended controls

- Uniform bucket-level access on all Cloud Storage buckets (no legacy ACLs)
- BigQuery policy controls on fields that may contain personal or organizational identifiers
- Signed-request validation on the webhook
- Service account impersonation throughout — no long-lived credentials stored anywhere
- Network and service perimeters where required by the production security model

### Gaps to assess before production

- Centralized audit review and alerting for data-store access
- Document-level authorization aligned with permissions in each source system

---

## No-Code / Low-Code Context — Why This Matters Now

This POC sits at an interesting intersection: it was built by engineers, using tools that are themselves products of the no-code / low-code revolution. Understanding where that ecosystem is heading contextualises both what this system can do today and where the remaining rough edges will be smoothed out by the platforms themselves.

### The three waves

**Wave 1 — Automation & publishing (2012–2018)**
Zapier, IFTTT, Wix, Airtable. Simple connectors and content tools. The ceiling was low — any non-trivial logic required engineering handoff.

**Wave 2 — Visual app builders (2018–2022)**
Visual development platforms expanded full-stack application creation and made enterprise data more accessible to non-engineering teams.

**Wave 3 — AI-native builders (2022–present)**

| Category | Platforms | What changed |
|---|---|---|
| AI agent builders | Vertex AI Agent Platform, Microsoft Copilot Studio | Configure multi-agent pipelines in natural language |
| App generators | v0, Lovable, bolt.new | Describe the interface → get working code |
| Embedded AI copilots | Power Platform, Google AppSheet | AI generates workflow logic inside the builder canvas |
| Enterprise AI layers | Managed enterprise copilots | Natural-language access over governed business data |

### Where this POC fits

This system is a Wave 3 product built with Wave 3 tools. The agent configuration, data store connections, and routing logic were set up through Vertex AI Agent Platform's console — no ML code written. What required an ML engineering team in 2021 is now configuration work.

The remaining friction — NL-to-SQL inconsistency, indexing latency, and chat response constraints — reflects the maturity of current managed platforms. As governed natural-language query layers improve, some custom intermediary components may become unnecessary.

**The design implication:** The architectural decisions in this system that feel like engineering compromises (batch exports, `.txt` code snapshots, threading-based async) are mostly temporary workarounds for platform gaps that are actively closing. Build for the platform to meet you — do not over-engineer permanent solutions to what the managed service will solve in 18 months.

### Current no-code / low-code limitations

- Real-time, latency-sensitive systems requiring sub-second guarantees
- Complex stateful distributed architectures with custom consistency models
- Fine-grained performance optimisation at scale
- Deep security review and compliance-critical infrastructure

Everything else is converging.

---

## Next Steps

This POC validated the core hypothesis: a multi-agent system can meaningfully answer cross-domain questions about a program using documents, issue tracker data, and source code — all through a conversational Google Chat interface. That part works.

What follows are the natural directions to explore if this POC is taken further. These are not a project plan — they are open questions and known gaps that the POC surfaced, roughly grouped by what they address.

### Fix the known issues first

Before building anything new, the three active issues documented above should be resolved. They are the most user-visible problems and have clear solutions that do not require new architecture:

- Use a managed task queue for durable asynchronous processing
- Add a SQL validation layer and schema synchronization
- Switch to incremental BigQuery MERGE and add an Eventarc re-index trigger (reduces data staleness)

None of these require changing the agent design or the data store structure — they are improvements to the plumbing.

### Data freshness and real-time sync

The batch export architecture is the biggest gap between "POC" and "production-useful". The path forward:

- **For issue data:** Where an authorized change feed is available, process incremental updates instead of repeatedly exporting the full dataset.
- **For code:** Replace the `.txt` snapshot approach with a direct connector to Cloud Source Repositories or a GitHub/GitLab integration. Vertex AI Search has a native repository connector that eliminates the CI/CD export step entirely.
- **For documents:** Add document version metadata so the agent can surface "this document was last updated 6 months ago" alongside its answers, making staleness visible rather than invisible.

### Reliability of the Chat integration

The current async threading approach is fragile. The recommended next step is Cloud Tasks (code already in this document). Beyond that, a Pub/Sub-based architecture decouples every component:

- Chat event → Pub/Sub topic → subscriber calls agent → response posted back to Chat
- Each component can fail and retry independently, with no coupling to Cloud Function instance lifecycle

### Agent quality

The POC demonstrated that routing accuracy and response quality are primarily determined by the system prompt, not the model. The next investments here:

- **Eval harness:** A test set of representative questions with expected answers, run against the agent after every system prompt change. Right now, changes are validated manually.
- **User feedback loop:** A thumbs up/down reaction in Chat that logs to BigQuery, creating a training signal for which responses are actually useful.
- **Multi-turn memory:** The current session management is per-user-per-space but the agent does not use prior turns well. Improving context carry-over would make follow-up questions work naturally.

### Expanding the data surface

The three current pillars (documents, issues, code) cover the most common question types. Natural additions:

- **Meeting notes and decisions:** Recorded meeting transcripts or decision logs are a common source of "why did I do it this way?" questions that documents don't capture well.
- **Monitoring and SLOs:** Connecting to Cloud Monitoring or a metrics store would let the agent answer "is the service healthy right now?" alongside "what is the SLA supposed to be?".
- **On-call history:** Past incident timelines and post-mortems are highly valuable for new team members and are often trapped in docs no one reads.

### If this moves toward production

A few things that are deliberately out of scope for the POC but would matter at scale:

- **Per-document access control:** Right now, every user in the Chat space can query every document in the bucket. A production system needs to respect the same access controls as the source documents.
- **Audit logging dashboard:** Cloud Logging already captures all data access — it just needs a dashboard to make it reviewable.
- **Cost model:** Vertex AI Search, BigQuery, and Cloud Functions all have usage-based costs. At low POC volume these are negligible; at team-wide adoption they need to be monitored.
- **Multi-space / multi-program support:** The current design serves one program's data. A shared infrastructure model would let multiple programs each bring their own data stores and share the orchestration layer.

---

## Reusing This Reference

Teams adapting this pattern should use their own approved repositories, naming conventions, access controls, data-classification rules, and review process. Before publishing implementation details, confirm that examples contain no customer information, employee identifiers, internal URLs, credentials, proprietary schemas, unpublished metrics, or source code.

---

*NDA-safe external edition. Reviewed and generalized: June 2026.*
