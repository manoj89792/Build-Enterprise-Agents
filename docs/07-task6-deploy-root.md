# Task 6 â€” Deploy the Root Agent and Register in Gemini Enterprise

## Overview

Deploy the root agent to Agent Runtime, authorize its service account to call specialists, verify the full system end-to-end, and register it in Gemini Enterprise so users can access it via a web app.

---

## Step 1: Set Environment Variables

```bash
export PROJECT_ID=YOUR_PROJECT_ID
export REGION=us-central1
gcloud config set project ${PROJECT_ID}
```

---

## Step 2: Deploy the Root Agent

> âš ï¸ Remove `.venv` first â€” it contains symlinks that break the packager.

```bash
cd ~/agents/code-assistant

# Remove local venv (Agent Runtime rebuilds from pyproject.toml + uv.lock)
rm -rf .venv

# Deploy
agents-cli deploy \
  --project ${PROJECT_ID} \
  --no-confirm-project \
  --no-wait \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

**Why remove `.venv`?** Testing locally created a virtual environment. `agents-cli` packages the directory as an archive. `.venv/bin/python` is a symlink pointing outside the project â€” the packager rejects this. Agent Runtime rebuilds deps server-side.

### Monitor deployment

```bash
# Poll until ACTIVE (takes ~10 minutes)
agents-cli deploy --status
```

---

## Step 3: Authorize the Root Agent's Service Account

When running locally (Task 5), the root called specialists with **your** user credentials. Now it runs as the **runtime service account** â€” which has a valid token but hasn't been granted permission yet.

**Missing grants cause**: `403 Permission Denied` (valid token, insufficient authorization).

### 3.1 Provision the Reasoning Engine Service Agent

```bash
gcloud beta services identity create \
  --service=aiplatform.googleapis.com \
  --project=${PROJECT_ID}
```

This creates `service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com`.

> If the output only names the Vertex AI Service Agent, that's fine â€” the Reasoning Engine Agent is also created.

### 3.2 Grant the required roles

```bash
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')

# roles/aiplatform.user: allows calling the GitHub Agent + loading skills from Skill Registry
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform-re.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"

# roles/discoveryengine.viewer: allows querying the code-manuals datastore
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform-re.iam.gserviceaccount.com" \
  --role="roles/discoveryengine.viewer"
```

> If a grant fails with "service account does not exist", wait a few seconds and re-run `gcloud beta services identity create`, then retry the grants.

> **IAM changes take 1â€“2 minutes to propagate.** If the first end-to-end call fails, wait and retry.

---

## Step 4: Verify the Deployed System

### In the Cloud Console

1. Open **Agent Platform** â†’ **Agents** â†’ **Scale** â†’ **Deployments**  
   (or search "Agent Platform" â†’ Deployments)
2. Click your `code-assistant` runtime
3. Click the **Playground** tab
4. Send a test query:

```
A CloudSQL connection fails with too many connections under load. What's the fix?
```

**Expected**: A single, cited answer blending multiple sources.

### In Agent Registry

From the left nav â†’ **Govern** â†’ **Agent Registry** â†’ confirm `code-assistant` appears.

---

## Step 5: Create the Gemini Enterprise App

> âš ï¸ **Create in the console, not the API.** Creating via API skips setup steps that are required before agent registration.

1. In the Cloud Console, search for **Gemini Enterprise**
2. Accept terms â†’ **Continue** (if prompted, to enable the service)
3. Click **Create your first app** (or **Create app**)
4. App name: `enterprise-app`
5. Click **Edit** next to the App ID â†’ set it to exactly `enterprise-app`  
   (the publish command targets `.../engines/enterprise-app`)
6. Location: `global`
7. Click **Create**

---

## Step 6: Configure Identity Provider

1. Go to **Gemini Enterprise** â†’ **Settings** â†’ **Authentication**
2. Click **Edit** for the global location
3. In **Add identity provider** â†’ click **Google Identity**
4. Click **Save**

---

## Step 7: Register the Agent in Gemini Enterprise

```bash
cd ~/agents/code-assistant

# Get project number if not already set
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')

agents-cli publish gemini-enterprise \
  --gemini-enterprise-app-id "projects/${PROJECT_NUMBER}/locations/global/collections/default_collection/engines/enterprise-app" \
  --display-name "Code Assist" \
  --description "Researches a production error across a bug database, the engineering handbook, Salesforce, GitHub, and Stack Exchange, and returns one cited fix."
```

> **Note**: Run from `~/agents/code-assistant` so CLI auto-detects the reasoning engine from `deployment_metadata.json`.

> If the console appended a suffix to your App ID (e.g. `enterprise-app_1699...`), use your actual App ID in the command.

---

## Step 8: Share the Agent with Users

1. In Gemini Enterprise console â†’ open `enterprise-app` â†’ **Agents**
2. Confirm **Code Assist** is listed
3. Click **Code Assist** â†’ **User permissions** tab
4. Click **Add user** â†’ add your lab user email (or `All users`) â†’ role: **Agent User** â†’ **Save**

---

## Step 9: Test in Gemini Enterprise Web App

1. Go to **Integration** tab â†’ open the web app link
2. Choose **Code Assist** in the agents window
3. Ask a multi-source question:

```
A CloudSQL connection fails with "too many connections" under load.
Have we hit this incident before, what does the engineering handbook recommend, and what are the known fixes?
```

**Verify the reply**:
- âœ… Cites past incidents from the bug database (BUG-XXXX)
- âœ… References the internal engineering handbook
- âœ… Cites GitHub or Stack Exchange results
- âœ… Single coherent, cited answer

---

## âœ… Task 6 Checkpoint

- [ ] Root agent deployed (`agents-cli deploy --status` shows `ACTIVE`)
- [ ] Reasoning Engine Service Agent provisioned
- [ ] `roles/aiplatform.user` granted to runtime service account
- [ ] `roles/discoveryengine.viewer` granted to runtime service account
- [ ] Console Playground test returns multi-source answer
- [ ] `code-assistant` visible in Agent Registry
- [ ] Gemini Enterprise app created with ID `enterprise-app`
- [ ] Google Identity provider configured
- [ ] Agent registered and visible in enterprise-app
- [ ] User permissions set
- [ ] Web app test returns multi-source cited answer

**Check progress**: Click **Check my progress** â†’ *Deploy and authorize the root agent*.

---

## Authentication vs Authorization: Quick Reference

| Scenario | Error | Fix |
|----------|-------|-----|
| No token sent | `401 Unauthorized` | Configure ADC / httpx_client |
| Valid token, no permission | `403 Permission Denied` | Add IAM role binding |
| Token expired | `401` during execution | ADC auto-refreshes; if persistent, re-run `gcloud auth application-default login` |

---

## Why Two IAM Roles?

| Role | What it unlocks |
|------|----------------|
| `roles/aiplatform.user` | Call GitHub Agent on Agent Runtime (`aiplatform.reasoningEngines.query`) + load skills from Skill Registry |
| `roles/discoveryengine.viewer` | Query the `code-manuals` Vertex AI Search datastore |
