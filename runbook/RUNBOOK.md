# RUNBOOK â€” Build Enterprise Agents (APEX012)

> **Single-page operational reference for running the entire lab from start to finish.**  
> For detailed explanations, see the individual task docs in `../docs/`.

---

## ðŸ”‘ Credentials to Collect Before Starting

```
GITHUB_PERSONAL_ACCESS_TOKEN = ghp_...
SALESFORCE_CLIENT_ID         = (Consumer Key)
SALESFORCE_CLIENT_SECRET     = (Consumer Secret)
SALESFORCE_DOMAIN            = org.develop.my.salesforce.com
```

---

## ðŸŒ Environment Baseline

```bash
# Run at the start of every Cloud Shell session
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
export PATH="${HOME}/.local/bin:${PATH}"
source ~/.bashrc
gcloud config set project ${PROJECT_ID}
```

---

## TASK 1 â€” Environment Setup

```bash
# Auth
gcloud auth login
gcloud auth application-default login

# Download code
gcloud storage cp -r gs://${PROJECT_ID}-bucket/agents .
mkdir ~/.vscode
gcloud storage cp -r gs://${PROJECT_ID}-bucket/settings.json ~/.vscode/

# Install agents-cli
uv tool install google-agents-cli
agents-cli info

# Open editor
cloudshell workspace ~

# Create Vertex AI Search datastore
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores?dataStoreId=code-manuals-datastore" \
  -d '{"displayName":"Code manuals","industryVertical":"GENERIC","solutionTypes":["SOLUTION_TYPE_SEARCH"],"contentConfig":"CONTENT_REQUIRED"}'

# Import documents
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/code-manuals-datastore/branches/0/documents:import" \
  -d '{"reconciliationMode":"INCREMENTAL","gcsSource":{"inputUris":["gs://'${PROJECT_ID}'-bucket/code_manuals/*"],"dataSchema":"content"}}'
```

**âœ… Check**: Click "Check my progress" â†’ *Create the code-manuals datastore*

---

## TASK 2 â€” GitHub Agent

```bash
# 1. Configure
cd ~/agents/github-agent
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
GITHUB_PERSONAL_ACCESS_TOKEN=ghp_YOUR_TOKEN
GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
EOF
# Edit .env and replace ghp_YOUR_TOKEN

# 2. Implement agent.py TODO: McpToolset with StreamableHTTPConnectionParams

# 3. Deploy
agents-cli deploy --project ${PROJECT_ID} --no-confirm-project --no-wait \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"

# 4. Poll
agents-cli deploy --status

# 5. Record URL
export GITHUB_AGENT_URL="PASTE_AGENT_CARD_URL"
echo "export GITHUB_AGENT_URL='${GITHUB_AGENT_URL}'" >> ~/.bashrc

# 6. Test
curl -sS -H "Authorization: Bearer $(gcloud auth print-access-token)" "${GITHUB_AGENT_URL}" | python3 -m json.tool
export GITHUB_ENGINE_URL=$(echo "${GITHUB_AGENT_URL}" | sed -E 's#/reasoningEngines/v1/#/v1/#; s#/api/.*##')
agents-cli run --url "${GITHUB_ENGINE_URL}" --mode adk "Find the google/adk-python repository"
```

**âœ… Check**: Click "Check my progress" â†’ *Deploy the GitHub agent to Agent Runtime*

---

## TASK 3 â€” Stack Exchange Agent

```bash
# 1. Implement agent.py TODO: add_node("search", search) + edges STARTâ†’searchâ†’END
# 2. Implement server.py TODO: supported_interfaces with v1.0 and v0.3 AgentInterface

# 3. Build image
cd ~/agents/stackexchange-agent
gcloud builds submit --tag ${REGION}-docker.pkg.dev/${PROJECT_ID}/agents/stackexchange-agent:latest

# 4. Deploy
gcloud run deploy stackexchange-agent \
  --image ${REGION}-docker.pkg.dev/${PROJECT_ID}/agents/stackexchange-agent:latest \
  --region ${REGION} --allow-unauthenticated --port 8080

# 5. Set PUBLIC_URL
export SE_URL=$(gcloud run services describe stackexchange-agent --region ${REGION} --format='value(status.url)')
gcloud run services update stackexchange-agent --region ${REGION} --set-env-vars PUBLIC_URL=${SE_URL}

# 6. Record URL
export STACKEXCHANGE_AGENT_URL="${SE_URL}/.well-known/agent-card.json"
echo "export STACKEXCHANGE_AGENT_URL='${STACKEXCHANGE_AGENT_URL}'" >> ~/.bashrc
curl -sS "${STACKEXCHANGE_AGENT_URL}" | python3 -m json.tool
```

**âœ… Check**: Click "Check my progress" â†’ *Deploy the Stack Exchange agent to Cloud Run*

---

## TASK 4 â€” BQ Agent (Antigravity + GKE)

```bash
# 1. Install tooling
cd ~/agents
agents-cli setup --agent all
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform

# 2. Launch Antigravity
agy
# Login: Google Cloud Project, location: global
# Paste the generation prompt from docs/05-task4-bq-agent.md
# Wait ~40-50 minutes

# 3. Deploy (if Antigravity didn't auto-deploy)
cd ~/agents/bq-agent
agents-cli deploy --project ${PROJECT_ID} --no-confirm-project

# 4. Grant BigQuery permissions (if needed)
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:bq-agent-app@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:bq-agent-app@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataViewer"

# 5. Record URL
export BQ_AGENT_URL="http://EXTERNAL_IP:8080/a2a/app/.well-known/agent-card.json"
echo "export BQ_AGENT_URL='${BQ_AGENT_URL}'" >> ~/.bashrc
curl -sS "${BQ_AGENT_URL}" | python3 -m json.tool

# 6. Test
BQ_BASE_URL=$(echo "${BQ_AGENT_URL}" | sed -E 's#(://[^/]+).*#\1#')
agents-cli run --url "${BQ_BASE_URL}" --mode a2a "Have we seen a memory leak in a background worker before?"
```

**âœ… Check**: Click "Check my progress" â†’ *Deploy the Bug DB agent to GKE*

---

## TASK 5 â€” Root Orchestrator (Local Test)

```bash
# 1. Implement callbacks.py TODO: capture_response_to_state()
# 2. Implement auth.py TODO: GoogleADCAuth._apply() with before_request()
# 3. Implement agent.py TODOs:
#    - github_agent as RemoteA2aAgent (use_legacy=False, google_authed_client())
#    - stackexchange_agent as CleanQueryRemoteA2aAgent
#    - Workflow graph: STARTâ†’query_extractorâ†’[fan-out]â†’merge(JoinNode)â†’synthesizerâ†’END

# 4. Create .env
cd ~/agents/code-assistant
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
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

# 5. Verify all URLs set
blank=$(grep -E '^(GITHUB_AGENT_URL|STACKEXCHANGE_AGENT_URL|BQ_AGENT_URL)=$' .env)
if [ -n "${blank}" ]; then echo "BLANK: ${blank}"; else echo "All set."; fi

# 6. Publish Developer Knowledge skill
uv run python scripts/publish_skill.py --project ${PROJECT_ID} --location us-central1

# 7. Test locally
agents-cli install
agents-cli playground
# Test: "Our FastAPI service throws a 500 on startup after a dependency bump. What's the fix?"
# Verify multi-source cited answer
# CTRL+C to stop
```

**âœ… Check**: Click "Check my progress" â†’ *Orchestrate the specialists in a graph workflow*

---

## TASK 6 â€” Deploy Root + Register in Gemini Enterprise

```bash
# 1. Deploy root
cd ~/agents/code-assistant
rm -rf .venv
agents-cli deploy --project ${PROJECT_ID} --no-confirm-project --no-wait \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
agents-cli deploy --status  # poll until ACTIVE

# 2. Provision runtime service account
gcloud beta services identity create --service=aiplatform.googleapis.com --project=${PROJECT_ID}

# 3. Grant roles
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform-re.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform-re.iam.gserviceaccount.com" \
  --role="roles/discoveryengine.viewer"

# 4. Create Gemini Enterprise app (IN CONSOLE â€” not API):
#    Search "Gemini Enterprise" â†’ Create app
#    Name: enterprise-app, App ID: enterprise-app (exact), Location: global

# 5. Configure Google Identity provider:
#    Settings â†’ Authentication â†’ Edit (global) â†’ Google Identity â†’ Save

# 6. Register agent
agents-cli publish gemini-enterprise \
  --gemini-enterprise-app-id "projects/${PROJECT_NUMBER}/locations/global/collections/default_collection/engines/enterprise-app" \
  --display-name "Code Assist" \
  --description "Researches a production error across a bug database, the engineering handbook, Salesforce, GitHub, and Stack Exchange, and returns one cited fix."

# 7. Grant user access in console: enterprise-app â†’ Agents â†’ Code Assist â†’ User permissions â†’ Add user
```

**âœ… Check**: Click "Check my progress" â†’ *Deploy and authorize the root agent*

---

## TASK 7 â€” Salesforce Agent

```bash
# After Salesforce setup (see docs/08-task7-salesforce-agent.md):
# Implement salesforce_client.py TODO: authenticate() with POST to /services/oauth2/token
# Implement salesforce-agent/app/app_utils/a2a.py TODO: convert_a2a_request_to_agent_run_request
# Implement agent.py TODOs: generate_memories_callback + PreloadMemoryTool

cd ~/agents/salesforce-agent
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
SALESFORCE_DOMAIN=your-domain.develop.my.salesforce.com
SALESFORCE_CLIENT_ID=your_consumer_key
SALESFORCE_CLIENT_SECRET=your_consumer_secret
EOF

# Deploy
agents-cli deploy --project ${PROJECT_ID} --region ${REGION} --no-confirm-project \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"

# Allow public access
gcloud run services add-iam-policy-binding salesforce-agent \
  --region=${REGION} --member=allUsers --role=roles/run.invoker

# Record URL
export SF_URL=$(gcloud run services describe salesforce-agent --region ${REGION} --format='value(status.url)')
export SALESFORCE_AGENT_URL="${SF_URL}/a2a/app/.well-known/agent-card.json"
echo "export SALESFORCE_AGENT_URL='${SALESFORCE_AGENT_URL}'" >> ~/.bashrc
curl -sS "${SALESFORCE_AGENT_URL}" | python3 -m json.tool

# Test
agents-cli run --url "${SF_URL}" --mode a2a "ADR-007"

# Update root .env (replace "added-in-task-7" with actual URL)
# Then redeploy root:
cd ~/agents/code-assistant
rm -rf .venv
agents-cli deploy --project ${PROJECT_ID} --no-confirm-project --no-wait \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

**âœ… Check**: Click "Check my progress" â†’ *Deploy the Salesforce agent to Cloud Run*

---

## TASK 8 â€” Vertex AI Memory Bank

```bash
# Implement code-assistant/app/app_utils/a2a.py TODO: return {'user_id': ctx.user_id}
# Implement salesforce-agent/scripts/create_memory_bank.py TODO: main() idempotent create/find

# Create Memory Bank
cd ~/agents/salesforce-agent
GOOGLE_CLOUD_PROJECT=${PROJECT_ID} MEMORY_BANK_LOCATION=us-central1 \
  uv run python scripts/create_memory_bank.py
# Copy the numeric ID from "Resource name: .../reasoningEngines/XXXXXXXXX"

# Add to .env
echo "MEMORY_BANK_ID=XXXXXXXXX" >> .env
echo "MEMORY_BANK_LOCATION=us-central1" >> .env

# Redeploy Salesforce agent
agents-cli deploy --project ${PROJECT_ID} --region ${REGION} --no-confirm-project \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"

# Re-grant public access (redeploying resets IAM)
gcloud run services add-iam-policy-binding salesforce-agent \
  --region=${REGION} --member=allUsers --role=roles/run.invoker

# Redeploy root
cd ~/agents/code-assistant
rm -rf .venv
agents-cli deploy --project ${PROJECT_ID} --no-confirm-project --no-wait \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"

# Test memory:
# Conversation 1: "I'm on the Payments team and I only care about Spanner documents."
# Wait ~1 minute
# Conversation 2 (new session): ask related question without restating preference
# Verify recall
```

**âœ… Check**: Click "Check my progress" â†’ *Create a Vertex AI Memory Bank for the Salesforce agent*

---

## ðŸ” Auth Recovery Commands

```bash
# ADC expired (Python client libraries)
gcloud auth application-default login

# gcloud CLI expired
gcloud auth login

# Both broken â€” restart Cloud Shell, then:
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
export PATH="${HOME}/.local/bin:${PATH}"
source ~/.bashrc
gcloud auth login
gcloud auth application-default login
gcloud config set project ${PROJECT_ID}
```

---

## ðŸ“‹ All Environment Variables

```bash
PROJECT_ID=YOUR_PROJECT_ID
REGION=us-central1
GITHUB_AGENT_URL=https://....aiplatform.googleapis.com/.../agent-card.json
STACKEXCHANGE_AGENT_URL=https://stackexchange-agent-....run.app/.well-known/agent-card.json
BQ_AGENT_URL=http://EXTERNAL_IP:8080/a2a/app/.well-known/agent-card.json
SALESFORCE_AGENT_URL=https://salesforce-agent-....run.app/a2a/app/.well-known/agent-card.json
```

```bash
# Verify all are set
for var in PROJECT_ID REGION GITHUB_AGENT_URL STACKEXCHANGE_AGENT_URL BQ_AGENT_URL SALESFORCE_AGENT_URL; do
  echo "${var}: ${!var}"
done
```
