# Task 4 â€” Generate the BQ Agent with Antigravity and Deploy to GKE

## Overview

Generate the Bug Database agent using **vibecoding** â€” describe the agent's behavior in a precise prompt, and let the **Antigravity coding agent** produce a working ADK project. The agent runs semantic vector search over a BigQuery table of past incidents.

**Why generate it?** Writing a precise specification is now the skill â€” the AI produces the implementation.

**Why vector search?** Past incidents are matched by meaning, not keywords. Query and stored vectors must use the same embedding model (`text-embedding-004`, 768 dimensions) or distances are meaningless.

**Why GKE?** Demonstrates a third runtime; the root still reaches it as another A2A card.

---

## Step 1: Install Lab Tooling

### Install agents-cli skills for Antigravity

```bash
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
gcloud config set project ${PROJECT_ID}

cd ~/agents

# Install agents-cli skills so Antigravity knows how to scaffold and deploy
agents-cli setup --agent all
```

### Install Terraform (required for GKE deployment)

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com \
  $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install -y terraform
terraform -version
```

---

## Step 2: Launch Antigravity

```bash
cd ~/agents
agy
```

When prompted, choose **Google Cloud Project** as the login method, with location **global**, and authenticate.

---

## Step 3: Paste the Generation Prompt

Paste the following prompt into the Antigravity chat interface:

```
# Build a Bug Database agent (ADK + BigQuery semantic search, A2A, on GKE)

Build a standalone agent named **`bq-agent`**.
You have the **agents-cli** skills; use them to design, build, and deploy it.
This is a description of the product and its capabilities, not a set of steps â€” decide the implementation yourself.

## What it is
A self-contained specialist agent that answers one question well:
*"Have we seen this software problem before, and how was it fixed?"*
It performs **semantic (meaning-based) search** over an internal database of past engineering incidents in **BigQuery** and returns the most similar incidents with their resolutions.

- Framework: **ADK** (Python), built with agents-cli.
- Model: **`gemini-3.5-flash`** (read from config; don't hardcode).
- Interface: **A2A** â€” a peer service with no knowledge of any specific caller.
- Runtime: **GKE**.
- Access: **read-only** over BigQuery. Never mutates incident data.

## The tool: `find_similar_bugs`
One tool provides the entire competency.

- **Signature:** `find_similar_bugs(query: str) -> str`
- **Input:** a free-text problem description (exception, key stack-trace terms, language/framework, symptom).
- **Retrieval:**
  1. Embed `query` with the same embedding model used for the stored vectors (`text-embedding-004`), in the table's region.
  2. Run BigQuery `VECTOR_SEARCH` over `description_embedding` with `distance_type => 'COSINE'`, `top_k => 3`, passing the query vector as a `FLOAT64` array parameter. Order results by ascending distance.
- **Output:** for each match â€” `bug_id`, `title`, `description`, `resolution`, and the cosine distance. Return a clear "no similar bugs found" when the table yields nothing, and a graceful degradation message if embedding or BigQuery is temporarily unavailable (do not raise).

## The knowledge base (BigQuery)
The table is **pre-populated** for you. Do **not** create, seed, or write to it â€” query it read-only.

**Table `code_assist_bugs.known_bugs` (region `us-central1`):**
| column | type | notes |
|--------|------|-------|
| `bug_id` | STRING | cite it in answers (e.g. `BUG-1001`) |
| `title` | STRING | short incident title |
| `description` | STRING | problem statement that was embedded |
| `resolution` | STRING | the concrete fix |
| `description_embedding` | `ARRAY<FLOAT64>` | 768-dim text-embedding-004 vector |

- **Vector search:** cosine distance; a vector index is optional at this scale.
- **Embedding compatibility:** stored vectors use `text-embedding-004` (768-dim); embed queries with the same model in `us-central1`.

## Behavior
- On a reported problem, search and answer with matching incidents and fixes, **citing each `bug_id`** used.
- **Grounded:** summarize only what the search returns. Never invent incidents or resolutions.

## A2A interface
- Expose over A2A with a **single skill** (semantic search of the internal bug database).
- Representative example query: *"FastAPI returns a 500 on startup after a dependency bump."*
- The agent card must advertise a **reachable** address.

## Runtime, credentials, deployment
- **Target:** GKE.
- **Credentials:** Application Default Credentials â€” developer identity locally, workload service identity when deployed (no key files). Service account named `bq-agent-app` needs `roles/bigquery.jobUser` and `roles/bigquery.dataViewer`.
- **Reachability:** external GKE LoadBalancer (not internal); agent card must advertise the public address.

## Definition of done
- Built with agents-cli on ADK, exposing A2A; deployable to GKE with a reachable A2A endpoint.
- Queries run against the pre-populated table; query embeddings use the same model and dimension as stored vectors.
- A matching problem returns that incident with its `resolution` and `bug_id`; unrelated queries are not fabricated.
- Project tests pass.
```

> â±ï¸ **This takes ~40â€“50 minutes.** Antigravity generates the project, runs tests, and deploys.

### Tips for Antigravity command approvals

When Antigravity asks permission before each shell command, select **"Always allow this command and persist to settings.json"** to avoid approving each one individually.

---

## Step 4: Review Generated Code

After generation, review the key files:

- `~/agents/bq-agent/app/agent.py` â€” ADK agent definition
- `~/agents/bq-agent/app/tools.py` â€” `find_similar_bugs()` tool implementation

**Verify in `tools.py`:**
- Uses `text-embedding-004` for query embedding
- Uses BigQuery `VECTOR_SEARCH` with `distance_type => 'COSINE'`, `top_k => 3`
- Returns `bug_id`, `title`, `description`, `resolution`, `distance`
- Handles empty results and errors gracefully (no raises)

---

## Step 5: Deploy to GKE

Antigravity will ask to run the deployment â€” approve it. Or run manually:

```bash
cd ~/agents/bq-agent
agents-cli deploy --project ${PROJECT_ID} --no-confirm-project
```

**GKE Autopilot cluster creation takes ~8â€“10 minutes.** Total deploy time: 15â€“20 minutes.

### If deploy fails right after cluster creation (Unauthorized on kubernetes_* resource)

```bash
# Just re-run â€” the cluster now exists and credentials resolve on retry
agents-cli deploy --project ${PROJECT_ID} --no-confirm-project
```

---

## Step 6: Grant BigQuery Permissions

If the agent can't query BigQuery (permission error), grant the roles manually:

```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:bq-agent-app@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:bq-agent-app@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataViewer"
```

---

## Step 7: Record the Agent Card URL

```bash
# Replace with the external IP from the deploy output
export BQ_AGENT_URL="http://EXTERNAL_IP:8080/a2a/app/.well-known/agent-card.json"
echo "export BQ_AGENT_URL='${BQ_AGENT_URL}'" >> ~/.bashrc

# Fetch the card to verify
curl -sS "${BQ_AGENT_URL}" | python3 -m json.tool
```

**Format**: `http://<EXTERNAL_IP>:8080/a2a/app/.well-known/agent-card.json`

---

## Step 8: Test the Deployed Agent

```bash
cd ~/agents/bq-agent

# Derive the base URL (strip the card path)
BQ_BASE_URL=$(echo "${BQ_AGENT_URL}" | sed -E 's#(://[^/]+).*#\1#')
echo "${BQ_BASE_URL}"

# Send a test message (public GKE endpoint, use --mode a2a)
agents-cli run \
  --url "${BQ_BASE_URL}" \
  --mode a2a \
  "Have we seen a memory leak in a background worker before?"
```

**Expected**: The agent finds similar incidents from the BigQuery table and returns them with `BUG-XXXX` citations and resolutions.

---

## âœ… Task 4 Checkpoint

- [ ] `agents-cli setup --agent all` completed
- [ ] Terraform installed
- [ ] Antigravity generated `~/agents/bq-agent/` project
- [ ] Code reviewed: `tools.py` uses `text-embedding-004` + `VECTOR_SEARCH`
- [ ] GKE cluster deployed (Autopilot)
- [ ] `bq-agent-app` service account has `roles/bigquery.jobUser` + `roles/bigquery.dataViewer`
- [ ] `BQ_AGENT_URL` exported and saved to `~/.bashrc`
- [ ] Agent card curl returns valid JSON
- [ ] Test query returns incident results with bug IDs

**Check progress**: Click **Check my progress** â†’ *Deploy the Bug DB agent to GKE*.

> **Note**: If the check fails despite successful deployment, verify:
> `bq-agent-app@PROJECT_ID.iam.gserviceaccount.com` exists and has `roles/iam.workloadIdentityUser` bound to the GKE Kubernetes Service Account.

---

## BigQuery Vector Search: How it Works

```
User query (text)
      â”‚
      â–¼
text-embedding-004 (768-dim vector)
      â”‚
      â–¼
BigQuery VECTOR_SEARCH(
  base_table => code_assist_bugs.known_bugs,
  column_to_search => description_embedding,
  query => query_vector,
  distance_type => 'COSINE',
  top_k => 3
)
      â”‚
      â–¼
Nearest 3 incidents by cosine distance â†’ cited answer
```
