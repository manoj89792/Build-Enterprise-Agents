# Task 7 â€” Extend the System with the Salesforce Agent

## Overview

Add a **Salesforce agent** that searches the organization's document library in Salesforce Files using 2-legged OAuth (app-only auth). Deploy it to Cloud Run and rewire the root to include it â€” demonstrating how A2A lets you grow the system without touching existing specialists.

---

## Part A: Salesforce Setup

### A.1 Create an External Client App

1. In Salesforce, click **Setup** (gear icon) â†’ **Setup**
2. Quick Find â†’ type `App Manager` â†’ click **App Manager**
3. Click **New External Client App**
4. Fill in:
   - **External Client App Name**: `L400 Lab`
   - **Contact Email**: your email
   - The **API name** fills in automatically
5. In the **API (Enable OAuth Settings)** section â†’ check **Enable OAuth**
6. **Callback URL**: `https://login.salesforce.com/services/oauth2/callback`
7. **Selected OAuth Scopes** â†’ add:
   - `Manage user data via APIs (api)`
   - `Perform requests at any time (refresh_token, offline_access)`
8. In **Flow Enablement** â†’ check **Enable Client Credentials Flow**
9. Click **Create**

### A.2 Enable Client Credentials Flow & Set Run-As User

1. Quick Find â†’ `External Client App Manager` â†’ click it
2. Find **L400 Lab** â†’ row dropdown â†’ **Edit Policies**
3. Expand **OAuth Policies** â†’ **OAuth Flows and External Client App Enhancements**
4. Check **Enable Client Credentials Flow**
5. **Run-As User**: enter your Salesforce username
6. **IP Relaxation**: set to `Relax IP restrictions`
7. Click **Save**

### A.3 Get Consumer Key and Secret

1. In App Manager, find L400 Lab â†’ **Settings** tab â†’ expand **OAuth Settings**
2. Under **App Settings** â†’ click **Consumer Key and Secret**
3. Enter the verification code from the Salesforce email
4. Copy:
   - **Consumer Key** â†’ this is your `SALESFORCE_CLIENT_ID`
   - **Consumer Secret** â†’ this is your `SALESFORCE_CLIENT_SECRET`

### A.4 Find Your My Domain URL

1. Quick Find â†’ `My Domain` â†’ click **My Domain**
2. From **Current My Domain URL**, copy the **host only**  
   Example: `orgfarm-f49dce3e51-dev-ed.develop.my.salesforce.com`
   - âŒ Do NOT include `https://`
   - âŒ Do NOT include trailing text after the domain
3. This is your `SALESFORCE_DOMAIN`

---

## Part B: Implement the Code TODOs

### B.1 Implement 2-Legged OAuth Authentication

Open `~/agents/salesforce-agent/app/salesforce_client.py` and complete the `authenticate()` method:

```python
import httpx

_TIMEOUT_SECONDS = 30

async def authenticate(self) -> str:
    """Perform 2-legged (app-only) OAuth authentication."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"https://{self.domain}/services/oauth2/token",
            data={
                "grant_type": "client_credentials",
                "client_id": self.client_id,
                "client_secret": self.client_secret,
            },
            timeout=_TIMEOUT_SECONDS,
        )
        response.raise_for_status()
        return response.json()["access_token"]
```

**Why 2-legged OAuth (app-only)?** The agent acts as the application, not a user. No user login redirect needed â€” ideal for server-to-server integration.

### B.2 Convert the A2A Request

Open `~/agents/salesforce-agent/app/app_utils/a2a.py` and complete the `_user_scoped_request_converter` function:

```python
from google.adk.a2a.converters.request_converter import convert_a2a_request_to_agent_run_request

def _user_scoped_request_converter(request, context):
    run_request = convert_a2a_request_to_agent_run_request(
        request,
        part_converter=part_converter,
    )
    return run_request
```

### B.3 Add Memory Callbacks to the Agent

Open `~/agents/salesforce-agent/app/agent.py` and complete the TODOs:

```python
def generate_memories_callback(callback_context):
    """Save the session to Memory Bank after each turn."""
    callback_context.add_session_to_memory()
    # No return value needed

# In root_agent definition:
root_agent = Agent(
    name="salesforce_agent",
    # ... other config ...
    after_agent_callback=generate_memories_callback,  # Save memory after each turn
    tools=[
        PreloadMemoryTool(),  # Load long-term memory before each turn
        # ... other tools ...
    ],
)
```

---

## Part C: Deploy the Salesforce Agent

### C.1 Set Environment Variables

```bash
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
gcloud config set project ${PROJECT_ID}
```

### C.2 Create the .env File

```bash
cd ~/agents/salesforce-agent

cat << EOF > .env
GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
SALESFORCE_DOMAIN=your-domain.develop.my.salesforce.com
SALESFORCE_CLIENT_ID=your_consumer_key
SALESFORCE_CLIENT_SECRET=your_consumer_secret
EOF
```

Open `~/agents/salesforce-agent/.env` and replace the placeholder values.

### C.3 Deploy to Cloud Run

```bash
agents-cli deploy \
  --project ${PROJECT_ID} \
  --region ${REGION} \
  --no-confirm-project \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

### C.4 Allow Public Access

```bash
gcloud run services add-iam-policy-binding salesforce-agent \
  --region=${REGION} \
  --member=allUsers \
  --role=roles/run.invoker
```

### C.5 Record the Agent Card URL

```bash
export SF_URL=$(gcloud run services describe salesforce-agent \
  --region ${REGION} \
  --format='value(status.url)')

export SALESFORCE_AGENT_URL="${SF_URL}/a2a/app/.well-known/agent-card.json"
echo "export SALESFORCE_AGENT_URL='${SALESFORCE_AGENT_URL}'" >> ~/.bashrc

# Verify
curl -sS "${SALESFORCE_AGENT_URL}" | python3 -m json.tool
```

### C.6 Test the Agent

```bash
cd ~/agents/salesforce-agent

agents-cli run \
  --url "${SF_URL}" \
  --mode a2a \
  "ADR-007"
```

**Expected**: The agent authenticates with Salesforce, searches Files, and returns document results.

---

## Part D: Rewire the Root Agent

### D.1 Update the Root .env

Open `~/agents/code-assistant/.env` and update:

```bash
SALESFORCE_AGENT_URL=YOUR_SALESFORCE_CARD_URL_HERE
```

Replace `added-in-task-7` with the actual card URL you recorded above.

### D.2 Redeploy the Root

```bash
cd ~/agents/code-assistant
rm -rf .venv

agents-cli deploy \
  --project ${PROJECT_ID} \
  --no-confirm-project \
  --no-wait \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

Monitor status:
```bash
agents-cli deploy --status
```

---

## âœ… Task 7 Checkpoint

- [ ] Salesforce External Client App created
- [ ] Client Credentials Flow enabled, Run-As user set, IP relaxed
- [ ] Consumer Key (Client ID) and Secret saved
- [ ] My Domain URL saved (host only, no `https://`)
- [ ] `salesforce_client.py` â†’ `authenticate()` completed with POST to `/services/oauth2/token`
- [ ] `a2a.py` â†’ `_user_scoped_request_converter` completed
- [ ] `agent.py` â†’ `generate_memories_callback` + `PreloadMemoryTool` added
- [ ] `.env` created with Salesforce credentials
- [ ] Deployed to Cloud Run
- [ ] Public access granted (`allUsers` â†’ `roles/run.invoker`)
- [ ] `SALESFORCE_AGENT_URL` exported and saved to `~/.bashrc`
- [ ] Agent card curl returns valid JSON
- [ ] Test query returns Salesforce document results
- [ ] Root `.env` updated with `SALESFORCE_AGENT_URL`
- [ ] Root redeployed successfully

**Check progress**: Click **Check my progress** â†’ *Deploy the Salesforce agent to Cloud Run*.

---

## Salesforce OAuth Flow Diagram

```
salesforce_agent
      â”‚
      â–¼ POST /services/oauth2/token
      â”‚   grant_type=client_credentials
      â”‚   client_id=CONSUMER_KEY
      â”‚   client_secret=CONSUMER_SECRET
      â”‚
      â–¼
Salesforce Auth Server
      â”‚
      â–¼ 200 OK { access_token: "..." }
      â”‚
      â–¼
salesforce_agent uses access_token for all API calls
```
