# 11 â€” Key Concepts Reference

## Agent2Agent (A2A) Protocol

### What is A2A?

A2A is a protocol that gives every agent a **common contract**, regardless of framework:

1. **Agent Card** â€” JSON at `/.well-known/agent-card.json` or `/a2a/app/.well-known/agent-card.json`
   - Describes the agent's name, skills, and JSON-RPC endpoint URL
   - A caller reads the card, then sends messages to the endpoint
   - The caller never needs to know the callee's framework

2. **Transport** â€” JSON-RPC over HTTP
   - Standard HTTP + JSON means an ADK agent, a LangGraph server, and a managed engine all look the same to a caller

### Agent Card Example

```json
{
  "name": "github_agent",
  "description": "Searches GitHub repositories and issues",
  "skills": [
    {
      "name": "search_github",
      "description": "Find repositories, issues, and code on GitHub"
    }
  ],
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["text/plain"],
  "interfaces": [
    {
      "url": "https://....aiplatform.googleapis.com/.../rpc",
      "protocolBinding": "JSONRPC",
      "protocolVersion": "0.3"
    }
  ]
}
```

### A2A Protocol Versions

| Version | Card shape | When used |
|---------|-----------|-----------|
| 0.3 | `url` field at top level | ADK client (current default) |
| 1.0 | `url` inside `interfaces[].url` | a2a-sdk 1.x default |

**Compatibility rule**: Serve both versions when your server is on `a2a-sdk 1.x` but callers may use ADK's 0.3 client.

---

## Agent Development Kit (ADK)

### Agent Types

| Type | Use case |
|------|----------|
| `LlmAgent` | Standard agent with LLM + tools |
| `RemoteA2aAgent` | Proxy to a remote A2A endpoint |
| `WorkflowAgent` | Graph-based workflow orchestrator |
| `SequentialAgent` | Runs sub-agents in sequence |

### ADK A2A Extension (`use_legacy=False`)

When an ADK agent calls another **ADK** agent, it can use `use_legacy=False` to enable the ADK A2A extension, which carries richer event data. Use the default (legacy mode) when calling **non-ADK** servers.

```python
# ADK â†’ ADK: use extension
RemoteA2aAgent(..., use_legacy=False)

# ADK â†’ LangGraph: standard A2A
RemoteA2aAgent(...)  # use_legacy defaults to True (standard)
```

### Callbacks

```python
# after_agent_callback: runs after every agent turn
def my_callback(callback_context):
    # callback_context.session â€” current session
    # callback_context.agent_name â€” current agent's name
    # callback_context.state â€” shared session state dict
    pass
```

---

## ADK 2.0 Graph Workflow

### Graph Nodes

| Node type | Purpose |
|-----------|---------|
| `START` | Entry point |
| `END` | Exit point |
| `LlmAgent` | LLM-powered node |
| `RemoteA2aAgent` | Remote A2A proxy node |
| `JoinNode` | Synchronization barrier â€” waits for ALL incoming edges |

### Fan-Out + Fan-In Pattern

```python
# Fan-out: one node â†’ multiple parallel nodes
graph.add_edge(START, "query_extractor")
graph.add_edge("query_extractor", "specialist_a")
graph.add_edge("query_extractor", "specialist_b")
graph.add_edge("query_extractor", "specialist_c")

# Fan-in: all parallel nodes â†’ JoinNode (waits for all)
graph.add_edge("specialist_a", "merge")
graph.add_edge("specialist_b", "merge")
graph.add_edge("specialist_c", "merge")
graph.add_edge("merge", "synthesizer")
graph.add_edge("synthesizer", END)
```

**Why JoinNode?** Without it, a normal node used as fan-in would fire once per branch â€” the synthesizer would run before slower specialists finish.

---

## Model Context Protocol (MCP)

MCP lets an agent use tools hosted by a separate server, over HTTP. The tools live on the server, so:
- No tool code to write, package, or maintain in the agent
- Agent behaves identically locally and deployed (tools are always remote)

```python
from google.adk.tools.mcp_tool.mcp_toolset import McpToolset, StreamableHTTPConnectionParams

toolset = McpToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="https://api.githubcopilot.com/mcp/",
        headers={"Authorization": f"Bearer {token}"}
    ),
    tool_filter=["list_repos", "search_code"]  # restrict to needed tools
)
```

---

## Authentication & Authorization

### The Two-Layer Model

1. **Authentication** â€” proves *who you are*  
   â†’ missing or invalid token = `401 Unauthorized`

2. **Authorization** â€” proves *what you may do*  
   â†’ valid token but no permission = `403 Permission Denied`

### Application Default Credentials (ADC)

ADC is the standard way Python client libraries authenticate on Google Cloud:

| Environment | ADC resolves to |
|-------------|----------------|
| Local dev / Cloud Shell | Your user credentials (`gcloud auth application-default login`) |
| Agent Runtime | Runtime service account (automatic) |
| GKE | Workload identity service account (automatic) |
| Cloud Run | Compute default service account (automatic) |

```bash
# Set ADC for local dev
gcloud auth application-default login
```

### Service Account Auth Flow (Agent Runtime)

```
Root Agent (Agent Runtime)
      â”‚
      â–¼  calls github_agent
GoogleADCAuth._apply()
      â”‚
      â–¼  ADC resolves runtime service account credentials
      â”‚   service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com
      â”‚
      â–¼  adds Authorization: Bearer <token> header
      â”‚
      â–¼  github_agent validates token + checks IAM
      â”‚   (needs roles/aiplatform.user on the project)
      â”‚
      â–¼  200 OK if authorized
```

---

## Vertex AI Memory Bank

### Key Concepts

| Term | Meaning |
|------|---------|
| Memory Bank | A managed service (backed by an Agent Engine) that stores user facts |
| Memory | A durable fact about a user extracted from their conversation |
| `PreloadMemoryTool` | Loads stored memories before each agent turn |
| `add_session_to_memory()` | Submits current session for async fact extraction |
| `app_name` | Namespace for memories (usually the agent name) |
| `user_id` | Identifies the user; memories are scoped per `(app_name, user_id)` |

### Memory vs Session State

```
Session state (short-term):
  callback_context.state["key"] = value  â† lost when session ends

Memory Bank (long-term):
  callback_context.add_session_to_memory()  â† persists across sessions
  PreloadMemoryTool()  â† loads facts at start of next session
```

---

## Vector Search in BigQuery

### How It Works

```sql
SELECT base.*,
       distance.distance AS cosine_distance
FROM VECTOR_SEARCH(
  TABLE code_assist_bugs.known_bugs,
  'description_embedding',
  (SELECT @query_vector),
  distance_type => 'COSINE',
  top_k => 3
)
ORDER BY cosine_distance ASC;
```

### Critical Rule

**Query vectors and stored vectors must use the same embedding model and dimension.**  
- Stored: `text-embedding-004` (768-dim, region `us-central1`)  
- Query: must also use `text-embedding-004` (768-dim) in `us-central1`

If models differ â†’ vectors live in different spaces â†’ distances are meaningless.

---

## LangGraph Integration with A2A

A LangGraph graph wrapped in an `a2a-sdk` server is a valid A2A specialist. The ADK root doesn't know it's LangGraph â€” it just reads the card and sends JSON-RPC.

```python
# LangGraph graph
from langgraph.graph import StateGraph, START, END

builder = StateGraph(State)
builder.add_node("search", search_function)
builder.add_edge(START, "search")
builder.add_edge("search", END)
graph = builder.compile()

# Wrapped in a2a-sdk server â†’ publishes agent card â†’ looks like any A2A agent
```

---

## agents-cli Deployment Modes

| Mode flag | When to use |
|-----------|------------|
| `--mode adk` | Calling an ADK agent on Agent Runtime |
| `--mode a2a` | Calling a standard A2A server (LangGraph, Cloud Run, GKE) |
| (no `--url`) | Run locally from source |
| `--url ENGINE_URL` | Target a specific deployed agent |

---

## Salesforce 2-Legged OAuth (Client Credentials)

```
                                POST /services/oauth2/token
                                grant_type=client_credentials
salesforce_agent â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–º Salesforce Auth
                                client_id=CONSUMER_KEY
                                client_secret=CONSUMER_SECRET
                                
                â—„â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€ { access_token }

salesforce_agent â”€â”€â”€â”€ GET /services/data/.../search?q=... â”€â”€â–º Salesforce API
                      Authorization: Bearer <access_token>
```

No user login required. The agent authenticates as the application itself.
