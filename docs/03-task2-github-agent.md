# Task 2 â€” Deploy the GitHub Agent (ADK + MCP â†’ Agent Runtime)

## Overview

Deploy the first specialist: the **GitHub agent** â€” an ADK agent whose only tool is a remote GitHub MCP server. It is deployed to **Agent Runtime**, Google's fully managed ADK runtime.

**Why Agent Runtime?** It builds the container, serves the A2A card and JSON-RPC endpoint, and manages sessions â€” fastest path from ADK code to a running agent.

**Why MCP?** Tools are hosted on GitHub's remote MCP endpoint. No tool code to write or package; the agent behaves identically locally and deployed.

---

## Step 1: Create a GitHub Personal Access Token (if not done yet)

1. Go to [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. Note: `code-assist-lab`
4. Scopes: âœ… `repo` + âœ… `read:org`
5. **Generate token** â†’ copy `ghp_...`

> âš ï¸ Copy immediately â€” shown only once!

---

## Step 2: Configure the Agent

```bash
cd ~/agents/github-agent

cat << EOF > .env
GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
GITHUB_PERSONAL_ACCESS_TOKEN=ghp_YOUR_TOKEN_HERE
GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
EOF
```

> ðŸ’¡ `.env` is a hidden file. In Cloud Shell Editor: **View > Toggle Hidden Files** to see it.

Open `~/agents/github-agent/.env` and replace `ghp_YOUR_TOKEN_HERE` with your actual token.

---

## Step 3: Implement the MCP Toolset (Code TODO)

Open `~/agents/github-agent/app/agent.py`.

Complete the TODO to assign `github_mcp_toolset`:

```python
from google.adk.tools.mcp_tool.mcp_toolset import McpToolset, StreamableHTTPConnectionParams
import os

github_mcp_toolset = McpToolset(
    connection_params=StreamableHTTPConnectionParams(
        url=os.environ["GITHUB_MCP_URL"],
        headers={
            "Authorization": f"Bearer {os.environ['GITHUB_TOKEN']}"
        }
    ),
    tool_filter=GITHUB_TOOL_FILTER  # restrict to the pre-defined allowed tools
)
```

**Why `tool_filter`?** Narrows the agent's capabilities to the minimum it needs â€” improves safety, latency, and tool-selection accuracy.

---

## Step 4: Set Environment Variables

```bash
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
gcloud config set project ${PROJECT_ID}
```

---

## Step 5: Deploy to Agent Runtime

```bash
cd ~/agents/github-agent

agents-cli deploy \
  --project ${PROJECT_ID} \
  --no-confirm-project \
  --no-wait \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

**Note**: `GOOGLE_CLOUD_PROJECT` is excluded because `--project` already sets it.

**Expected**: Deploy job queued. Agent Runtime builds the container and provisions the endpoint.

### Monitor deployment status

```bash
# Poll until "success" (takes 5â€“10 minutes)
agents-cli deploy --status
```

**Success output looks like:**
```
Status: ACTIVE
Agent Card URL: https://us-central1-aiplatform.googleapis.com/.../.well-known/agent-card.json
```

---

## Step 6: Record the Agent Card URL

```bash
# Replace with your actual URL from the deploy --status output
export GITHUB_AGENT_URL="https://us-central1-aiplatform.googleapis.com/v1/projects/YOUR_PROJECT/locations/global/reasoningEngines/v1/YOUR_ENGINE_ID/a2a/app/.well-known/agent-card.json"

# Persist to .bashrc
echo "export GITHUB_AGENT_URL='${GITHUB_AGENT_URL}'" >> ~/.bashrc
```

---

## Step 7: Test the Deployed Agent

### Fetch the Agent Card

```bash
curl -sS \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "${GITHUB_AGENT_URL}" | python3 -m json.tool
```

**Expected**: JSON with `name`, `description`, `skills`, and `url` fields.

### Derive the engine URL

```bash
export GITHUB_ENGINE_URL=$(echo "${GITHUB_AGENT_URL}" | sed -E 's#/reasoningEngines/v1/#/v1/#; s#/api/.*##')
echo "${GITHUB_ENGINE_URL}"
```

### Send a test message

```bash
agents-cli run \
  --url "${GITHUB_ENGINE_URL}" \
  --mode adk \
  "Find the google/adk-python repository"
```

**Expected**: The agent finds the repository and returns its details.

---

## âœ… Task 2 Checkpoint

- [ ] `.env` configured with your GitHub PAT
- [ ] `agent.py` TODO completed with `McpToolset`
- [ ] Deploy completed successfully (`agents-cli deploy --status` shows `ACTIVE`)
- [ ] `GITHUB_AGENT_URL` exported and saved to `~/.bashrc`
- [ ] Agent card curl returns valid JSON
- [ ] `agents-cli run` returns a GitHub repository result

**Check progress**: Click **Check my progress** â†’ *Deploy the GitHub agent to Agent Runtime*.

---

## Auth Troubleshooting

| Error | Fix |
|-------|-----|
| `service account info is missing 'email' field` | Run `gcloud auth application-default login` |
| `gcloud reauthentication` / no active account | Run `gcloud auth login` |
| Persistent failures | Click **Restart** in Cloud Shell toolbar, re-run exports |

---

## Key Concepts

**Agent Runtime auto-enables A2A**: `is_a2a: true` in `agents-cli-manifest.yaml` â€” no extra flag needed.

**`--mode adk`**: Tells agents-cli to use the ADK A2A extension (richer event data) when calling this ADK-hosted agent.

**No local run needed**: The deployed agent is always authenticated; you test it with a bearer token.
