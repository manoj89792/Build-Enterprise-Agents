# Task 3 â€” Add a LangGraph Specialist over A2A (Stack Exchange â†’ Cloud Run)

## Overview

Deploy a specialist that is **not an ADK agent**. The Stack Exchange agent is a **LangGraph graph** wrapped in a small `a2a-sdk` server, deployed to Cloud Run. To the rest of the system it is just another A2A card.

**Why this matters**: Your organization can keep existing LangGraph services and plug them into an ADK orchestrator, as long as each one publishes an A2A card. No rewrites needed.

**Why Cloud Run?** Plain container + HTTP + managed HTTPS + scale-to-zero = perfect fit for a self-hosted A2A server.

---

## Step 1: Implement the LangGraph Graph (Code TODO)

Open `~/agents/stackexchange-agent/app/agent.py`.

Complete the TODO to add the search node and connect edges:

```python
from langgraph.graph import START, END, StateGraph

# ... (search function already implemented above the TODO)

# Build the graph
graph_builder = StateGraph(State)

# Add the search node
graph_builder.add_node("search", search)

# Connect edges: START â†’ search â†’ END
graph_builder.add_edge(START, "search")
graph_builder.add_edge("search", END)

graph = graph_builder.compile()
```

> ðŸ’¡ Note: The `search()` function runs a Stack Exchange search with **no LLM** â€” it's deterministic pure code. An "agent" node can be plain code!

---

## Step 2: Update the A2A Server (Code TODO)

Open `~/agents/stackexchange-agent/app/server.py`.

Update `supported_interfaces` to include both A2A protocol versions:

```python
from a2a.types import AgentInterface, ProtocolBinding

supported_interfaces = [
    AgentInterface(
        url=rpc_url,
        protocol_binding=ProtocolBinding.JSONRPC,
        protocol_version="1.0",
    ),
    AgentInterface(
        url=rpc_url,
        protocol_binding=ProtocolBinding.JSONRPC,
        protocol_version="0.3",
    ),
]
```

**Why dual-version?** ADK pins the `a2a-sdk` 0.3 client. This server uses `a2a-sdk` 1.x. Without the 0.3-compatible interface, the ADK root agent can't parse the card. This is the most common real-world A2A interop issue.

---

## Step 3: Set Environment Variables

```bash
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
gcloud config set project ${PROJECT_ID}
```

---

## Step 4: Build the Container Image

```bash
cd ~/agents/stackexchange-agent

gcloud builds submit \
  --tag ${REGION}-docker.pkg.dev/${PROJECT_ID}/agents/stackexchange-agent:latest
```

This uses Cloud Build to build and push the image to Artifact Registry. Takes ~2â€“3 minutes.

---

## Step 5: Deploy to Cloud Run

```bash
gcloud run deploy stackexchange-agent \
  --image ${REGION}-docker.pkg.dev/${PROJECT_ID}/agents/stackexchange-agent:latest \
  --region ${REGION} \
  --allow-unauthenticated \
  --port 8080
```

`--allow-unauthenticated`: Makes this a **public** service (no auth token needed). Matches a public search service pattern.

---

## Step 6: Set the Public URL Environment Variable

The agent card must advertise a reachable endpoint. Read the deployed URL and inject it:

```bash
export SE_URL=$(gcloud run services describe stackexchange-agent \
  --region ${REGION} \
  --format='value(status.url)')

gcloud run services update stackexchange-agent \
  --region ${REGION} \
  --set-env-vars PUBLIC_URL=${SE_URL}

echo ${SE_URL}
```

---

## Step 7: Test the Deployed Agent

### Set and persist the card URL

```bash
export STACKEXCHANGE_AGENT_URL="${SE_URL}/.well-known/agent-card.json"
echo "export STACKEXCHANGE_AGENT_URL='${STACKEXCHANGE_AGENT_URL}'" >> ~/.bashrc
```

### Fetch the agent card (no auth needed â€” public service)

```bash
curl -sS "${STACKEXCHANGE_AGENT_URL}" | python3 -m json.tool
```

**Verify the card includes**:
- The Stack Exchange skill description
- **Both** interface versions (1.0 and 0.3)
- The `url` field pointing to the Cloud Run service

---

## âœ… Task 3 Checkpoint

- [ ] `agent.py` graph completed: `START â†’ search â†’ END`
- [ ] `server.py` updated with dual-version `supported_interfaces`
- [ ] Container built and pushed to Artifact Registry
- [ ] Service deployed to Cloud Run (`--allow-unauthenticated`)
- [ ] `PUBLIC_URL` environment variable set on the service
- [ ] `STACKEXCHANGE_AGENT_URL` exported and saved to `~/.bashrc`
- [ ] Agent card shows both 1.0 and 0.3 interface versions

**Check progress**: Click **Check my progress** â†’ *Deploy the Stack Exchange agent to Cloud Run*.

---

## Architecture Note: Protocol Version Compatibility

```
ADK Root Agent (a2a-sdk 0.3 client)
        â”‚
        â”‚  reads agent card â”€â”€â–º stackexchange agent card
        â”‚                         â”œâ”€â”€ interface: v1.0 (native)
        â”‚                         â””â”€â”€ interface: v0.3 (compatibility layer)
        â”‚
        â””â”€â”€â–º understands v0.3 interface â”€â”€â–º sends JSON-RPC call â”€â”€â–º LangGraph handler
```

The server runs on `a2a-sdk 1.x` but advertises a `0.3`-compatible card shape, allowing the ADK 0.3 client to parse it successfully.
