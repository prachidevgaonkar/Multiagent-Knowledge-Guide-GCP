<img width="1280" height="640" alt="image" src="https://github.com/user-attachments/assets/63e8eb4f-e2b5-4ba6-a20a-652b92d78c96" />


# Multi-Agent Knowledge Guide System
### A Learning-While-Building POC on Google Cloud Vertex AI Agent Platform

> **What this is:** A working proof-of-concept and living technical journal. Every section documents not just *what* was built, but *what was learned*, *what broke*, and *what was changed as a result*. The goal is to make this a reference for anyone building similar AI-agent systems on Google Cloud — without the benefit of a clean retrospective.

---

## Table of Contents

- [The Idea](#the-idea)
- [Who This Is For](#who-this-is-for)
- [What We Learned Building It](#what-we-learned-building-it)
- [Architecture — How We Got Here](#architecture--how-we-got-here)
- [Data Sources — The Three Pillars](#data-sources--the-three-pillars)
- [Agent Design — What We Built and Why](#agent-design--what-we-built-and-why)
- [Google Chat Integration — The Hard Part](#google-chat-integration--the-hard-part)
- [Active Issues — Still Learning](#active-issues--still-learning)
- [Security & IAM](#security--iam)
- [No-Code / Low-Code Context — Why This Matters Now](#no-code--low-code-context--why-this-matters-now)
- [Next Steps](#next-steps)
- [How to Contribute to This POC](#how-to-contribute-to-this-poc)

---

## The Idea

Every engineering program accumulates a mountain of knowledge — design documents, onboarding guides, runbooks, issue histories, source code. That knowledge lives in five different systems and is effectively inaccessible unless you know exactly where to look or who to ask.

The hypothesis was simple: **what if your team could ask questions in plain English, in Google Chat, and get answers drawn from all of those sources at once?**

This POC tests that hypothesis using Google Cloud's Vertex AI Agent Platform to orchestrate multiple specialised AI agents — one for documents, one for issue tracking data, one for source code — all reachable through a single conversational interface.

It is not finished. It works, imperfectly, and that's the point. This document is as much about what we learned designing and building it as it is a technical spec.

---

## Who This Is For

This POC was built to serve four groups — and the design decisions at every layer were shaped by their different needs:

| Audience | Primary use | Example question |
|---|---|---|
| **Technical team** (SWE / SRE) | Code understanding, issue triage, runbook lookup | *"What does the `DataPipeline` class do?"* |
| **Product team** (PM / TPM) | Feature status, sprint health, decision history | *"Summarise open P1s in the Payments component"* |
| **Business / leadership** | Program status, KPIs, risk summary | *"What are the current blockers for Project Titan?"* |
| **New joiners** | Onboarding, team structure, tooling setup | *"How do I set up my dev environment?"* |

**Early learning:** We initially designed primarily for the technical team. The first demo to a PM revealed that the query patterns were completely different — PMs want summaries and status, not raw data. The routing logic in the Orchestrator Agent was redesigned after that session.

---

## What We Learned Building It

This is the section we wish we had found before starting. A summary of the non-obvious lessons, with pointers to where each is documented in detail.

### On agent design

- **Start with one agent, not many.** We built three sub-agents on day one. We should have started with one and split only when a single agent demonstrably could not handle the context switching. Multi-agent overhead is real — routing errors compound.
- **The system prompt is the product.** The quality of agent responses is almost entirely determined by how precisely the system prompt describes the agent's role, data format, and constraints. Treat it like code — version it, review it, test it.
- **Temperature matters for structured data.** NL-to-SQL with temperature > 0 produces non-deterministic SQL. The same query can return different results across sessions. Setting `temperature=0` for the issue tracker sub-agent was the single highest-impact fix we made.

### On data integration

- **Buganizer via BigQuery has a freshness problem.** Buganizer — Google's internal issue and feature tracking system, analogous to Jira for bug/feature lifecycle management — exports to BigQuery on a batch schedule. Users asking about recently filed issues get stale answers. This is the most-complained-about limitation and is actively being addressed (see [Issue #3](#issue-3--no-real-time-sync-with-code-repo-and-buganizer)).
- **Code-as-text works better than expected.** Storing source files as `.txt` in Cloud Storage felt like a compromise, but Vertex AI Search indexes them well and semantic search over code is surprisingly effective for "what does X do?" queries. The limitation shows up on "how does X interact with Y?" — cross-file reasoning is weak.
- **Document freshness is an invisible problem.** If a document is updated in Drive and not re-uploaded to the bucket, the agent answers from the old version with full confidence. We needed to make this visible.

### On the Google Chat integration

- **The 30-second timeout is the hardest constraint in the system.** Google Chat requires your webhook to respond within 30 seconds or it drops the connection silently — the user sees nothing. Vertex AI agent calls, especially ones that hit BigQuery, routinely take longer. This caused the most user-visible failures and required an architectural change. See [Issue #2](#issue-2--no-response-on-google-chat-timeout).
- **"Thinking..." is not enough UX.** Users who sent a complex query and saw only "Thinking..." for 20 seconds assumed the bot was broken. Adding a brief description of what the agent is looking up dramatically improved perceived reliability even before the response arrived.

---

## Architecture — How We Got Here

### Current architecture

The system has four layers. This is not the architecture we designed on day one — it evolved through three significant restructuring events documented below.

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1 — PRESENTATION                                     │
│  Google Chat  (natural language queries via chat space)     │
└───────────────────────┬─────────────────────────────────────┘
                        │ HTTP POST webhook
┌───────────────────────▼─────────────────────────────────────┐
│  LAYER 2 — MIDDLEWARE                                       │
│  Cloud Function (2nd gen, Python 3.12)                      │
│  JWT validation · session management · async fire-and-forget│
└───────────────────────┬─────────────────────────────────────┘
                        │ detect_intent API
┌───────────────────────▼─────────────────────────────────────┐
│  LAYER 3 — AGENT ORCHESTRATION (Vertex AI Agent Designer)    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Orchestrator Agent                                  │   │
│  │  intent classification · routing · aggregation       │   │
│  └──────────┬──────────────────┬────────────┬───────────┘   │
│             │                  │            │               │
│    ┌────────▼──────┐  ┌────────▼──────┐  ┌─▼────────────┐ │
│    │  Doc agent    │  │  Issue agent  │  │  Code agent  │ │
│    │ (documents)   │  │  (Buganizer)  │  │ (.txt files) │ │
│    └───────────────┘  └───────────────┘  └──────────────┘ │
└──────────┬─────────────────┬──────────────────┬────────────┘
           │                 │                  │
  ┌────────▼──────┐  ┌───────▼────────┐  ┌─────▼──────────┐
  │  Cloud        │  │   BigQuery     │  │  Cloud Storage │
  │  Storage      │  │  (Buganizer    │  │  (code .txt    │
  │  (docs)       │  │   export)      │  │   snapshots)   │
  └───────────────┘  └────────────────┘  └────────────────┘
```

### How the architecture evolved

**V1 — Single agent, all data stores**
Started with one Vertex AI agent connected to all three data stores. Context window quickly became a bottleneck — the agent confused document content with issue data on ambiguous queries. Response quality was inconsistent.

**V2 — Sub-agents, synchronous middleware**
Introduced the Orchestrator + three sub-agents pattern. Cloud Function called the agent synchronously. This immediately exposed the 30-second timeout problem — any BigQuery query that took longer produced a silent failure in Chat.

**V3 — Async middleware (current)**
Cloud Function now returns HTTP 200 immediately and calls the agent asynchronously in a background thread. This resolved the timeout failures for most queries. The remaining failure mode is thread reliability on cold-start instances (see [Issue #2](#issue-2--no-response-on-google-chat-timeout)).

### Technology stack

| Layer | Technology | Purpose |
|---|---|---|
| Agent platform | Vertex AI Agent Designer | Hosts orchestrator and sub-agents |
| Orchestration | Vertex AI Agent Engine | Routes queries, aggregates responses |
| Document data store | Vertex AI Search on Cloud Storage | Semantic search over program docs |
| Issue tracker data store | BigQuery + Vertex AI BigQuery connector | Structured query on Buganizer export |
| Code data store | Vertex AI Search on Cloud Storage (.txt) | Semantic search over code files |
| Middleware | Cloud Functions (2nd gen, Python 3.12) | HTTP webhook bridge, async orchestration |
| Chat interface | Google Chat App | User-facing conversational UI |
| Auth | Google Service Accounts + Workload Identity | Secure agent-to-data-store access |
| Observability | Cloud Logging, Cloud Trace, BigQuery audit logs | Tracing, latency, error tracking |

---

## Data Sources — The Three Pillars

### Pillar 1 — Program documents (Cloud Storage)

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
- Unstructured document index with semantic chunking (500 tokens, 50-token overlap)
- Embeddings: `textembedding-gecko@003`
- Refresh: Cloud Scheduler every 6 hours + Eventarc trigger on new uploads

**What works well:** Summarisation, policy lookup, onboarding Q&A, finding the right document when the user does not know its name.

**What does not work well:** Cross-document synthesis, version awareness (the agent does not know if it is reading an outdated version), very large PDFs where relevant content is buried in a single deeply nested paragraph.

---

### Pillar 2 — Issue tracker data (BigQuery / Buganizer)

#### What is Buganizer?

**Buganizer is Google's internal issue and feature tracking system** — the equivalent of Jira for engineering bug and feature lifecycle management, or ServiceNow for IT service management workflows.

Like those enterprise tools, Buganizer:
- Tracks issues through defined lifecycle states: Open → Assigned → Fixed → Verified → Closed
- Classifies work by priority (P0 critical → P4 low) and severity (S0 emergency → S3 minor)
- Organises issues by component and product area
- Assigns ownership to engineers and teams
- Links issues to sprints, milestones, and releases
- Supports structured querying, reporting, and dashboarding

For teams outside Google, the closest public-facing equivalent is the **Google Issue Tracker** (issuetracker.google.com), which runs on the same underlying system. Any reference to "Buganizer data" in this document means structured issue metadata from this system, exported to BigQuery for agent access.

Teams using Jira or ServiceNow can replicate this exact integration pattern: export structured issue data to BigQuery via the Jira REST API or ServiceNow table API, then point the Vertex AI sub-agent at the resulting BigQuery dataset. The NL-to-SQL approach and schema design translate directly.

| Capability | Jira (typical) | ServiceNow (typical) | Buganizer (this POC) |
|---|---|---|---|
| Real-time API | REST + webhooks | REST + webhooks | Batch export to BigQuery |
| Query language | JQL | GlideQuery | NL-to-SQL via LLM |
| Agent integration | Atlassian Intelligence | Now Intelligence | Custom Vertex AI sub-agent |
| Data freshness | Near real-time | Near real-time | 4-hour lag (current) |
| AI query layer | Atlassian AI (managed) | ServiceNow AI Search | Custom (this system) |

#### Integration approach

Because Buganizer does not currently expose a real-time streaming API suitable for direct agent query in this context, the data is exported to BigQuery on a scheduled basis and queried from there. The Issue agent uses NL-to-SQL to translate natural language questions into BigQuery queries.

**BigQuery schema — issues table:**

| Column | Type | Description |
|---|---|---|
| `issue_id` | STRING | Unique Buganizer issue identifier |
| `title` | STRING | Issue title / summary |
| `status` | STRING | OPEN, ASSIGNED, FIXED, VERIFIED, CLOSED |
| `priority` | STRING | P0 (critical) → P4 (low) |
| `severity` | STRING | S0 (emergency) → S3 (minor) |
| `component` | STRING | Product component or module |
| `assignee` | STRING | Assigned engineer (email) |
| `reporter` | STRING | Issue reporter (email) |
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
- Export frequency: every 4 hours (a known limitation — see [Next Steps](#next-steps) for improvement directions)
- Mode: `WRITE_TRUNCATE` for small tables; incremental `MERGE` for large tables
- Deduplication: `issue_id + updated_at`
- Partitioning: `issues` table by `created_at` DATE
- Clustering: `(priority, status, component)` — matches common query filter patterns

---

### Pillar 3 — Source code (Cloud Storage / .txt files)

Source files are exported from the code repository as `.txt` files via CI/CD and indexed by Vertex AI Search.

**Why .txt and not direct repo integration?**

Direct repository integration was the original plan. We moved away from it for the POC because: Vertex AI Search indexes unstructured text rather than code natively; real-time repo webhooks add significant middleware complexity; and `.txt` export via Cloud Build gives us control over exactly which files are indexed (no generated files, no binaries). This is a deliberate short-term compromise — see [Next Steps](#next-steps) for where this goes.

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

**What does not work well:** Cross-file reasoning, recently merged code (up to 24-hour lag), generated or minified files that slip past the exclusion filter.

---

## Agent Design — What We Built and Why

### The Orchestrator Agent

The Orchestrator is the entry point for all queries. It classifies intent and routes to the appropriate sub-agent. For cross-domain queries it calls multiple sub-agents in parallel and aggregates the responses.

**Routing table:**

| User intent | Target | Trigger signals |
|---|---|---|
| Document / policy | Doc agent | onboarding, architecture, SLA, PRD, design doc, spec, runbook |
| Issue / bug / feature | Issue agent | bug, issue, P0, P1, sprint, blockers, Buganizer, ticket, defect, feature request |
| Code / implementation | Code agent | function, class, module, file, implementation, source, code, API |
| Cross-domain | All agents (parallel) | status, summary, program health, tell me about |

**Learning from routing failures:**

Early versions frequently mis-routed "how does authentication work?" to the Code agent because of the word "how does", when the user actually wanted the architecture document. Adding explicit disambiguation examples to the system prompt significantly improved routing accuracy. The examples in the system prompt matter as much as the routing rules.

### The Issue Agent (Buganizer)

**Design constraints:**
- Must generate only `SELECT` statements — no DML under any circumstances
- Must include the generated SQL in its response for transparency
- Must handle empty result sets gracefully with helpful guidance on query refinement
- Must limit result sets to 50 rows by default with pagination support

**System prompt structure (simplified):**
```
You are an issue tracker specialist with read-only access to a BigQuery dataset
containing exported Buganizer issue data. Buganizer is Google's internal issue
tracking system equivalent to Jira or ServiceNow.

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

### The 30-second problem

Integrating with Google Chat looked straightforward: expose a webhook, receive a message, call the agent, return the response. In practice, Google Chat's 30-second synchronous response requirement creates a fundamental tension with AI agent response times. This section documents what we tried and what we learned.

### How the integration works (current state)

```
1.  User sends message in Google Chat space
2.  Chat delivers HTTP POST to Cloud Function webhook
3.  Cloud Function validates Google-signed JWT
4.  Cloud Function returns HTTP 200 + "Thinking..." IMMEDIATELY
    └── Critical: must happen within 30 seconds or Chat drops the connection silently
5.  Cloud Function spawns background thread to call Vertex AI agent
6.  Agent Engine routes to sub-agent(s), queries data stores
7.  Agent returns generated response
8.  Cloud Function calls Chat REST API to post the real response
```

### Async pattern (current implementation)

```python
@functions_framework.http
def handle_chat_event(request):
    payload = request.get_json()
    verify_google_chat_jwt(request)

    # Return 200 FIRST — do not wait for the agent
    send_typing_indicator(payload['space']['name'])

    # Fire background thread for agent work
    thread = threading.Thread(
        target=call_agent_and_respond,
        args=(payload,)
    )
    thread.start()
    return 'OK', 200


def call_agent_and_respond(payload):
    try:
        session_id = f"{payload['space']['name']}_{payload['user']['name']}"
        response = call_vertex_agent(payload['message']['text'], session_id)
        post_to_chat(payload['space']['name'], response)
    except Exception as e:
        # Always post something — silent failures are worse than error messages
        post_to_chat(payload['space']['name'],
                     f"Something went wrong: {str(e)}. Please try again.")
```

### Next: Cloud Tasks pattern (recommended improvement)

The background thread approach has a reliability problem: if the Cloud Function instance is recycled mid-execution, the thread dies silently. Cloud Tasks eliminates this:

```python
def handle_chat_event(request):
    payload = request.get_json()
    verify_google_chat_jwt(request)

    client = tasks_v2.CloudTasksClient()
    task = {
        'http_request': {
            'http_method': tasks_v2.HttpMethod.POST,
            'url': AGENT_WORKER_URL,
            'body': json.dumps(payload).encode(),
        }
    }
    client.create_task(parent=QUEUE_PATH, task=task)

    send_thinking_message(payload)   # immediate ack to Chat
    return 'Queued', 200
```

### Environment variables

| Variable | Description | Example |
|---|---|---|
| `VERTEX_PROJECT_ID` | GCP project hosting the agent | `my-project-123` |
| `VERTEX_LOCATION` | Agent region | `us-central1` |
| `AGENT_ID` | Vertex AI Agent Designer agent ID | `abc123def456` |
| `CHAT_SA_EMAIL` | Service account for Chat API calls | `chat-bot@project.iam.gserviceaccount.com` |
| `MAX_RESPONSE_WAIT_SECS` | Timeout for agent response | `45` |

---

## Active Issues — Still Learning

These are not bugs to fix before launch. They are the active learning surface of the POC — open questions about how to build AI agent systems well. Each one is documented with what was observed, what was learned, what was changed, and what remains to be done.

---

### Issue #1 — Inconsistent responses on Buganizer data

> **Status: Active** — The same question about issue data can produce different answers in different sessions, even when the underlying data has not changed.

**Observed:**
- P0 bug count returns 12 in one session, 11 in another with no data change
- Occasionally returns empty result for queries that should match hundreds of rows
- Component name misspellings cause total query failure with no helpful error message
- Multi-table JOIN queries are more prone to inconsistency than single-table queries

**Learned:**
The root cause is NL-to-SQL non-determinism. At `temperature > 0`, the LLM generating SQL from natural language produces structurally different queries for the same input. A query like "open P0s in Authentication" might generate `component = 'Authentication'` in one session and `component LIKE '%auth%'` in another — returning different result sets.

A secondary problem: the BigQuery schema is embedded in the agent system prompt as static text. When the schema evolves, the prompt is not updated automatically, and the LLM generates SQL referencing columns that no longer exist.

**Changed:**
- Set `temperature=0` for the Issue agent — the single highest-impact change
- Added explicit schema documentation with column descriptions and valid enum values to the system prompt
- Pre-built BigQuery Views for the 8 most common query patterns and added these as named tools in the agent

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| ✅ Done | `temperature=0` for Issue agent | Deployed |
| ✅ Done | Pre-built Views for common queries | Deployed |
| 🔧 Next | SQL validation layer in Cloud Function | Open |
| 🔧 Next | Automated schema sync via Data Catalog events | Open |
| 🔧 Next | Redis query result cache (15-min TTL) | Open |
| 🔭 If productionised | Constrained query builder with parameterised templates | Future |

---

### Issue #2 — No response on Google Chat (timeout)

> **Status: Active** — Users intermittently receive no response, especially for issue tracker queries involving complex BigQuery jobs.

**Observed:**
- User sends message, sees "Thinking...", waits, sees nothing
- Cloud Function logs confirm HTTP 200 was returned to Chat, but subsequent agent call timed out
- Most common on Buganizer queries with complex BigQuery jobs
- Worse after deployments due to cold starts

**Learned:**
Google Chat has a hard 30-second limit. Vertex AI agent calls involving BigQuery routinely take 15–45 seconds. The background thread pattern resolved most timeouts but introduced a new failure mode: threads that die when the Cloud Function instance is recycled. We discovered this by correlating "no response" user reports with instance recycling events in Cloud Logging.

A second failure mode: the Chat API POST delivering the real response can fail due to an expired OAuth token or rate limit — and these failures were not being logged. Users saw "Thinking..." with no follow-up because the response was computed but never delivered.

**Changed:**
- Error handling in background thread: always post to Chat on failure, never fail silently
- Cloud Function timeout increased from 60 to 540 seconds (the maximum)
- Structured logging added for all Chat API post failures

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| ✅ Done | Async pattern (HTTP 200 before agent call) | Deployed |
| ✅ Done | Error callback — always post on failure | Deployed |
| ✅ Done | Cloud Function timeout → 540 seconds | Deployed |
| ✅ Done | Structured logging for Chat API failures | Deployed |
| 🔧 Next | Cloud Tasks queue (replaces background thread) | Open |
| 🔧 Next | "Still working..." follow-up at 15-second mark | Open |
| 🔭 If productionised | Full Pub/Sub async architecture | Future |

---

### Issue #3 — No real-time sync with code repo and Buganizer

> **Status: Known limitation** — Buganizer data can be up to 4 hours stale. Code files can be up to 24 hours stale. Users are reporting answers about issues that have already been resolved.

**Observed:**
- Newly filed Buganizer issues do not appear in agent responses for up to 4 hours
- Code changes merged to `main` are not reflected in agent responses for up to 24 hours
- Agent says a bug is "Open / Assigned" when it was resolved the same morning
- Sprint velocity data is inaccurate between export runs

**Learned:**
This is a fundamental architectural constraint of the batch export design. The Buganizer export pipeline runs every 4 hours — too slow for active sprint management queries. The code snapshot export triggers on CI/CD completion but still faces Vertex AI Search re-index latency of 15–60 minutes.

The deeper issue: Vertex AI Search re-indexing is not instant. Even with an immediate file upload, propagation to the search index takes 15–60 minutes. Truly real-time code search is not achievable with the current data store architecture — this is a known POC constraint.

**Planned:**

Switching to incremental `MERGE` operations for Buganizer (processing only records changed since the last run):

```python
from google.cloud.bigquery_storage_v1 import BigQueryWriteClient
from google.cloud.bigquery_storage_v1.types import WriteStream

client = BigQueryWriteClient()
write_stream = client.create_write_stream(
    parent=table_path,
    write_stream=WriteStream(type_=WriteStream.Type.COMMITTED)
)
# Append individual issue updates as they arrive via Pub/Sub
```

Adding an Eventarc trigger on Cloud Storage object creation to initiate on-demand Vertex AI Search re-index of only changed files (target: ~15-minute propagation for code).

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| 🔧 Next | Incremental BigQuery MERGE (more frequent refresh) | Open |
| 🔧 Next | Eventarc trigger → on-demand Vertex AI re-index | Open |
| 🔭 If productionised | Buganizer Pub/Sub → BigQuery Storage Write API | Future |
| 🔭 If productionised | Direct Buganizer API (gRPC streaming) | Future |
| 🔭 If productionised | Direct Git connector (Cloud Source Repositories) | Future |

---

## Security & IAM

### Service accounts

| Service account | Roles | Used by |
|---|---|---|
| `vertex-agent-sa` | `aiplatform.user`, `discoveryengine.viewer`, `bigquery.dataViewer`, `bigquery.jobUser` | Vertex AI Agent Engine |
| `cloud-function-sa` | `run.invoker`, `chat.bot`, `aiplatform.user` | Cloud Function |
| `bq-export-sa` | `bigquery.dataEditor`, `storage.objectAdmin` | Dataflow / export pipeline |
| `ci-export-sa` | `storage.objectCreator` | Cloud Build code snapshot export |

### Controls in place

- Uniform bucket-level access on all Cloud Storage buckets (no legacy ACLs)
- BigQuery column-level security on PII fields (`assignee`, `reporter` emails)
- Google-signed JWT validation on the Cloud Function webhook
- Service account impersonation throughout — no long-lived credentials stored anywhere
- VPC Service Controls perimeter around Vertex AI, BigQuery, and Cloud Storage (production only)

### Still to do

- Audit log dashboard: Cloud Logging captures all data store access but no dashboard exists yet
- Document-level access control: currently any user in the Chat space can query any document in the bucket — no per-document ACL enforcement

---

## No-Code / Low-Code Context — Why This Matters Now

This POC sits at an interesting intersection: it was built by engineers, using tools that are themselves products of the no-code / low-code revolution. Understanding where that ecosystem is heading contextualises both what this system can do today and where the remaining rough edges will be smoothed out by the platforms themselves.

### The three waves

**Wave 1 — Automation & publishing (2012–2018)**
Zapier, IFTTT, Wix, Airtable. Simple connectors and content tools. The ceiling was low — any non-trivial logic required engineering handoff.

**Wave 2 — Visual app builders (2018–2022)**
Bubble, Retool, Appsmith, OutSystems, Mendix. Full-stack apps without backend code. Jira and ServiceNow both launched low-code app layers in this era, making issue data more accessible to non-engineering teams.

**Wave 3 — AI-native builders (2022–present)**

| Category | Platforms | What changed |
|---|---|---|
| AI agent builders | Vertex AI Agent Designer, Microsoft Copilot Studio | Configure multi-agent pipelines in natural language |
| App generators | v0, Lovable, bolt.new | Describe the interface → get working code |
| Embedded AI copilots | Power Platform, Google AppSheet | AI generates workflow logic inside the builder canvas |
| Enterprise AI layers | Jira AI (Atlassian Intelligence), ServiceNow Now Intelligence | Managed NL query over issue data — no custom agent needed |

### Where this POC fits

This system is a Wave 3 product built with Wave 3 tools. The agent configuration, data store connections, and routing logic were set up through Vertex AI Agent Designer's console — no ML code written. What required an ML engineering team in 2021 is now configuration work.

The remaining friction — NL-to-SQL inconsistency, re-index latency, the Chat timeout — maps precisely onto where the platforms are still maturing. Jira's Atlassian Intelligence and ServiceNow's Now Intelligence are building managed NL-to-SQL layers that will eventually make the BigQuery intermediary unnecessary for those platforms. For Buganizer, the same trajectory applies through Google's internal tooling investments.

**The design implication:** The architectural decisions in this system that feel like engineering compromises (batch exports, `.txt` code snapshots, threading-based async) are mostly temporary workarounds for platform gaps that are actively closing. Build for the platform to meet you — do not over-engineer permanent solutions to what the managed service will solve in 18 months.

### What no-code / low-code still cannot do well (mid-2025)

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

- Replace the background thread with a Cloud Tasks queue (fixes the Chat timeout reliability)
- Add a SQL validation layer and schema auto-sync (fixes Buganizer response inconsistency)
- Switch to incremental BigQuery MERGE and add an Eventarc re-index trigger (reduces data staleness)

None of these require changing the agent design or the data store structure — they are improvements to the plumbing.

### Data freshness and real-time sync

The batch export architecture is the biggest gap between "POC" and "production-useful". The path forward:

- **For Buganizer:** If a Pub/Sub change notification feed becomes available, use it to stream individual issue updates to BigQuery via the Storage Write API rather than batch exporting the full dataset. This collapses the 4-hour lag to near-real-time.
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

- **Meeting notes and decisions:** Recorded meeting transcripts or decision logs are a common source of "why did we do it this way?" questions that documents don't capture well.
- **Monitoring and SLOs:** Connecting to Cloud Monitoring or a metrics store would let the agent answer "is the service healthy right now?" alongside "what is the SLA supposed to be?".
- **On-call history:** Past incident timelines and post-mortems are highly valuable for new team members and are often trapped in docs no one reads.

### If this moves toward production

A few things that are deliberately out of scope for the POC but would matter at scale:

- **Per-document access control:** Right now, every user in the Chat space can query every document in the bucket. A production system needs to respect the same access controls as the source documents.
- **Audit logging dashboard:** Cloud Logging already captures all data access — it just needs a dashboard to make it reviewable.
- **Cost model:** Vertex AI Search, BigQuery, and Cloud Functions all have usage-based costs. At low POC volume these are negligible; at team-wide adoption they need to be monitored.
- **Multi-space / multi-program support:** The current design serves one program's data. A shared infrastructure model would let multiple programs each bring their own data stores and share the orchestration layer.

---

## How to Contribute to This POC

This is a learning document as much as a technical spec. Contributions in three forms are welcome:

**1. Code and configuration**
Fork, branch, and PR as usual. For agent system prompt changes, include before/after accuracy notes in the PR description. For data pipeline changes, include freshness impact estimates.

**2. Issue reports and learnings**
If you hit a new failure mode not documented here, open an issue with: what you asked, what you got, what you expected. The "What we learned" sections above came from exactly these reports.

**3. Document and schema updates**
Add documents to the program-docs bucket following the folder structure. Notify the team in `#knowledge-guide` so others know new content is searchable.

```bash
# Branch naming convention
git checkout -b feature/your-feature-name
git checkout -b fix/issue-description
git checkout -b learning/what-was-discovered
```

---

*This document is updated as the POC evolves. For questions, reach out in the `#knowledge-guide` Google Chat space. Last significant revision: June 2025.*


# Multi-Agent Knowledge Guide System
### A Learning-While-Building POC on Google Cloud Vertex AI Agent Platform

> **What this is:** A working proof-of-concept and living technical journal. Every section documents not just *what* was built, but *what was learned*, *what broke*, and *what was changed as a result*. The goal is to make this a reference for anyone building similar AI-agent systems on Google Cloud — without the benefit of a clean retrospective.

---

## Table of Contents

- [The Idea](#the-idea)
- [Who This Is For](#who-this-is-for)
- [What We Learned Building It](#what-we-learned-building-it)
- [Architecture — How We Got Here](#architecture--how-we-got-here)
- [Data Sources — The Three Pillars](#data-sources--the-three-pillars)
- [Agent Design — What We Built and Why](#agent-design--what-we-built-and-why)
- [Google Chat Integration — The Hard Part](#google-chat-integration--the-hard-part)
- [Active Issues — Still Learning](#active-issues--still-learning)
- [Security & IAM](#security--iam)
- [No-Code / Low-Code Context — Why This Matters Now](#no-code--low-code-context--why-this-matters-now)
- [Next Steps](#next-steps)
- [How to Contribute to This POC](#how-to-contribute-to-this-poc)

---

## The Idea

Every engineering program accumulates a mountain of knowledge — design documents, onboarding guides, runbooks, issue histories, source code. That knowledge lives in five different systems and is effectively inaccessible unless you know exactly where to look or who to ask.

The hypothesis was simple: **what if your team could ask questions in plain English, in Google Chat, and get answers drawn from all of those sources at once?**

This POC tests that hypothesis using Google Cloud's Vertex AI Agent Designer to orchestrate multiple specialised AI agents — one for documents, one for issue tracking data, one for source code — all reachable through a single conversational interface.

It is not finished. It works, imperfectly, and that's the point. This document is as much about what we learned designing and building it as it is a technical spec.

---

## Who This Is For

This POC was built to serve four groups — and the design decisions at every layer were shaped by their different needs:

| Audience | Primary use | Example question |
|---|---|---|
| **Technical team** (SWE / SRE) | Code understanding, issue triage, runbook lookup | *"What does the `DataPipeline` class do?"* |
| **Product team** (PM / TPM) | Feature status, sprint health, decision history | *"Summarise open P1s in the Payments component"* |
| **Business / leadership** | Program status, KPIs, risk summary | *"What are the current blockers for Project Titan?"* |
| **New joiners** | Onboarding, team structure, tooling setup | *"How do I set up my dev environment?"* |

**Early learning:** We initially designed primarily for the technical team. The first demo to a PM revealed that the query patterns were completely different — PMs want summaries and status, not raw data. The routing logic in the Orchestrator Agent was redesigned after that session.

---

## What We Learned Building It

This is the section we wish we had found before starting. A summary of the non-obvious lessons, with pointers to where each is documented in detail.

### On agent design

- **Start with one agent, not many.** We built three sub-agents on day one. We should have started with one and split only when a single agent demonstrably could not handle the context switching. Multi-agent overhead is real — routing errors compound.
- **The system prompt is the product.** The quality of agent responses is almost entirely determined by how precisely the system prompt describes the agent's role, data format, and constraints. Treat it like code — version it, review it, test it.
- **Temperature matters for structured data.** NL-to-SQL with temperature > 0 produces non-deterministic SQL. The same query can return different results across sessions. Setting `temperature=0` for the issue tracker sub-agent was the single highest-impact fix we made.

### On data integration

- **Buganizer via BigQuery has a freshness problem.** Buganizer — Google's internal issue and feature tracking system, analogous to Jira for bug/feature lifecycle management — exports to BigQuery on a batch schedule. Users asking about recently filed issues get stale answers. This is the most-complained-about limitation and is actively being addressed (see [Issue #3](#issue-3--no-real-time-sync-with-code-repo-and-buganizer)).
- **Code-as-text works better than expected.** Storing source files as `.txt` in Cloud Storage felt like a compromise, but Vertex AI Search indexes them well and semantic search over code is surprisingly effective for "what does X do?" queries. The limitation shows up on "how does X interact with Y?" — cross-file reasoning is weak.
- **Document freshness is an invisible problem.** If a document is updated in Drive and not re-uploaded to the bucket, the agent answers from the old version with full confidence. We needed to make this visible.

### On the Google Chat integration

- **The 30-second timeout is the hardest constraint in the system.** Google Chat requires your webhook to respond within 30 seconds or it drops the connection silently — the user sees nothing. Vertex AI agent calls, especially ones that hit BigQuery, routinely take longer. This caused the most user-visible failures and required an architectural change. See [Issue #2](#issue-2--no-response-on-google-chat-timeout).
- **"Thinking..." is not enough UX.** Users who sent a complex query and saw only "Thinking..." for 20 seconds assumed the bot was broken. Adding a brief description of what the agent is looking up dramatically improved perceived reliability even before the response arrived.

---

## Architecture — How We Got Here

### Current architecture

The system has four layers. This is not the architecture we designed on day one — it evolved through three significant restructuring events documented below.

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1 — PRESENTATION                                     │
│  Google Chat  (natural language queries via chat space)     │
└───────────────────────┬─────────────────────────────────────┘
                        │ HTTP POST webhook
┌───────────────────────▼─────────────────────────────────────┐
│  LAYER 2 — MIDDLEWARE                                       │
│  Cloud Function (2nd gen, Python 3.12)                      │
│  JWT validation · session management · async fire-and-forget│
└───────────────────────┬─────────────────────────────────────┘
                        │ detect_intent API
┌───────────────────────▼─────────────────────────────────────┐
│  LAYER 3 — AGENT ORCHESTRATION (Vertex AI Agent Designer)    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Orchestrator Agent                                  │   │
│  │  intent classification · routing · aggregation       │   │
│  └──────────┬──────────────────┬────────────┬───────────┘   │
│             │                  │            │               │
│    ┌────────▼──────┐  ┌────────▼──────┐  ┌─▼────────────┐ │
│    │  Doc agent    │  │  Issue agent  │  │  Code agent  │ │
│    │ (documents)   │  │  (Buganizer)  │  │ (.txt files) │ │
│    └───────────────┘  └───────────────┘  └──────────────┘ │
└──────────┬─────────────────┬──────────────────┬────────────┘
           │                 │                  │
  ┌────────▼──────┐  ┌───────▼────────┐  ┌─────▼──────────┐
  │  Cloud        │  │   BigQuery     │  │  Cloud Storage │
  │  Storage      │  │  (Buganizer    │  │  (code .txt    │
  │  (docs)       │  │   export)      │  │   snapshots)   │
  └───────────────┘  └────────────────┘  └────────────────┘
```

### How the architecture evolved

**V1 — Single agent, all data stores**
Started with one Vertex AI agent connected to all three data stores. Context window quickly became a bottleneck — the agent confused document content with issue data on ambiguous queries. Response quality was inconsistent.

**V2 — Sub-agents, synchronous middleware**
Introduced the Orchestrator + three sub-agents pattern. Cloud Function called the agent synchronously. This immediately exposed the 30-second timeout problem — any BigQuery query that took longer produced a silent failure in Chat.

**V3 — Async middleware (current)**
Cloud Function now returns HTTP 200 immediately and calls the agent asynchronously in a background thread. This resolved the timeout failures for most queries. The remaining failure mode is thread reliability on cold-start instances (see [Issue #2](#issue-2--no-response-on-google-chat-timeout)).

### Technology stack

| Layer | Technology | Purpose |
|---|---|---|
| Agent platform | Vertex AI Agent Designer | Hosts orchestrator and sub-agents |
| Orchestration | Vertex AI Agent Engine | Routes queries, aggregates responses |
| Document data store | Vertex AI Search on Cloud Storage | Semantic search over program docs |
| Issue tracker data store | BigQuery + Vertex AI BigQuery connector | Structured query on Buganizer export |
| Code data store | Vertex AI Search on Cloud Storage (.txt) | Semantic search over code files |
| Middleware | Cloud Functions (2nd gen, Python 3.12) | HTTP webhook bridge, async orchestration |
| Chat interface | Google Chat App | User-facing conversational UI |
| Auth | Google Service Accounts + Workload Identity | Secure agent-to-data-store access |
| Observability | Cloud Logging, Cloud Trace, BigQuery audit logs | Tracing, latency, error tracking |

---

## Data Sources — The Three Pillars

### Pillar 1 — Program documents (Cloud Storage)

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
- Unstructured document index with semantic chunking (500 tokens, 50-token overlap)
- Embeddings: `textembedding-gecko@003`
- Refresh: Cloud Scheduler every 6 hours + Eventarc trigger on new uploads

**What works well:** Summarisation, policy lookup, onboarding Q&A, finding the right document when the user does not know its name.

**What does not work well:** Cross-document synthesis, version awareness (the agent does not know if it is reading an outdated version), very large PDFs where relevant content is buried in a single deeply nested paragraph.

---

### Pillar 2 — Issue tracker data (BigQuery / Buganizer)

#### What is Buganizer?

**Buganizer is Google's internal issue and feature tracking system** — the equivalent of Jira for engineering bug and feature lifecycle management, or ServiceNow for IT service management workflows.

Like those enterprise tools, Buganizer:
- Tracks issues through defined lifecycle states: Open → Assigned → Fixed → Verified → Closed
- Classifies work by priority (P0 critical → P4 low) and severity (S0 emergency → S3 minor)
- Organises issues by component and product area
- Assigns ownership to engineers and teams
- Links issues to sprints, milestones, and releases
- Supports structured querying, reporting, and dashboarding

For teams outside Google, the closest public-facing equivalent is the **Google Issue Tracker** (issuetracker.google.com), which runs on the same underlying system. Any reference to "Buganizer data" in this document means structured issue metadata from this system, exported to BigQuery for agent access.

Teams using Jira or ServiceNow can replicate this exact integration pattern: export structured issue data to BigQuery via the Jira REST API or ServiceNow table API, then point the Vertex AI sub-agent at the resulting BigQuery dataset. The NL-to-SQL approach and schema design translate directly.

| Capability | Jira (typical) | ServiceNow (typical) | Buganizer (this POC) |
|---|---|---|---|
| Real-time API | REST + webhooks | REST + webhooks | Batch export to BigQuery |
| Query language | JQL | GlideQuery | NL-to-SQL via LLM |
| Agent integration | Atlassian Intelligence | Now Intelligence | Custom Vertex AI sub-agent |
| Data freshness | Near real-time | Near real-time | 4-hour lag (current) |
| AI query layer | Atlassian AI (managed) | ServiceNow AI Search | Custom (this system) |

#### Integration approach

Because Buganizer does not currently expose a real-time streaming API suitable for direct agent query in this context, the data is exported to BigQuery on a scheduled basis and queried from there. The Issue agent uses NL-to-SQL to translate natural language questions into BigQuery queries.

**BigQuery schema — issues table:**

| Column | Type | Description |
|---|---|---|
| `issue_id` | STRING | Unique Buganizer issue identifier |
| `title` | STRING | Issue title / summary |
| `status` | STRING | OPEN, ASSIGNED, FIXED, VERIFIED, CLOSED |
| `priority` | STRING | P0 (critical) → P4 (low) |
| `severity` | STRING | S0 (emergency) → S3 (minor) |
| `component` | STRING | Product component or module |
| `assignee` | STRING | Assigned engineer (email) |
| `reporter` | STRING | Issue reporter (email) |
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
- Export frequency: every 4 hours (a known limitation — see [Next Steps](#next-steps) for improvement directions)
- Mode: `WRITE_TRUNCATE` for small tables; incremental `MERGE` for large tables
- Deduplication: `issue_id + updated_at`
- Partitioning: `issues` table by `created_at` DATE
- Clustering: `(priority, status, component)` — matches common query filter patterns

---

### Pillar 3 — Source code (Cloud Storage / .txt files)

Source files are exported from the code repository as `.txt` files via CI/CD and indexed by Vertex AI Search.

**Why .txt and not direct repo integration?**

Direct repository integration was the original plan. We moved away from it for the POC because: Vertex AI Search indexes unstructured text rather than code natively; real-time repo webhooks add significant middleware complexity; and `.txt` export via Cloud Build gives us control over exactly which files are indexed (no generated files, no binaries). This is a deliberate short-term compromise — see [Next Steps](#next-steps) for where this goes.

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

**What does not work well:** Cross-file reasoning, recently merged code (up to 24-hour lag), generated or minified files that slip past the exclusion filter.

---

## Agent Design — What We Built and Why

### The Orchestrator Agent

The Orchestrator is the entry point for all queries. It classifies intent and routes to the appropriate sub-agent. For cross-domain queries it calls multiple sub-agents in parallel and aggregates the responses.

**Routing table:**

| User intent | Target | Trigger signals |
|---|---|---|
| Document / policy | Doc agent | onboarding, architecture, SLA, PRD, design doc, spec, runbook |
| Issue / bug / feature | Issue agent | bug, issue, P0, P1, sprint, blockers, Buganizer, ticket, defect, feature request |
| Code / implementation | Code agent | function, class, module, file, implementation, source, code, API |
| Cross-domain | All agents (parallel) | status, summary, program health, tell me about |

**Learning from routing failures:**

Early versions frequently mis-routed "how does authentication work?" to the Code agent because of the word "how does", when the user actually wanted the architecture document. Adding explicit disambiguation examples to the system prompt significantly improved routing accuracy. The examples in the system prompt matter as much as the routing rules.

### The Issue Agent (Buganizer)

**Design constraints:**
- Must generate only `SELECT` statements — no DML under any circumstances
- Must include the generated SQL in its response for transparency
- Must handle empty result sets gracefully with helpful guidance on query refinement
- Must limit result sets to 50 rows by default with pagination support

**System prompt structure (simplified):**
```
You are an issue tracker specialist with read-only access to a BigQuery dataset
containing exported Buganizer issue data. Buganizer is Google's internal issue
tracking system equivalent to Jira or ServiceNow.

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

### The 30-second problem

Integrating with Google Chat looked straightforward: expose a webhook, receive a message, call the agent, return the response. In practice, Google Chat's 30-second synchronous response requirement creates a fundamental tension with AI agent response times. This section documents what we tried and what we learned.

### How the integration works (current state)

```
1.  User sends message in Google Chat space
2.  Chat delivers HTTP POST to Cloud Function webhook
3.  Cloud Function validates Google-signed JWT
4.  Cloud Function returns HTTP 200 + "Thinking..." IMMEDIATELY
    └── Critical: must happen within 30 seconds or Chat drops the connection silently
5.  Cloud Function spawns background thread to call Vertex AI agent
6.  Agent Engine routes to sub-agent(s), queries data stores
7.  Agent returns generated response
8.  Cloud Function calls Chat REST API to post the real response
```

### Async pattern (current implementation)

```python
@functions_framework.http
def handle_chat_event(request):
    payload = request.get_json()
    verify_google_chat_jwt(request)

    # Return 200 FIRST — do not wait for the agent
    send_typing_indicator(payload['space']['name'])

    # Fire background thread for agent work
    thread = threading.Thread(
        target=call_agent_and_respond,
        args=(payload,)
    )
    thread.start()
    return 'OK', 200


def call_agent_and_respond(payload):
    try:
        session_id = f"{payload['space']['name']}_{payload['user']['name']}"
        response = call_vertex_agent(payload['message']['text'], session_id)
        post_to_chat(payload['space']['name'], response)
    except Exception as e:
        # Always post something — silent failures are worse than error messages
        post_to_chat(payload['space']['name'],
                     f"Something went wrong: {str(e)}. Please try again.")
```

### Next: Cloud Tasks pattern (recommended improvement)

The background thread approach has a reliability problem: if the Cloud Function instance is recycled mid-execution, the thread dies silently. Cloud Tasks eliminates this:

```python
def handle_chat_event(request):
    payload = request.get_json()
    verify_google_chat_jwt(request)

    client = tasks_v2.CloudTasksClient()
    task = {
        'http_request': {
            'http_method': tasks_v2.HttpMethod.POST,
            'url': AGENT_WORKER_URL,
            'body': json.dumps(payload).encode(),
        }
    }
    client.create_task(parent=QUEUE_PATH, task=task)

    send_thinking_message(payload)   # immediate ack to Chat
    return 'Queued', 200
```

### Environment variables

| Variable | Description | Example |
|---|---|---|
| `VERTEX_PROJECT_ID` | GCP project hosting the agent | `my-project-123` |
| `VERTEX_LOCATION` | Agent region | `us-central1` |
| `AGENT_ID` | Vertex AI Agent Designer agent ID | `abc123def456` |
| `CHAT_SA_EMAIL` | Service account for Chat API calls | `chat-bot@project.iam.gserviceaccount.com` |
| `MAX_RESPONSE_WAIT_SECS` | Timeout for agent response | `45` |

---

## Active Issues — Still Learning

These are not bugs to fix before launch. They are the active learning surface of the POC — open questions about how to build AI agent systems well. Each one is documented with what was observed, what was learned, what was changed, and what remains to be done.

---

### Issue #1 — Inconsistent responses on Buganizer data

> **Status: Active** — The same question about issue data can produce different answers in different sessions, even when the underlying data has not changed.

**Observed:**
- P0 bug count returns 12 in one session, 11 in another with no data change
- Occasionally returns empty result for queries that should match hundreds of rows
- Component name misspellings cause total query failure with no helpful error message
- Multi-table JOIN queries are more prone to inconsistency than single-table queries

**Learned:**
The root cause is NL-to-SQL non-determinism. At `temperature > 0`, the LLM generating SQL from natural language produces structurally different queries for the same input. A query like "open P0s in Authentication" might generate `component = 'Authentication'` in one session and `component LIKE '%auth%'` in another — returning different result sets.

A secondary problem: the BigQuery schema is embedded in the agent system prompt as static text. When the schema evolves, the prompt is not updated automatically, and the LLM generates SQL referencing columns that no longer exist.

**Changed:**
- Set `temperature=0` for the Issue agent — the single highest-impact change
- Added explicit schema documentation with column descriptions and valid enum values to the system prompt
- Pre-built BigQuery Views for the 8 most common query patterns and added these as named tools in the agent

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| ✅ Done | `temperature=0` for Issue agent | Deployed |
| ✅ Done | Pre-built Views for common queries | Deployed |
| 🔧 Next | SQL validation layer in Cloud Function | Open |
| 🔧 Next | Automated schema sync via Data Catalog events | Open |
| 🔧 Next | Redis query result cache (15-min TTL) | Open |
| 🔭 If productionised | Constrained query builder with parameterised templates | Future |

---

### Issue #2 — No response on Google Chat (timeout)

> **Status: Active** — Users intermittently receive no response, especially for issue tracker queries involving complex BigQuery jobs.

**Observed:**
- User sends message, sees "Thinking...", waits, sees nothing
- Cloud Function logs confirm HTTP 200 was returned to Chat, but subsequent agent call timed out
- Most common on Buganizer queries with complex BigQuery jobs
- Worse after deployments due to cold starts

**Learned:**
Google Chat has a hard 30-second limit. Vertex AI agent calls involving BigQuery routinely take 15–45 seconds. The background thread pattern resolved most timeouts but introduced a new failure mode: threads that die when the Cloud Function instance is recycled. We discovered this by correlating "no response" user reports with instance recycling events in Cloud Logging.

A second failure mode: the Chat API POST delivering the real response can fail due to an expired OAuth token or rate limit — and these failures were not being logged. Users saw "Thinking..." with no follow-up because the response was computed but never delivered.

**Changed:**
- Error handling in background thread: always post to Chat on failure, never fail silently
- Cloud Function timeout increased from 60 to 540 seconds (the maximum)
- Structured logging added for all Chat API post failures

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| ✅ Done | Async pattern (HTTP 200 before agent call) | Deployed |
| ✅ Done | Error callback — always post on failure | Deployed |
| ✅ Done | Cloud Function timeout → 540 seconds | Deployed |
| ✅ Done | Structured logging for Chat API failures | Deployed |
| 🔧 Next | Cloud Tasks queue (replaces background thread) | Open |
| 🔧 Next | "Still working..." follow-up at 15-second mark | Open |
| 🔭 If productionised | Full Pub/Sub async architecture | Future |

---

### Issue #3 — No real-time sync with code repo and Buganizer

> **Status: Known limitation** — Buganizer data can be up to 4 hours stale. Code files can be up to 24 hours stale. Users are reporting answers about issues that have already been resolved.

**Observed:**
- Newly filed Buganizer issues do not appear in agent responses for up to 4 hours
- Code changes merged to `main` are not reflected in agent responses for up to 24 hours
- Agent says a bug is "Open / Assigned" when it was resolved the same morning
- Sprint velocity data is inaccurate between export runs

**Learned:**
This is a fundamental architectural constraint of the batch export design. The Buganizer export pipeline runs every 4 hours — too slow for active sprint management queries. The code snapshot export triggers on CI/CD completion but still faces Vertex AI Search re-index latency of 15–60 minutes.

The deeper issue: Vertex AI Search re-indexing is not instant. Even with an immediate file upload, propagation to the search index takes 15–60 minutes. Truly real-time code search is not achievable with the current data store architecture — this is a known POC constraint.

**Planned:**

Switching to incremental `MERGE` operations for Buganizer (processing only records changed since the last run):

```python
from google.cloud.bigquery_storage_v1 import BigQueryWriteClient
from google.cloud.bigquery_storage_v1.types import WriteStream

client = BigQueryWriteClient()
write_stream = client.create_write_stream(
    parent=table_path,
    write_stream=WriteStream(type_=WriteStream.Type.COMMITTED)
)
# Append individual issue updates as they arrive via Pub/Sub
```

Adding an Eventarc trigger on Cloud Storage object creation to initiate on-demand Vertex AI Search re-index of only changed files (target: ~15-minute propagation for code).

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| 🔧 Next | Incremental BigQuery MERGE (more frequent refresh) | Open |
| 🔧 Next | Eventarc trigger → on-demand Vertex AI re-index | Open |
| 🔭 If productionised | Buganizer Pub/Sub → BigQuery Storage Write API | Future |
| 🔭 If productionised | Direct Buganizer API (gRPC streaming) | Future |
| 🔭 If productionised | Direct Git connector (Cloud Source Repositories) | Future |

---

## Security & IAM

### Service accounts

| Service account | Roles | Used by |
|---|---|---|
| `vertex-agent-sa` | `aiplatform.user`, `discoveryengine.viewer`, `bigquery.dataViewer`, `bigquery.jobUser` | Vertex AI Agent Engine |
| `cloud-function-sa` | `run.invoker`, `chat.bot`, `aiplatform.user` | Cloud Function |
| `bq-export-sa` | `bigquery.dataEditor`, `storage.objectAdmin` | Dataflow / export pipeline |
| `ci-export-sa` | `storage.objectCreator` | Cloud Build code snapshot export |

### Controls in place

- Uniform bucket-level access on all Cloud Storage buckets (no legacy ACLs)
- BigQuery column-level security on PII fields (`assignee`, `reporter` emails)
- Google-signed JWT validation on the Cloud Function webhook
- Service account impersonation throughout — no long-lived credentials stored anywhere
- VPC Service Controls perimeter around Vertex AI, BigQuery, and Cloud Storage (production only)

### Still to do

- Audit log dashboard: Cloud Logging captures all data store access but no dashboard exists yet
- Document-level access control: currently any user in the Chat space can query any document in the bucket — no per-document ACL enforcement

---

## No-Code / Low-Code Context — Why This Matters Now

This POC sits at an interesting intersection: it was built by engineers, using tools that are themselves products of the no-code / low-code revolution. Understanding where that ecosystem is heading contextualises both what this system can do today and where the remaining rough edges will be smoothed out by the platforms themselves.

### The three waves

**Wave 1 — Automation & publishing (2012–2018)**
Zapier, IFTTT, Wix, Airtable. Simple connectors and content tools. The ceiling was low — any non-trivial logic required engineering handoff.

**Wave 2 — Visual app builders (2018–2022)**
Bubble, Retool, Appsmith, OutSystems, Mendix. Full-stack apps without backend code. Jira and ServiceNow both launched low-code app layers in this era, making issue data more accessible to non-engineering teams.

**Wave 3 — AI-native builders (2022–present)**

| Category | Platforms | What changed |
|---|---|---|
| AI agent builders | Vertex AI Agent Designer, Microsoft Copilot Studio | Configure multi-agent pipelines in natural language |
| App generators | v0, Lovable, bolt.new | Describe the interface → get working code |
| Embedded AI copilots | Power Platform, Google AppSheet | AI generates workflow logic inside the builder canvas |
| Enterprise AI layers | Jira AI (Atlassian Intelligence), ServiceNow Now Intelligence | Managed NL query over issue data — no custom agent needed |

### Where this POC fits

This system is a Wave 3 product built with Wave 3 tools. The agent configuration, data store connections, and routing logic were set up through Vertex AI Agent Designer's console — no ML code written. What required an ML engineering team in 2021 is now configuration work.

The remaining friction — NL-to-SQL inconsistency, re-index latency, the Chat timeout — maps precisely onto where the platforms are still maturing. Jira's Atlassian Intelligence and ServiceNow's Now Intelligence are building managed NL-to-SQL layers that will eventually make the BigQuery intermediary unnecessary for those platforms. For Buganizer, the same trajectory applies through Google's internal tooling investments.

**The design implication:** The architectural decisions in this system that feel like engineering compromises (batch exports, `.txt` code snapshots, threading-based async) are mostly temporary workarounds for platform gaps that are actively closing. Build for the platform to meet you — do not over-engineer permanent solutions to what the managed service will solve in 18 months.

### What no-code / low-code still cannot do well (mid-2025)

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

- Replace the background thread with a Cloud Tasks queue (fixes the Chat timeout reliability)
- Add a SQL validation layer and schema auto-sync (fixes Buganizer response inconsistency)
- Switch to incremental BigQuery MERGE and add an Eventarc re-index trigger (reduces data staleness)

None of these require changing the agent design or the data store structure — they are improvements to the plumbing.

### Data freshness and real-time sync

The batch export architecture is the biggest gap between "POC" and "production-useful". The path forward:

- **For Buganizer:** If a Pub/Sub change notification feed becomes available, use it to stream individual issue updates to BigQuery via the Storage Write API rather than batch exporting the full dataset. This collapses the 4-hour lag to near-real-time.
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

- **Meeting notes and decisions:** Recorded meeting transcripts or decision logs are a common source of "why did we do it this way?" questions that documents don't capture well.
- **Monitoring and SLOs:** Connecting to Cloud Monitoring or a metrics store would let the agent answer "is the service healthy right now?" alongside "what is the SLA supposed to be?".
- **On-call history:** Past incident timelines and post-mortems are highly valuable for new team members and are often trapped in docs no one reads.

### If this moves toward production

A few things that are deliberately out of scope for the POC but would matter at scale:

- **Per-document access control:** Right now, every user in the Chat space can query every document in the bucket. A production system needs to respect the same access controls as the source documents.
- **Audit logging dashboard:** Cloud Logging already captures all data access — it just needs a dashboard to make it reviewable.
- **Cost model:** Vertex AI Search, BigQuery, and Cloud Functions all have usage-based costs. At low POC volume these are negligible; at team-wide adoption they need to be monitored.
- **Multi-space / multi-program support:** The current design serves one program's data. A shared infrastructure model would let multiple programs each bring their own data stores and share the orchestration layer.

---

## How to Contribute to This POC

This is a learning document as much as a technical spec. Contributions in three forms are welcome:

**1. Code and configuration**
Fork, branch, and PR as usual. For agent system prompt changes, include before/after accuracy notes in the PR description. For data pipeline changes, include freshness impact estimates.

**2. Issue reports and learnings**
If you hit a new failure mode not documented here, open an issue with: what you asked, what you got, what you expected. The "What we learned" sections above came from exactly these reports.

**3. Document and schema updates**
Add documents to the program-docs bucket following the folder structure. Notify the team in `#knowledge-guide` so others know new content is searchable.

```bash
# Branch naming convention
git checkout -b feature/your-feature-name
git checkout -b fix/issue-description
git checkout -b learning/what-was-discovered
```

---

*This document is updated as the POC evolves. For questions, reach out in the `#knowledge-guide` Google Chat space. Last significant revision: June 2025.*


# Multi-Agent Knowledge Guide System
### A Learning-While-Building POC on Google Cloud Vertex AI Agent Platform

> **What this is:** A working proof-of-concept and living technical journal. Every section documents not just *what* was built, but *what was learned*, *what broke*, and *what was changed as a result*. The goal is to make this a reference for anyone building similar AI-agent systems on Google Cloud — without the benefit of a clean retrospective.

---

## Table of Contents

- [The Idea](#the-idea)
- [Who This Is For](#who-this-is-for)
- [What We Learned Building It](#what-we-learned-building-it)
- [Architecture — How We Got Here](#architecture--how-we-got-here)
- [Data Sources — The Three Pillars](#data-sources--the-three-pillars)
- [Agent Design — What We Built and Why](#agent-design--what-we-built-and-why)
- [Google Chat Integration — The Hard Part](#google-chat-integration--the-hard-part)
- [Active Issues — Still Learning](#active-issues--still-learning)
- [Security & IAM](#security--iam)
- [No-Code / Low-Code Context — Why This Matters Now](#no-code--low-code-context--why-this-matters-now)
- [Next Steps](#next-steps)
- [How to Contribute to This POC](#how-to-contribute-to-this-poc)

---

## The Idea

Every engineering program accumulates a mountain of knowledge — design documents, onboarding guides, runbooks, issue histories, source code. That knowledge lives in five different systems and is effectively inaccessible unless you know exactly where to look or who to ask.

The hypothesis was simple: **what if your team could ask questions in plain English, in Google Chat, and get answers drawn from all of those sources at once?**

This POC tests that hypothesis using Google Cloud's Vertex AI Agent Designer to orchestrate multiple specialised AI agents — one for documents, one for issue tracking data, one for source code — all reachable through a single conversational interface.

It is not finished. It works, imperfectly, and that's the point. This document is as much about what we learned designing and building it as it is a technical spec.

---

## Who This Is For

This POC was built to serve four groups — and the design decisions at every layer were shaped by their different needs:

| Audience | Primary use | Example question |
|---|---|---|
| **Technical team** (SWE / SRE) | Code understanding, issue triage, runbook lookup | *"What does the `DataPipeline` class do?"* |
| **Product team** (PM / TPM) | Feature status, sprint health, decision history | *"Summarise open P1s in the Payments component"* |
| **Business / leadership** | Program status, KPIs, risk summary | *"What are the current blockers for Project Titan?"* |
| **New joiners** | Onboarding, team structure, tooling setup | *"How do I set up my dev environment?"* |

**Early learning:** We initially designed primarily for the technical team. The first demo to a PM revealed that the query patterns were completely different — PMs want summaries and status, not raw data. The routing logic in the Orchestrator Agent was redesigned after that session.

---

## What We Learned Building It

This is the section we wish we had found before starting. A summary of the non-obvious lessons, with pointers to where each is documented in detail.

### On agent design

- **Start with one agent, not many.** We built three sub-agents on day one. We should have started with one and split only when a single agent demonstrably could not handle the context switching. Multi-agent overhead is real — routing errors compound.
- **The system prompt is the product.** The quality of agent responses is almost entirely determined by how precisely the system prompt describes the agent's role, data format, and constraints. Treat it like code — version it, review it, test it.
- **Temperature matters for structured data.** NL-to-SQL with temperature > 0 produces non-deterministic SQL. The same query can return different results across sessions. Setting `temperature=0` for the issue tracker sub-agent was the single highest-impact fix we made.

### On data integration

- **Buganizer via BigQuery has a freshness problem.** Buganizer — Google's internal issue and feature tracking system, analogous to Jira for bug/feature lifecycle management — exports to BigQuery on a batch schedule. Users asking about recently filed issues get stale answers. This is the most-complained-about limitation and is actively being addressed (see [Issue #3](#issue-3--no-real-time-sync-with-code-repo-and-buganizer)).
- **Code-as-text works better than expected.** Storing source files as `.txt` in Cloud Storage felt like a compromise, but Vertex AI Search indexes them well and semantic search over code is surprisingly effective for "what does X do?" queries. The limitation shows up on "how does X interact with Y?" — cross-file reasoning is weak.
- **Document freshness is an invisible problem.** If a document is updated in Drive and not re-uploaded to the bucket, the agent answers from the old version with full confidence. We needed to make this visible.

### On the Google Chat integration

- **The 30-second timeout is the hardest constraint in the system.** Google Chat requires your webhook to respond within 30 seconds or it drops the connection silently — the user sees nothing. Vertex AI agent calls, especially ones that hit BigQuery, routinely take longer. This caused the most user-visible failures and required an architectural change. See [Issue #2](#issue-2--no-response-on-google-chat-timeout).
- **"Thinking..." is not enough UX.** Users who sent a complex query and saw only "Thinking..." for 20 seconds assumed the bot was broken. Adding a brief description of what the agent is looking up dramatically improved perceived reliability even before the response arrived.

---

## Architecture — How We Got Here

### Current architecture

The system has four layers. This is not the architecture we designed on day one — it evolved through three significant restructuring events documented below.

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1 — PRESENTATION                                     │
│  Google Chat  (natural language queries via chat space)     │
└───────────────────────┬─────────────────────────────────────┘
                        │ HTTP POST webhook
┌───────────────────────▼─────────────────────────────────────┐
│  LAYER 2 — MIDDLEWARE                                       │
│  Cloud Function (2nd gen, Python 3.12)                      │
│  JWT validation · session management · async fire-and-forget│
└───────────────────────┬─────────────────────────────────────┘
                        │ detect_intent API
┌───────────────────────▼─────────────────────────────────────┐
│  LAYER 3 — AGENT ORCHESTRATION (Vertex AI Agent Designer)    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Orchestrator Agent                                  │   │
│  │  intent classification · routing · aggregation       │   │
│  └──────────┬──────────────────┬────────────┬───────────┘   │
│             │                  │            │               │
│    ┌────────▼──────┐  ┌────────▼──────┐  ┌─▼────────────┐ │
│    │  Doc agent    │  │  Issue agent  │  │  Code agent  │ │
│    │ (documents)   │  │  (Buganizer)  │  │ (.txt files) │ │
│    └───────────────┘  └───────────────┘  └──────────────┘ │
└──────────┬─────────────────┬──────────────────┬────────────┘
           │                 │                  │
  ┌────────▼──────┐  ┌───────▼────────┐  ┌─────▼──────────┐
  │  Cloud        │  │   BigQuery     │  │  Cloud Storage │
  │  Storage      │  │  (Buganizer    │  │  (code .txt    │
  │  (docs)       │  │   export)      │  │   snapshots)   │
  └───────────────┘  └────────────────┘  └────────────────┘
```

### How the architecture evolved

**V1 — Single agent, all data stores**
Started with one Vertex AI agent connected to all three data stores. Context window quickly became a bottleneck — the agent confused document content with issue data on ambiguous queries. Response quality was inconsistent.

**V2 — Sub-agents, synchronous middleware**
Introduced the Orchestrator + three sub-agents pattern. Cloud Function called the agent synchronously. This immediately exposed the 30-second timeout problem — any BigQuery query that took longer produced a silent failure in Chat.

**V3 — Async middleware (current)**
Cloud Function now returns HTTP 200 immediately and calls the agent asynchronously in a background thread. This resolved the timeout failures for most queries. The remaining failure mode is thread reliability on cold-start instances (see [Issue #2](#issue-2--no-response-on-google-chat-timeout)).

### Technology stack

| Layer | Technology | Purpose |
|---|---|---|
| Agent platform | Vertex AI Agent Designer | Hosts orchestrator and sub-agents |
| Orchestration | Vertex AI Agent Engine | Routes queries, aggregates responses |
| Document data store | Vertex AI Search on Cloud Storage | Semantic search over program docs |
| Issue tracker data store | BigQuery + Vertex AI BigQuery connector | Structured query on Buganizer export |
| Code data store | Vertex AI Search on Cloud Storage (.txt) | Semantic search over code files |
| Middleware | Cloud Functions (2nd gen, Python 3.12) | HTTP webhook bridge, async orchestration |
| Chat interface | Google Chat App | User-facing conversational UI |
| Auth | Google Service Accounts + Workload Identity | Secure agent-to-data-store access |
| Observability | Cloud Logging, Cloud Trace, BigQuery audit logs | Tracing, latency, error tracking |

---

## Data Sources — The Three Pillars

### Pillar 1 — Program documents (Cloud Storage)

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
- Unstructured document index with semantic chunking (500 tokens, 50-token overlap)
- Embeddings: `textembedding-gecko@003`
- Refresh: Cloud Scheduler every 6 hours + Eventarc trigger on new uploads

**What works well:** Summarisation, policy lookup, onboarding Q&A, finding the right document when the user does not know its name.

**What does not work well:** Cross-document synthesis, version awareness (the agent does not know if it is reading an outdated version), very large PDFs where relevant content is buried in a single deeply nested paragraph.

---

### Pillar 2 — Issue tracker data (BigQuery / Buganizer)

#### What is Buganizer?

**Buganizer is Google's internal issue and feature tracking system** — the equivalent of Jira for engineering bug and feature lifecycle management, or ServiceNow for IT service management workflows.

Like those enterprise tools, Buganizer:
- Tracks issues through defined lifecycle states: Open → Assigned → Fixed → Verified → Closed
- Classifies work by priority (P0 critical → P4 low) and severity (S0 emergency → S3 minor)
- Organises issues by component and product area
- Assigns ownership to engineers and teams
- Links issues to sprints, milestones, and releases
- Supports structured querying, reporting, and dashboarding

For teams outside Google, the closest public-facing equivalent is the **Google Issue Tracker** (issuetracker.google.com), which runs on the same underlying system. Any reference to "Buganizer data" in this document means structured issue metadata from this system, exported to BigQuery for agent access.

Teams using Jira or ServiceNow can replicate this exact integration pattern: export structured issue data to BigQuery via the Jira REST API or ServiceNow table API, then point the Vertex AI sub-agent at the resulting BigQuery dataset. The NL-to-SQL approach and schema design translate directly.

| Capability | Jira (typical) | ServiceNow (typical) | Buganizer (this POC) |
|---|---|---|---|
| Real-time API | REST + webhooks | REST + webhooks | Batch export to BigQuery |
| Query language | JQL | GlideQuery | NL-to-SQL via LLM |
| Agent integration | Atlassian Intelligence | Now Intelligence | Custom Vertex AI sub-agent |
| Data freshness | Near real-time | Near real-time | 4-hour lag (current) |
| AI query layer | Atlassian AI (managed) | ServiceNow AI Search | Custom (this system) |

#### Integration approach

Because Buganizer does not currently expose a real-time streaming API suitable for direct agent query in this context, the data is exported to BigQuery on a scheduled basis and queried from there. The Issue agent uses NL-to-SQL to translate natural language questions into BigQuery queries.

**BigQuery schema — issues table:**

| Column | Type | Description |
|---|---|---|
| `issue_id` | STRING | Unique Buganizer issue identifier |
| `title` | STRING | Issue title / summary |
| `status` | STRING | OPEN, ASSIGNED, FIXED, VERIFIED, CLOSED |
| `priority` | STRING | P0 (critical) → P4 (low) |
| `severity` | STRING | S0 (emergency) → S3 (minor) |
| `component` | STRING | Product component or module |
| `assignee` | STRING | Assigned engineer (email) |
| `reporter` | STRING | Issue reporter (email) |
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
- Export frequency: every 4 hours (a known limitation — see [Next Steps](#next-steps) for improvement directions)
- Mode: `WRITE_TRUNCATE` for small tables; incremental `MERGE` for large tables
- Deduplication: `issue_id + updated_at`
- Partitioning: `issues` table by `created_at` DATE
- Clustering: `(priority, status, component)` — matches common query filter patterns

---

### Pillar 3 — Source code (Cloud Storage / .txt files)

Source files are exported from the code repository as `.txt` files via CI/CD and indexed by Vertex AI Search.

**Why .txt and not direct repo integration?**

Direct repository integration was the original plan. We moved away from it for the POC because: Vertex AI Search indexes unstructured text rather than code natively; real-time repo webhooks add significant middleware complexity; and `.txt` export via Cloud Build gives us control over exactly which files are indexed (no generated files, no binaries). This is a deliberate short-term compromise — see [Next Steps](#next-steps) for where this goes.

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

**What does not work well:** Cross-file reasoning, recently merged code (up to 24-hour lag), generated or minified files that slip past the exclusion filter.

---

## Agent Design — What We Built and Why

### The Orchestrator Agent

The Orchestrator is the entry point for all queries. It classifies intent and routes to the appropriate sub-agent. For cross-domain queries it calls multiple sub-agents in parallel and aggregates the responses.

**Routing table:**

| User intent | Target | Trigger signals |
|---|---|---|
| Document / policy | Doc agent | onboarding, architecture, SLA, PRD, design doc, spec, runbook |
| Issue / bug / feature | Issue agent | bug, issue, P0, P1, sprint, blockers, Buganizer, ticket, defect, feature request |
| Code / implementation | Code agent | function, class, module, file, implementation, source, code, API |
| Cross-domain | All agents (parallel) | status, summary, program health, tell me about |

**Learning from routing failures:**

Early versions frequently mis-routed "how does authentication work?" to the Code agent because of the word "how does", when the user actually wanted the architecture document. Adding explicit disambiguation examples to the system prompt significantly improved routing accuracy. The examples in the system prompt matter as much as the routing rules.

### The Issue Agent (Buganizer)

**Design constraints:**
- Must generate only `SELECT` statements — no DML under any circumstances
- Must include the generated SQL in its response for transparency
- Must handle empty result sets gracefully with helpful guidance on query refinement
- Must limit result sets to 50 rows by default with pagination support

**System prompt structure (simplified):**
```
You are an issue tracker specialist with read-only access to a BigQuery dataset
containing exported Buganizer issue data. Buganizer is Google's internal issue
tracking system equivalent to Jira or ServiceNow.

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

### The 30-second problem

Integrating with Google Chat looked straightforward: expose a webhook, receive a message, call the agent, return the response. In practice, Google Chat's 30-second synchronous response requirement creates a fundamental tension with AI agent response times. This section documents what we tried and what we learned.

### How the integration works (current state)

```
1.  User sends message in Google Chat space
2.  Chat delivers HTTP POST to Cloud Function webhook
3.  Cloud Function validates Google-signed JWT
4.  Cloud Function returns HTTP 200 + "Thinking..." IMMEDIATELY
    └── Critical: must happen within 30 seconds or Chat drops the connection silently
5.  Cloud Function spawns background thread to call Vertex AI agent
6.  Agent Engine routes to sub-agent(s), queries data stores
7.  Agent returns generated response
8.  Cloud Function calls Chat REST API to post the real response
```

### Async pattern (current implementation)

```python
@functions_framework.http
def handle_chat_event(request):
    payload = request.get_json()
    verify_google_chat_jwt(request)

    # Return 200 FIRST — do not wait for the agent
    send_typing_indicator(payload['space']['name'])

    # Fire background thread for agent work
    thread = threading.Thread(
        target=call_agent_and_respond,
        args=(payload,)
    )
    thread.start()
    return 'OK', 200


def call_agent_and_respond(payload):
    try:
        session_id = f"{payload['space']['name']}_{payload['user']['name']}"
        response = call_vertex_agent(payload['message']['text'], session_id)
        post_to_chat(payload['space']['name'], response)
    except Exception as e:
        # Always post something — silent failures are worse than error messages
        post_to_chat(payload['space']['name'],
                     f"Something went wrong: {str(e)}. Please try again.")
```

### Next: Cloud Tasks pattern (recommended improvement)

The background thread approach has a reliability problem: if the Cloud Function instance is recycled mid-execution, the thread dies silently. Cloud Tasks eliminates this:

```python
def handle_chat_event(request):
    payload = request.get_json()
    verify_google_chat_jwt(request)

    client = tasks_v2.CloudTasksClient()
    task = {
        'http_request': {
            'http_method': tasks_v2.HttpMethod.POST,
            'url': AGENT_WORKER_URL,
            'body': json.dumps(payload).encode(),
        }
    }
    client.create_task(parent=QUEUE_PATH, task=task)

    send_thinking_message(payload)   # immediate ack to Chat
    return 'Queued', 200
```

### Environment variables

| Variable | Description | Example |
|---|---|---|
| `VERTEX_PROJECT_ID` | GCP project hosting the agent | `my-project-123` |
| `VERTEX_LOCATION` | Agent region | `us-central1` |
| `AGENT_ID` | Vertex AI Agent Designer agent ID | `abc123def456` |
| `CHAT_SA_EMAIL` | Service account for Chat API calls | `chat-bot@project.iam.gserviceaccount.com` |
| `MAX_RESPONSE_WAIT_SECS` | Timeout for agent response | `45` |

---

## Active Issues — Still Learning

These are not bugs to fix before launch. They are the active learning surface of the POC — open questions about how to build AI agent systems well. Each one is documented with what was observed, what was learned, what was changed, and what remains to be done.

---

### Issue #1 — Inconsistent responses on Buganizer data

> **Status: Active** — The same question about issue data can produce different answers in different sessions, even when the underlying data has not changed.

**Observed:**
- P0 bug count returns 12 in one session, 11 in another with no data change
- Occasionally returns empty result for queries that should match hundreds of rows
- Component name misspellings cause total query failure with no helpful error message
- Multi-table JOIN queries are more prone to inconsistency than single-table queries

**Learned:**
The root cause is NL-to-SQL non-determinism. At `temperature > 0`, the LLM generating SQL from natural language produces structurally different queries for the same input. A query like "open P0s in Authentication" might generate `component = 'Authentication'` in one session and `component LIKE '%auth%'` in another — returning different result sets.

A secondary problem: the BigQuery schema is embedded in the agent system prompt as static text. When the schema evolves, the prompt is not updated automatically, and the LLM generates SQL referencing columns that no longer exist.

**Changed:**
- Set `temperature=0` for the Issue agent — the single highest-impact change
- Added explicit schema documentation with column descriptions and valid enum values to the system prompt
- Pre-built BigQuery Views for the 8 most common query patterns and added these as named tools in the agent

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| ✅ Done | `temperature=0` for Issue agent | Deployed |
| ✅ Done | Pre-built Views for common queries | Deployed |
| 🔧 Next | SQL validation layer in Cloud Function | Open |
| 🔧 Next | Automated schema sync via Data Catalog events | Open |
| 🔧 Next | Redis query result cache (15-min TTL) | Open |
| 🔭 If productionised | Constrained query builder with parameterised templates | Future |

---

### Issue #2 — No response on Google Chat (timeout)

> **Status: Active** — Users intermittently receive no response, especially for issue tracker queries involving complex BigQuery jobs.

**Observed:**
- User sends message, sees "Thinking...", waits, sees nothing
- Cloud Function logs confirm HTTP 200 was returned to Chat, but subsequent agent call timed out
- Most common on Buganizer queries with complex BigQuery jobs
- Worse after deployments due to cold starts

**Learned:**
Google Chat has a hard 30-second limit. Vertex AI agent calls involving BigQuery routinely take 15–45 seconds. The background thread pattern resolved most timeouts but introduced a new failure mode: threads that die when the Cloud Function instance is recycled. We discovered this by correlating "no response" user reports with instance recycling events in Cloud Logging.

A second failure mode: the Chat API POST delivering the real response can fail due to an expired OAuth token or rate limit — and these failures were not being logged. Users saw "Thinking..." with no follow-up because the response was computed but never delivered.

**Changed:**
- Error handling in background thread: always post to Chat on failure, never fail silently
- Cloud Function timeout increased from 60 to 540 seconds (the maximum)
- Structured logging added for all Chat API post failures

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| ✅ Done | Async pattern (HTTP 200 before agent call) | Deployed |
| ✅ Done | Error callback — always post on failure | Deployed |
| ✅ Done | Cloud Function timeout → 540 seconds | Deployed |
| ✅ Done | Structured logging for Chat API failures | Deployed |
| 🔧 Next | Cloud Tasks queue (replaces background thread) | Open |
| 🔧 Next | "Still working..." follow-up at 15-second mark | Open |
| 🔭 If productionised | Full Pub/Sub async architecture | Future |

---

### Issue #3 — No real-time sync with code repo and Buganizer

> **Status: Known limitation** — Buganizer data can be up to 4 hours stale. Code files can be up to 24 hours stale. Users are reporting answers about issues that have already been resolved.

**Observed:**
- Newly filed Buganizer issues do not appear in agent responses for up to 4 hours
- Code changes merged to `main` are not reflected in agent responses for up to 24 hours
- Agent says a bug is "Open / Assigned" when it was resolved the same morning
- Sprint velocity data is inaccurate between export runs

**Learned:**
This is a fundamental architectural constraint of the batch export design. The Buganizer export pipeline runs every 4 hours — too slow for active sprint management queries. The code snapshot export triggers on CI/CD completion but still faces Vertex AI Search re-index latency of 15–60 minutes.

The deeper issue: Vertex AI Search re-indexing is not instant. Even with an immediate file upload, propagation to the search index takes 15–60 minutes. Truly real-time code search is not achievable with the current data store architecture — this is a known POC constraint.

**Planned:**

Switching to incremental `MERGE` operations for Buganizer (processing only records changed since the last run):

```python
from google.cloud.bigquery_storage_v1 import BigQueryWriteClient
from google.cloud.bigquery_storage_v1.types import WriteStream

client = BigQueryWriteClient()
write_stream = client.create_write_stream(
    parent=table_path,
    write_stream=WriteStream(type_=WriteStream.Type.COMMITTED)
)
# Append individual issue updates as they arrive via Pub/Sub
```

Adding an Eventarc trigger on Cloud Storage object creation to initiate on-demand Vertex AI Search re-index of only changed files (target: ~15-minute propagation for code).

**Remaining:**

| Priority | Fix | Status |
|---|---|---|
| 🔧 Next | Incremental BigQuery MERGE (more frequent refresh) | Open |
| 🔧 Next | Eventarc trigger → on-demand Vertex AI re-index | Open |
| 🔭 If productionised | Buganizer Pub/Sub → BigQuery Storage Write API | Future |
| 🔭 If productionised | Direct Buganizer API (gRPC streaming) | Future |
| 🔭 If productionised | Direct Git connector (Cloud Source Repositories) | Future |

---

## Security & IAM

### Service accounts

| Service account | Roles | Used by |
|---|---|---|
| `vertex-agent-sa` | `aiplatform.user`, `discoveryengine.viewer`, `bigquery.dataViewer`, `bigquery.jobUser` | Vertex AI Agent Engine |
| `cloud-function-sa` | `run.invoker`, `chat.bot`, `aiplatform.user` | Cloud Function |
| `bq-export-sa` | `bigquery.dataEditor`, `storage.objectAdmin` | Dataflow / export pipeline |
| `ci-export-sa` | `storage.objectCreator` | Cloud Build code snapshot export |

### Controls in place

- Uniform bucket-level access on all Cloud Storage buckets (no legacy ACLs)
- BigQuery column-level security on PII fields (`assignee`, `reporter` emails)
- Google-signed JWT validation on the Cloud Function webhook
- Service account impersonation throughout — no long-lived credentials stored anywhere
- VPC Service Controls perimeter around Vertex AI, BigQuery, and Cloud Storage (production only)

### Still to do

- Audit log dashboard: Cloud Logging captures all data store access but no dashboard exists yet
- Document-level access control: currently any user in the Chat space can query any document in the bucket — no per-document ACL enforcement

---

## No-Code / Low-Code Context — Why This Matters Now

This POC sits at an interesting intersection: it was built by engineers, using tools that are themselves products of the no-code / low-code revolution. Understanding where that ecosystem is heading contextualises both what this system can do today and where the remaining rough edges will be smoothed out by the platforms themselves.

### The three waves

**Wave 1 — Automation & publishing (2012–2018)**
Zapier, IFTTT, Wix, Airtable. Simple connectors and content tools. The ceiling was low — any non-trivial logic required engineering handoff.

**Wave 2 — Visual app builders (2018–2022)**
Bubble, Retool, Appsmith, OutSystems, Mendix. Full-stack apps without backend code. Jira and ServiceNow both launched low-code app layers in this era, making issue data more accessible to non-engineering teams.

**Wave 3 — AI-native builders (2022–present)**

| Category | Platforms | What changed |
|---|---|---|
| AI agent builders | Vertex AI Agent Designer, Microsoft Copilot Studio | Configure multi-agent pipelines in natural language |
| App generators | v0, Lovable, bolt.new | Describe the interface → get working code |
| Embedded AI copilots | Power Platform, Google AppSheet | AI generates workflow logic inside the builder canvas |
| Enterprise AI layers | Jira AI (Atlassian Intelligence), ServiceNow Now Intelligence | Managed NL query over issue data — no custom agent needed |

### Where this POC fits

This system is a Wave 3 product built with Wave 3 tools. The agent configuration, data store connections, and routing logic were set up through Vertex AI Agent Designer's console — no ML code written. What required an ML engineering team in 2021 is now configuration work.

The remaining friction — NL-to-SQL inconsistency, re-index latency, the Chat timeout — maps precisely onto where the platforms are still maturing. Jira's Atlassian Intelligence and ServiceNow's Now Intelligence are building managed NL-to-SQL layers that will eventually make the BigQuery intermediary unnecessary for those platforms. For Buganizer, the same trajectory applies through Google's internal tooling investments.

**The design implication:** The architectural decisions in this system that feel like engineering compromises (batch exports, `.txt` code snapshots, threading-based async) are mostly temporary workarounds for platform gaps that are actively closing. Build for the platform to meet you — do not over-engineer permanent solutions to what the managed service will solve in 18 months.

### What no-code / low-code still cannot do well (mid-2025)

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

- Replace the background thread with a Cloud Tasks queue (fixes the Chat timeout reliability)
- Add a SQL validation layer and schema auto-sync (fixes Buganizer response inconsistency)
- Switch to incremental BigQuery MERGE and add an Eventarc re-index trigger (reduces data staleness)

None of these require changing the agent design or the data store structure — they are improvements to the plumbing.

### Data freshness and real-time sync

The batch export architecture is the biggest gap between "POC" and "production-useful". The path forward:

- **For Buganizer:** If a Pub/Sub change notification feed becomes available, use it to stream individual issue updates to BigQuery via the Storage Write API rather than batch exporting the full dataset. This collapses the 4-hour lag to near-real-time.
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

- **Meeting notes and decisions:** Recorded meeting transcripts or decision logs are a common source of "why did we do it this way?" questions that documents don't capture well.
- **Monitoring and SLOs:** Connecting to Cloud Monitoring or a metrics store would let the agent answer "is the service healthy right now?" alongside "what is the SLA supposed to be?".
- **On-call history:** Past incident timelines and post-mortems are highly valuable for new team members and are often trapped in docs no one reads.

### If this moves toward production

A few things that are deliberately out of scope for the POC but would matter at scale:

- **Per-document access control:** Right now, every user in the Chat space can query every document in the bucket. A production system needs to respect the same access controls as the source documents.
- **Audit logging dashboard:** Cloud Logging already captures all data access — it just needs a dashboard to make it reviewable.
- **Cost model:** Vertex AI Search, BigQuery, and Cloud Functions all have usage-based costs. At low POC volume these are negligible; at team-wide adoption they need to be monitored.
- **Multi-space / multi-program support:** The current design serves one program's data. A shared infrastructure model would let multiple programs each bring their own data stores and share the orchestration layer.

---

## How to Contribute to This POC

This is a learning document as much as a technical spec. Contributions in three forms are welcome:

**1. Code and configuration**
Fork, branch, and PR as usual. For agent system prompt changes, include before/after accuracy notes in the PR description. For data pipeline changes, include freshness impact estimates.

**2. Issue reports and learnings**
If you hit a new failure mode not documented here, open an issue with: what you asked, what you got, what you expected. The "What we learned" sections above came from exactly these reports.

**3. Document and schema updates**
Add documents to the program-docs bucket following the folder structure. Notify the team in `#knowledge-guide` so others know new content is searchable.

```bash
# Branch naming convention
git checkout -b feature/your-feature-name
git checkout -b fix/issue-description
git checkout -b learning/what-was-discovered
```

---

*This document is updated as the POC evolves. For questions, reach out to prachi.devgaonkar@gmail.com. Last significant revision: June 2026.*
