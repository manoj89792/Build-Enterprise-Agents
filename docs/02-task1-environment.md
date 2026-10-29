# Task 1 â€” Environment Setup

## Objectives
- Launch Cloud Shell and open the Cloud Shell Editor
- Download lab agent code
- Install `agents-cli`
- Configure Application Default Credentials (ADC)
- Create the Vertex AI Search datastore for internal code manuals

---

## Step 1: Open Cloud Shell

In the Google Cloud Console, press `G` then `S` (or click the **Activate Cloud Shell** button â˜ï¸ in the top right).

Click **Continue** and then **Authorize** when prompted.

---

## Step 2: Set Environment Variables

```bash
# Authenticate CLI
gcloud auth login

# Set project and region
export PROJECT_ID=YOUR_PROJECT_ID   # replace with your actual project ID
export REGION=us-central1
gcloud config set project ${PROJECT_ID}
```

> ðŸ“Œ **Tip**: Add these exports to `~/.bashrc` so they persist across Cloud Shell reconnects:
> ```bash
> echo "export PROJECT_ID=${PROJECT_ID}" >> ~/.bashrc
> echo "export REGION=us-central1" >> ~/.bashrc
> ```

---

## Step 3: Download Lab Code

```bash
# Download agent code from the lab bucket
gcloud storage cp -r gs://${PROJECT_ID}-bucket/agents .

# Download VS Code settings
mkdir ~/.vscode
gcloud storage cp -r gs://${PROJECT_ID}-bucket/settings.json ~/.vscode/
```

**Result**: You should see an `~/agents/` directory containing:
```
~/agents/
â”œâ”€â”€ github-agent/
â”œâ”€â”€ stackexchange-agent/
â”œâ”€â”€ bq-agent/           â† will be generated in Task 4
â”œâ”€â”€ code-assistant/
â””â”€â”€ salesforce-agent/
```

---

## Step 4: Install agents-cli

```bash
# Add local bin to PATH
export PATH="${HOME}/.local/bin:${PATH}"
echo 'export PATH="${HOME}/.local/bin:${PATH}"' >> ~/.bashrc

# Install agents-cli using uv
uv tool install google-agents-cli

# Verify installation
agents-cli info
```

**Expected output**: Version info for agents-cli and its dependencies.

---

## Step 5: Configure Application Default Credentials (ADC)

> âš ï¸ **This step is critical.** `gcloud auth login` only authenticates the CLI.  
> The Python client libraries (used by agents-cli) need ADC separately.

```bash
gcloud auth application-default login
```

Follow the browser link, authenticate with your lab Google account, and confirm authorization.

> If you skip this step, deploys fail with:  
> `service account info is missing 'email' field`

---

## Step 6: Open Cloud Shell Editor

```bash
# Open Cloud Shell in a new, larger window
# Click the "Open in new window" button (â¤¢) in the Cloud Shell terminal panel

# Open Cloud Shell Editor pointing to your home directory
cloudshell workspace ~
```

Close any extra panels (tutorial sidebar, Gemini) to maximize screen space.

> **For the rest of this lab, work in this window** â€” editor on top, terminal below.

---

## Step 7: Create the Code Manuals Search Datastore

The root agent's `manual_search` branch queries a Vertex AI Search datastore of internal code manuals. Create it now (it provisions in the background while you continue with Task 2).

### 7.1 Create the datastore

```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores?dataStoreId=code-manuals-datastore" \
  -d '{
    "displayName":"Code manuals",
    "industryVertical":"GENERIC",
    "solutionTypes":["SOLUTION_TYPE_SEARCH"],
    "contentConfig":"CONTENT_REQUIRED"
  }'
```

**Expected response**: JSON with `"name": "projects/.../dataStores/code-manuals-datastore"` and an `"operations/..."` reference.

### 7.2 Import the manual documents

```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/code-manuals-datastore/branches/0/documents:import" \
  -d '{
    "reconciliationMode":"INCREMENTAL",
    "gcsSource":{
      "inputUris":["gs://'${PROJECT_ID}'-bucket/code_manuals/*"],
      "dataSchema":"content"
    }
  }'
```

> If you get `403 PERMISSION_DENIED`, wait ~10 seconds and retry (the datastore is still provisioning).

---

## âœ… Task 1 Checkpoint

- [ ] Cloud Shell is open with the editor
- [ ] `~/agents/` directory exists with agent subdirectories
- [ ] `agents-cli info` runs successfully
- [ ] ADC is configured (`gcloud auth application-default login` completed)
- [ ] Datastore import started (can continue while it provisions)

**Check progress**: In the lab UI, click **Check my progress** â†’ *Create the code-manuals datastore*.

---

## Environment Variables Reference

| Variable | Value | Set in |
|----------|-------|--------|
| `PROJECT_ID` | Your GCP project ID | `~/.bashrc` |
| `REGION` | `us-central1` | `~/.bashrc` |
| `PATH` | Includes `~/.local/bin` | `~/.bashrc` |
