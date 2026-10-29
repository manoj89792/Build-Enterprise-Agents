# Task 8 â€” Give the Salesforce Agent Long-Term Memory (Vertex AI Memory Bank)

## Overview

Add **long-term, per-user memory** to the Salesforce agent using **Vertex AI Memory Bank** â€” a managed service on Agent Runtime. After each conversation, it extracts durable facts about the user and stores them. On the next conversation, it loads those facts so the agent personalizes its answers.

---

## Types of Memory

| Memory type | Scope | Duration | Managed by |
|-------------|-------|----------|-----------|
| **Short-term** (session) | Current conversation | Until session ends | ADK session state |
| **Long-term** (Memory Bank) | Per user, per app | Persistent across sessions | Vertex AI Memory Bank |

---

## How Memory Works in This System

```
User starts conversation
        â”‚
        â–¼
PreloadMemoryTool() â”€â”€â–º Memory Bank â”€â”€â–º load user facts into context
        â”‚
        â–¼
Agent uses facts to personalize answer
        â”‚
        â–¼
generate_memories_callback() â”€â”€â–º add_session_to_memory()
        â”‚                              â”‚
        â”‚                         Memory Bank extracts new
        â”‚                         durable facts (async)
        â–¼
Session ends â€” facts persist for next conversation
```

---

## Step 1: Implement the Memory Bank Creator (Code TODO)

Open `~/agents/salesforce-agent/scripts/create_memory_bank.py` and complete the `main()` function:

```python
import vertexai
from vertexai.preview import reasoning_engines

DISPLAY_NAME = "salesforce-agent-memory"

def main():
    # List existing agent engines (Memory Banks are agent engines)
    existing = list(reasoning_engines.ReasoningEngine.list())
    
    # Check if one already exists with our display name (idempotent)
    for engine in existing:
        if engine.display_name == DISPLAY_NAME:
            print(f"Found existing memory bank: {DISPLAY_NAME}")
            print(f"Resource name: {engine.resource_name}")
            return
    
    # Create a new Memory Bank
    print(f"Creating memory bank: {DISPLAY_NAME}")
    memory_bank = reasoning_engines.MemoryBank.create(
        display_name=DISPLAY_NAME,
    )
    print(f"Created: {DISPLAY_NAME}")
    print(f"Resource name: {memory_bank.resource_name}")
```

**Why idempotent?** Running the script multiple times should not create duplicate memory banks.

---

## Step 2: Implement User ID Forwarding in Root Agent (Code TODO)

Open `~/agents/code-assistant/app/app_utils/a2a.py` and complete the `user_id_meta_provider` function:

```python
def user_id_meta_provider(ctx) -> dict:
    """Forward the caller's real user ID to the specialist over A2A metadata."""
    return {'user_id': ctx.user_id}
```

**Why this matters**: Memory is stored per `(app_name, user_id)`. Without forwarding the real user ID, the Salesforce agent uses a per-conversation fallback ID (`A2A_USER_{context_id}`) and forgets the user at the end of each conversation.

---

## Step 3: Review How Memory Is Scoped

### In `code-assistant/app/app_utils/a2a.py`

The `user_id_meta_provider` function attaches `user_id` to every A2A call to the Salesforce agent.

### In `salesforce-agent/app/app_utils/a2a.py`

The `_user_scoped_request_converter` reads the forwarded `user_id`:

```python
# The A2A metadata includes {'user_id': 'real.user@domain.com'}
# This ensures Memory Bank stores facts under the real user identity
```

### In `salesforce-agent/app/app_utils/services.py`

```python
# Memory service is VertexAiMemoryBankService when MEMORY_BANK_ID is set
# Falls back to in-memory service when not set (no persistence)
if os.environ.get("MEMORY_BANK_ID"):
    memory_service = VertexAiMemoryBankService(memory_bank_id=MEMORY_BANK_ID)
else:
    memory_service = InMemoryMemoryService()  # stateless â€” no cross-session recall
```

---

## Step 4: Provision the Memory Bank

```bash
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
gcloud config set project ${PROJECT_ID}

cd ~/agents/salesforce-agent

# Create the Memory Bank (uses ADC + us-central1 endpoint)
GOOGLE_CLOUD_PROJECT=${PROJECT_ID} \
MEMORY_BANK_LOCATION=us-central1 \
uv run python scripts/create_memory_bank.py
```

**Expected output:**
```
Creating memory bank: salesforce-agent-memory
Created: salesforce-agent-memory
Resource name: projects/123456/locations/us-central1/reasoningEngines/987654321
```

Copy the **MEMORY_BANK_ID** â€” the numeric suffix of the resource name (e.g. `987654321`).

---

## Step 5: Connect the Agent to the Memory Bank

Open `~/agents/salesforce-agent/.env` and add two lines:

```bash
MEMORY_BANK_ID=987654321        # â† replace with your actual ID
MEMORY_BANK_LOCATION=us-central1
```

> âš ï¸ Keep `MEMORY_BANK_LOCATION=us-central1` even though other services use `global`. Memory Bank uses a **regional** endpoint; the global endpoint does not serve it.

---

## Step 6: Redeploy the Salesforce Agent

```bash
cd ~/agents/salesforce-agent

agents-cli deploy \
  --project ${PROJECT_ID} \
  --region ${REGION} \
  --no-confirm-project \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

### Re-grant public access (redeploying resets IAM)

```bash
gcloud run services add-iam-policy-binding salesforce-agent \
  --region=${REGION} \
  --member=allUsers \
  --role=roles/run.invoker
```

---

## Step 7: Redeploy the Root Agent

```bash
cd ~/agents/code-assistant
rm -rf .venv

agents-cli deploy \
  --project ${PROJECT_ID} \
  --no-confirm-project \
  --no-wait \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

---

## Step 8: Test Long-Term Memory

### Conversation 1 â€” State a durable fact

In Gemini Enterprise, start a **new conversation** with Code Assist and say:

```
I'm on the Payments team and I only care about Spanner documents.
```

Wait **~1 minute** for Memory Bank to process and store the fact asynchronously.

### Conversation 2 â€” Verify recall

Start a **new conversation** (different session) and ask a related question **without repeating your preference**:

```
What documents should I look at for our database issues?
```

**Expected**: The agent references your Payments team context and Spanner preference without being told again.

---

## Step 9: View Stored Memories in the Console

1. In the Google Cloud Console, go to **Agent Platform** â†’ **Agents** â†’ **Memory Bank**
2. Open the `salesforce-agent-memory` engine
3. Go to the **Memories** tab
4. Confirm your fact appears, scoped to your user ID (your lab email)

Or navigate directly:
```
https://console.cloud.google.com/agent-platform/runtimes/locations/us-central1/agent-engines/YOUR_MEMORY_BANK_ID/memories?project=YOUR_PROJECT_ID
```

---

## âœ… Task 8 Checkpoint

- [ ] `create_memory_bank.py` TODO completed (idempotent create/find logic)
- [ ] `code-assistant/app/app_utils/a2a.py` TODO completed (user_id forwarding)
- [ ] Memory Bank provisioned (`create_memory_bank.py` ran successfully)
- [ ] `MEMORY_BANK_ID` and `MEMORY_BANK_LOCATION=us-central1` added to Salesforce agent `.env`
- [ ] Salesforce agent redeployed with Memory Bank config
- [ ] Public access re-granted after redeploy
- [ ] Root agent redeployed to pick up user_id forwarding
- [ ] Conversation 1: stated a durable fact
- [ ] Waited ~1 minute for async memory generation
- [ ] Conversation 2: agent recalled the fact without being told
- [ ] Memory visible in Cloud Console under the Memory Bank engine

**Check progress**: Click **Check my progress** â†’ *Create a Vertex AI Memory Bank for the Salesforce agent*.

---

## Memory Design Principles

| Principle | Detail |
|-----------|--------|
| **Facts, not queries** | Memory stores durable facts about you (team, role, preferences). A one-off question stores nothing. |
| **Async generation** | Memory is written in the background after the turn. New facts take ~1 minute to appear. |
| **Consolidation** | The model de-duplicates and consolidates facts. Repeating the same thing doesn't create duplicates. |
| **Per-user scoping** | Facts are stored under `(app_name, user_id)`. Memory from one user never leaks to another. |
| **Regional endpoint** | Memory Bank uses `us-central1`, not `global`. Always set `MEMORY_BANK_LOCATION=us-central1`. |
