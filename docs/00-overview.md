# 00 â€” Overview & Architecture

## What This Lab Builds

**Code Assist** is an internal system that researches production errors for engineers. It is a **heterogeneous multi-agent system**: its specialist agents are written in different frameworks and run on different Google Cloud runtimes, yet they cooperate as one system by speaking the **Agent2Agent (A2A)** protocol.

---

## System Architecture Diagram

```
User / Gemini Enterprise
         â”‚
         â–¼
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚                    code_assistant  (root)                        â”‚
â”‚              ADK 2.0 Workflow  Â·  Agent Runtime                  â”‚
â”‚                                                                  â”‚
â”‚  START â†’ query_extractor â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”         â”‚
â”‚                          â”‚ fan-out (parallel)          â”‚         â”‚
â”‚                 â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”¼â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”         â”‚         â”‚
â”‚                 â–¼        â–¼        â–¼          â–¼         â”‚         â”‚
â”‚           github_  stack_  bq_    manual_  salesforce_ â”‚         â”‚
â”‚           agent   exchange agent  search   agent       â”‚         â”‚
â”‚                 â””â”€â”€â”€â”€â”€â”€â”€â”€â”´â”€â”€â”€â”€â”€â”€â”€â”€â”´â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜         â”‚         â”‚
â”‚                          â”‚ JoinNode (barrier)           â”‚         â”‚
â”‚                          â–¼                              â”‚         â”‚
â”‚                       merge â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–ºâ”‚         â”‚
â”‚                          â”‚                              â”‚         â”‚
â”‚                          â–¼                              â”‚         â”‚
â”‚                     synthesizer â†’ END                   â”‚         â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
         â”‚                â”‚           â”‚            â”‚
         â–¼                â–¼           â–¼            â–¼
   github_agent   stackexchange    bq-agent   salesforce_agent
   ADK + MCP       LangGraph       ADK+BQ      ADK + SF REST
   Agent Runtime   Cloud Run       GKE         Cloud Run
   (private)       (public)        (public)    (public)
```

---

## Agent Roster

| Agent | Framework | Runtime | Auth | Build method |
|-------|-----------|---------|------|-------------|
| `github_agent` | ADK + MCP | Agent Runtime | Google OAuth (ADC) | agents-cli |
| `stackexchange_agent` | LangGraph + a2a-sdk 1.x | Cloud Run | Public | gcloud run deploy |
| `bq-agent` | ADK + BigQuery | GKE (Autopilot) | Public LB | Antigravity + agents-cli |
| `manual_search` | ADK tools (in-process) | Inside root | n/a | Vertex AI Search + Developer Knowledge MCP |
| `salesforce_agent` | ADK + Salesforce REST | Cloud Run | Public | agents-cli |
| `code_assistant` | ADK 2.0 Workflow | Agent Runtime | Google OAuth (ADC) | agents-cli |

---

## The A2A Protocol â€” Key Concepts

### What is A2A?

Agent2Agent (A2A) is a protocol that gives every agent a **common contract**, regardless of framework:

1. **Agent Card** â€” a small JSON document at `/.well-known/agent-card.json` (or `/a2a/app/.well-known/agent-card.json`) describing the agent's skills and its endpoint.
2. **Transport** â€” JSON-RPC over HTTP. Every agent looks the same to a caller regardless of its internal framework.

### A2A Variants You'll Encounter

| Dimension | What it means | In this lab |
|-----------|--------------|-------------|
| Agent card | JSON at a well-known path | Every specialist publishes one |
| Transport | JSON-RPC over HTTP | All agents |
| Standard vs ADK extension | ADK agents can opt into richer event data | github, salesforce, bq-agent use `use_legacy=False` |
| Version (1.x vs 0.3) | a2a-sdk major versions differ in card shape | stackexchange serves 1.x with a 0.3-compatible card |
| Serving style | self-served vs scaffolded | stackexchange self-serves; others are scaffolded |

### The ADK A2A Extension

When an ADK agent calls another ADK agent, it uses `use_legacy=False` to enable the ADK A2A extension, which carries richer event data. When calling a non-ADK server (like the LangGraph stack exchange agent), it uses standard A2A.

---

## Authentication Matrix

| Target Runtime | Auth Mechanism | What the caller sends |
|---------------|---------------|----------------------|
| Agent Runtime (GitHub) | Google OAuth token | ADC access token (automatic) |
| Cloud Run â€” private | Google identity token | Audience = service URL, needs `roles/run.invoker` |
| Cloud Run â€” public | None | No token |
| GKE â€” public LoadBalancer | None | No token |

---

## Key Google Cloud Services Used

| Service | Purpose |
|---------|---------|
| Agent Runtime (Vertex AI) | Managed runtime for ADK agents |
| Cloud Run | Container runtime for LangGraph + Salesforce agents |
| GKE Autopilot | Kubernetes runtime for BQ agent |
| BigQuery | Vector search over past incidents |
| Vertex AI Memory Bank | Long-term, per-user memory |
| Vertex AI Search | Internal code manuals search |
| Developer Knowledge MCP | Official Google docs search |
| Skill Registry | Discover and load agent skills at runtime |
| Gemini Enterprise | User-facing web app for the agent |
| Artifact Registry | Container image storage |
| Cloud Build | Build container images |
| MCP (GitHub) | Remote GitHub tools |
