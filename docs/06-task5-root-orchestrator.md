# Task 5 â€” Orchestrate the Specialists in a Graph Workflow

## Overview

Build the brain of the system: the **root Code Assist agent** â€” an ADK 2.0 graph workflow that:
1. Extracts the query
2. Fans out to all specialists **in parallel**
3. Waits for all results at a `JoinNode`
4. Synthesizes one cited answer

Test the whole system **locally** before deploying.

---

## Step 1: Implement the State Capture Callback

**Purpose**: `RemoteA2aAgent` has no `output_key`, so its reply isn't automatically saved to session state. This callback copies the reply into state so the synthesizer can read it.

Open `~/agents/code-assistant/app/callbacks.py` and complete the TODO:

```python
def capture_response_to_state(state_key: str):
    """Returns an after_agent_callback that saves the agent's final reply to state."""
    
    def _after_agent_callback(callback_context):
        events = callback_context.session.events
        
        for event in reversed(events):
            # Find the most recent event from this agent with text content
            if (event.author == callback_context.agent_name and 
                event.content and 
                any(part.text for part in event.content.parts if hasattr(part, 'text'))):
                
                # Join all text parts
                text = "".join(
                    part.text 
                    for part in event.content.parts 
                    if hasattr(part, 'text') and part.text
                )
                
                # Save to shared state
                callback_context.state[state_key] = text
                return
    
    return _after_agent_callback
```

**Why needed?** A workflow node's output is captured from `event.output`, which `LlmAgent` sets but `RemoteA2aAgent` does not. Without this callback, the branch produces no output for the join.

---

## Step 2: Implement ADC Authentication

Open `~/agents/code-assistant/app/auth.py` and complete the `_apply` method in `GoogleADCAuth`:

```python
import google.auth
import google.auth.transport.requests
from google.auth.credentials import Credentials

class GoogleADCAuth(httpx.Auth):
    def __init__(self):
        self._creds, _ = google.auth.default(
            scopes=["https://www.googleapis.com/auth/cloud-platform"]
        )
    
    def _apply(self, request: httpx.Request) -> httpx.Request:
        """Attach ADC OAuth2 token to the request."""
        # Refresh credentials if needed and apply token to auth header
        self._creds.before_request(
            request=google.auth.transport.requests.Request(),
            method=request.method,
            url=str(request.url),
            headers=dict(request.headers),
        )
        # Apply the token to the httpx request headers
        request.headers["Authorization"] = f"Bearer {self._creds.token}"
        return request
    
    def auth_flow(self, request: httpx.Request):
        yield self._apply(request)
    
    async def async_auth_flow(self, request: httpx.Request):
        yield self._apply(request)


def google_authed_client() -> httpx.Client:
    return httpx.Client(auth=GoogleADCAuth())
```

**Why ADC?** Agent Runtime is always authenticated. Inside deployed agents, ADC automatically provides the runtime service account's token. Locally, it uses your user credentials. Same code works in both places.

---

## Step 3: Construct the Specialist Agents

Open `~/agents/code-assistant/app/agent.py` and complete the Remote specialists section:

### 3.1 GitHub Agent (ADK â†’ ADK, with ADK extension)

```python
from google.adk.agents.remote_a2a_agent import RemoteA2aAgent
from app.auth import google_authed_client
from app.callbacks import capture_response_to_state
import os

github_agent = RemoteA2aAgent(
    name="github_agent",
    description="Searches GitHub repositories, issues, and code for relevant information.",
    agent_card=os.environ["GITHUB_AGENT_URL"],
    httpx_client=google_authed_client(),  # ADC auth for Agent Runtime
    use_legacy=False,                      # Use ADK A2A extension
    after_agent_callback=capture_response_to_state("github_findings"),
)
```

### 3.2 Stack Exchange Agent (LangGraph, standard A2A, no ADK extension)

```python
# Note: Uses CleanQueryRemoteA2aAgent â€” a subclass that strips system prompts
#       before forwarding to the non-ADK LangGraph server
stackexchange_agent = CleanQueryRemoteA2aAgent(
    name="stackexchange_agent",
    description="Searches Stack Exchange for relevant Q&A threads.",
    agent_card=os.environ["STACKEXCHANGE_AGENT_URL"],
    # No httpx_client override â€” public service, no auth needed
    # use_legacy not set to False â€” standard A2A for non-ADK server
    after_agent_callback=capture_response_to_state("stackexchange_findings"),
)
```

**Note**: No `use_legacy=False` for the Stack Exchange node because it's a **non-ADK LangGraph server**.

---

## Step 4: Complete the Root Workflow Graph

In `~/agents/code-assistant/app/agent.py`, complete the `root_agent` Workflow graph:

```python
from google.adk.agents.workflow_agent import WorkflowAgent
from google.adk.agents.graph_nodes import JoinNode

root_agent = WorkflowAgent(
    name="code_assistant",
    description="Researches production errors across multiple knowledge sources.",
    # ... other config ...
)

# Build the graph
root_agent.graph.add_edge(START, "query_extractor")

# Fan out to all 5 specialists in parallel
root_agent.graph.add_edge("query_extractor", "github_agent")
root_agent.graph.add_edge("query_extractor", "stackexchange_agent")
root_agent.graph.add_edge("query_extractor", "bq_agent")
root_agent.graph.add_edge("query_extractor", "manual_search_agent")
root_agent.graph.add_edge("query_extractor", "salesforce_agent")

# All specialists flow into the JoinNode barrier
root_agent.graph.add_edge("github_agent", "merge")
root_agent.graph.add_edge("stackexchange_agent", "merge")
root_agent.graph.add_edge("bq_agent", "merge")
root_agent.graph.add_edge("manual_search_agent", "merge")
root_agent.graph.add_edge("salesforce_agent", "merge")

# After all specialists finish, synthesize
root_agent.graph.add_edge("merge", "synthesizer")
root_agent.graph.add_edge("synthesizer", END)
```

**Why `JoinNode`?** Specialists run concurrently and finish at different times. `JoinNode` is a synchronization barrier â€” it waits for **every** incoming branch before the next node runs. Without it, the synthesizer would fire after the fastest specialist, missing the others.

---

## Step 5: Configure the Root's .env

```bash
cd ~/agents/code-assistant

cat << EOF > .env
GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
DATASTORE_ID=code-manuals-datastore
DEVELOPER_KNOWLEDGE_MCP_URL=https://developerknowledge.googleapis.com/mcp
SKILL_REGISTRY_LOCATION=us-central1
GITHUB_AGENT_URL=${GITHUB_AGENT_URL}
STACKEXCHANGE_AGENT_URL=${STACKEXCHANGE_AGENT_URL}
SALESFORCE_AGENT_URL=added-in-task-7
BQ_AGENT_URL=${BQ_AGENT_URL}
EOF
```

> `SALESFORCE_AGENT_URL` is a placeholder â€” filled in Task 7.

### Verify all URLs are set

```bash
cd ~/agents/code-assistant

blank=$(grep -E '^(GITHUB_AGENT_URL|STACKEXCHANGE_AGENT_URL|BQ_AGENT_URL)=$' .env)
if [ -n "${blank}" ]; then
  echo "BLANK specialist URLs (re-export them and recreate .env):"
  echo "${blank}"
else
  echo "All specialist URLs are set."
fi
```

---

## Step 6: Publish the Developer Knowledge Skill

```bash
cd ~/agents/code-assistant

uv run python scripts/publish_skill.py \
  --project YOUR_PROJECT_ID \
  --location us-central1
```

**Expected**: `Published skill: using-developer-knowledge-mcp`

> If re-run: prints "skill already exists" and leaves it unchanged. Safe to re-run.

The skill is discovered at runtime by `manual_search_agent` using `search_skills` + `load_skill`.

---

## Step 7: Test the System Locally

### Install dependencies

```bash
agents-cli install
```

### Launch the playground

```bash
agents-cli playground
```

Open the playground URL from the terminal â†’ select **app**.

### Test queries

```
Our FastAPI service throws a 500 on startup after a dependency bump. What's the fix?
```

**Verify**:
- Answer cites **multiple** sources (GitHub + StackExchange + BQ + handbook)
- For Google-product errors, `manual_search_agent` trace shows `search_skills` â†’ `load_skill` â†’ Developer Knowledge tools â†’ official Google docs cited

### Stop the playground

```bash
CTRL+C
```

---

## âœ… Task 5 Checkpoint

- [ ] `callbacks.py` TODO completed â€” `capture_response_to_state` works
- [ ] `auth.py` TODO completed â€” `_apply` attaches ADC token
- [ ] `agent.py` specialists constructed (github + stackexchange)
- [ ] Root workflow graph has correct fan-out + JoinNode + fan-in
- [ ] `.env` created with all specialist URLs set
- [ ] URL validation check passes ("All specialist URLs are set.")
- [ ] Developer Knowledge skill published
- [ ] Local playground test returns multi-source cited answer

**Check progress**: Click **Check my progress** â†’ *Orchestrate the specialists in a graph workflow*.

---

## Graph Workflow: Visual Flow

```
START
  â”‚
  â–¼
query_extractor
  â”‚
  â”œâ”€â”€â–º github_agent â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
  â”‚         â””â”€â”€ capture_response_to_state("github_findings") â”‚
  â”‚                                                           â”‚
  â”œâ”€â”€â–º stackexchange_agent â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–ºâ”‚
  â”‚         â””â”€â”€ capture_response_to_state("stackexchange_findings")
  â”‚                                                           â”‚
  â”œâ”€â”€â–º bq_agent â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–ºâ”‚
  â”‚         â””â”€â”€ capture_response_to_state("bq_findings")     â”‚
  â”‚                                                           â”‚
  â”œâ”€â”€â–º manual_search_agent â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–ºâ”‚
  â”‚                                                           â”‚
  â””â”€â”€â–º salesforce_agent â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–ºâ”‚
                                                             â”‚
                                               â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
                                               â–¼
                                           merge (JoinNode â€” waits for ALL)
                                               â”‚
                                               â–¼
                                           synthesizer (reads all *_findings from state)
                                               â”‚
                                               â–¼
                                             END
```
