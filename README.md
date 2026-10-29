# ðŸ¤– Build Enterprise Agents â€” Lab Runbook & Step-by-Step Guide

> **APEX012 Â· Google Cloud Advanced Lab**  
> A complete heterogeneous multi-agent system built with ADK, LangGraph, BigQuery, GKE, Cloud Run, Agent Runtime, and Gemini Enterprise â€” all cooperating over the **Agent2Agent (A2A)** protocol.

---

## ðŸ“‹ Table of Contents

| # | Document | Description |
|---|----------|-------------|
| 1 | [Overview & Architecture](docs/00-overview.md) | System design, agent map, protocol concepts |
| 2 | [Prerequisites & Setup](docs/01-setup.md) | Environment bootstrap, ADC, agents-cli install |
| 3 | [Task 1 â€” Environment Setup](docs/02-task1-environment.md) | Cloud Shell, Vertex AI Search datastore |
| 4 | [Task 2 â€” GitHub Agent (ADK + MCP â†’ Agent Runtime)](docs/03-task2-github-agent.md) | Deploy first specialist |
| 5 | [Task 3 â€” Stack Exchange Agent (LangGraph â†’ Cloud Run)](docs/04-task3-stackexchange-agent.md) | A2A interop across frameworks |
| 6 | [Task 4 â€” BQ Agent (Antigravity generated â†’ GKE)](docs/05-task4-bq-agent.md) | Vibecoding with Antigravity |
| 7 | [Task 5 â€” Root Orchestrator (ADK 2.0 Workflow)](docs/06-task5-root-orchestrator.md) | Graph workflow, join nodes, local test |
| 8 | [Task 6 â€” Deploy Root + Gemini Enterprise](docs/07-task6-deploy-root.md) | IAM, Agent Runtime, enterprise registration |
| 9 | [Task 7 â€” Salesforce Agent (Cloud Run)](docs/08-task7-salesforce-agent.md) | 2-legged OAuth, extend live system |
| 10 | [Task 8 â€” Long-Term Memory (Vertex AI Memory Bank)](docs/09-task8-memory-bank.md) | Per-user persistent memory |
| 11 | [Troubleshooting Guide](docs/10-troubleshooting.md) | Common errors and fixes |
| 12 | [Key Concepts Reference](docs/11-concepts.md) | A2A, ADK, MCP, auth, graph workflow |

---

## ðŸ—ï¸ System Architecture

```
                        â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
                        â”‚          code_assistant (root)        â”‚
                        â”‚       ADK 2.0 Workflow Â· Agent Runtimeâ”‚
                        â”‚                                       â”‚
                        â”‚  query_extractor â†’ [fan-out] â†’ merge â†’ synthesizer
                        â””â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                           â”‚        â”‚        â”‚        â”‚
              â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜  â”Œâ”€â”€â”€â”€â”€â”˜  â”Œâ”€â”€â”€â”€â”˜  â”Œâ”€â”€â”€â”€â”˜
              â–¼               â–¼         â–¼        â–¼
     github_agent    stackexchange  bq_agent  manual_search  salesforce_agent
     ADK + MCP       LangGraph      ADK+BQ    local tools    ADK + SF REST
     Agent Runtime   Cloud Run      GKE       (in root)      Cloud Run
```

---

## ðŸš€ Quick Start

```bash
# 1. Clone this repo
git clone https://github.com/manoj89792/Build-Enterprise-Agents.git
cd Build-Enterprise-Agents

# 2. Follow the guides in order:
# docs/01-setup.md â†’ docs/02-task1-environment.md â†’ ... â†’ docs/09-task8-memory-bank.md
```

---

## ðŸ› ï¸ Agent Summary

| Agent | Framework | Runtime | Build method |
|-------|-----------|---------|-------------|
| `github_agent` | ADK + MCP | Agent Runtime | `agents-cli deploy` |
| `stackexchange_agent` | LangGraph + a2a-sdk | Cloud Run | `gcloud run deploy` |
| `bq-agent` | ADK + BigQuery | GKE | Antigravity generated + `agents-cli deploy` |
| `manual_search` | ADK tools (local) | In root | Vertex AI Search + Developer Knowledge MCP |
| `salesforce_agent` | ADK + Salesforce REST | Cloud Run | `agents-cli deploy` |
| `code_assistant` (root) | ADK 2.0 Workflow | Agent Runtime | `agents-cli deploy` |

---

## ðŸ“ Repository Structure

```
Build-Enterprise-Agents/
â”œâ”€â”€ README.md                    â† You are here
â”œâ”€â”€ docs/
â”‚   â”œâ”€â”€ 00-overview.md           â† Architecture & concepts
â”‚   â”œâ”€â”€ 01-setup.md              â† Prerequisites
â”‚   â”œâ”€â”€ 02-task1-environment.md  â† Task 1
â”‚   â”œâ”€â”€ 03-task2-github-agent.md â† Task 2
â”‚   â”œâ”€â”€ 04-task3-stackexchange-agent.md â† Task 3
â”‚   â”œâ”€â”€ 05-task4-bq-agent.md     â† Task 4
â”‚   â”œâ”€â”€ 06-task5-root-orchestrator.md   â† Task 5
â”‚   â”œâ”€â”€ 07-task6-deploy-root.md  â† Task 6
â”‚   â”œâ”€â”€ 08-task7-salesforce-agent.md    â† Task 7
â”‚   â”œâ”€â”€ 09-task8-memory-bank.md  â† Task 8
â”‚   â”œâ”€â”€ 10-troubleshooting.md    â† Troubleshooting
â”‚   â””â”€â”€ 11-concepts.md           â† Concepts reference
â””â”€â”€ runbook/
    â””â”€â”€ RUNBOOK.md               â† Operational runbook (single-page)
```

---

## â±ï¸ Estimated Completion Time

| Phase | Time |
|-------|------|
| Prerequisites (Salesforce signup) | 15â€“20 min (do before lab) |
| Task 1: Environment setup | 10 min |
| Task 2: GitHub agent | 15 min (+ 5â€“10 min deploy) |
| Task 3: Stack Exchange agent | 15 min (+ 5 min build) |
| Task 4: BQ agent (Antigravity) | 50â€“60 min |
| Task 5: Root orchestrator | 20 min |
| Task 6: Deploy root + register | 15 min (+ 10 min deploy) |
| Task 7: Salesforce agent | 20 min |
| Task 8: Memory Bank | 15 min |
| **Total** | **~3 hours** |

---

## ðŸ”‘ Credentials Checklist

Save these before starting the lab â€” they are reused in follow-on labs:

- [ ] GitHub PAT (`ghp_...`) â€” `repo` + `read:org` scopes
- [ ] Salesforce Consumer Key (Client ID)
- [ ] Salesforce Consumer Secret (Client Secret)
- [ ] Salesforce My Domain URL

---

## ðŸ“š Related Resources

- [Agent Development Kit (ADK)](https://google.github.io/adk-docs/)
- [agents-cli documentation](https://cloud.google.com/agent-platform)
- [A2A Protocol spec](https://google.github.io/A2A/)
- [Vertex AI Memory Bank](https://cloud.google.com/vertex-ai/docs/memory-bank)
- [LangGraph](https://langchain-ai.github.io/langgraph/)
