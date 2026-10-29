# 10 â€” Troubleshooting Guide

## Quick Reference: Most Common Issues

| Error message | Likely cause | Fix |
|---------------|-------------|-----|
| `service account info is missing 'email' field` | ADC expired | `gcloud auth application-default login` |
| `gcloud reauthentication` / no active account | CLI auth expired | `gcloud auth login` |
| `403 Permission Denied` on Agent Runtime call | Missing IAM role for runtime SA | Grant `roles/aiplatform.user` |
| `403 Permission Denied` on datastore | Missing IAM role | Grant `roles/discoveryengine.viewer` |
| `403 PERMISSION_DENIED` on datastore create | Datastore still provisioning | Wait 10 seconds, retry |
| `Unauthorized` on `kubernetes_*` resource | GKE cluster just created | Re-run `agents-cli deploy` |
| BQ `PERMISSION_DENIED` | `bq-agent-app` missing BigQuery roles | Grant `roles/bigquery.jobUser` + `roles/bigquery.dataViewer` |
| A2A card parse failure | Protocol version mismatch (1.x vs 0.3) | Ensure server advertises 0.3-compatible interface |
| Memory doesn't persist | `MEMORY_BANK_ID` not set or wrong location | Check `.env`, use `us-central1` |
| Memory Bank endpoint error | Using `global` instead of `us-central1` | Set `MEMORY_BANK_LOCATION=us-central1` |
| IAM grants fail (SA doesn't exist) | Service Agent not yet provisioned | Re-run `gcloud beta services identity create`, wait, retry |

---

## Authentication Issues

### `service account info is missing 'email' field`

**Cause**: Application Default Credentials (ADC) have expired or were never set.  
**Fix**:
```bash
gcloud auth application-default login
```
Follow the browser link and authenticate. This sets credentials for Python client libraries.

---

### `gcloud reauthentication` or no active account

**Cause**: `gcloud` CLI credentials expired.  
**Fix**:
```bash
gcloud auth login
```

---

### Both auth errors persist after re-running

**Fix**: Restart Cloud Shell:
- Click **Restart** in the Cloud Shell toolbar
- Re-run your environment variable exports:
  ```bash
  export PROJECT_ID=YOUR_PROJECT_ID
  export REGION=us-central1
  export PATH="${HOME}/.local/bin:${PATH}"
  source ~/.bashrc
  ```

---

## IAM / Permission Issues

### `403 Permission Denied` when root calls GitHub Agent

**Cause**: Runtime service account lacks permission to call Agent Runtime.  
**Fix**:
```bash
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-aiplatform-re.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
```
Wait 1â€“2 minutes for IAM to propagate.

---

### Service account `service-...@gcp-sa-aiplatform-re.iam.gserviceaccount.com` does not exist

**Cause**: Service Agent not yet provisioned.  
**Fix**:
```bash
gcloud beta services identity create \
  --service=aiplatform.googleapis.com \
  --project=${PROJECT_ID}
```
Wait a few seconds, then retry the IAM grant.

---

### BigQuery `PERMISSION_DENIED` in BQ agent

**Cause**: `bq-agent-app` service account missing BigQuery roles.  
**Fix**:
```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:bq-agent-app@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/bigquery.jobUser"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:bq-agent-app@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/bigquery.dataViewer"
```

---

## Deployment Issues

### agents-cli deploy fails with `Unauthorized` on kubernetes_* resource

**Cause**: GKE Autopilot cluster was just created; credentials haven't propagated yet.  
**Fix**: Just re-run the deploy:
```bash
agents-cli deploy --project ${PROJECT_ID} --no-confirm-project
```

---

### `.venv` causes deploy failure (symlink error)

**Cause**: `.venv/bin/python` is a symlink pointing outside the project directory.  
**Fix**: Remove it before deploying:
```bash
rm -rf .venv
agents-cli deploy ...
```
Agent Runtime rebuilds dependencies from `pyproject.toml` and `uv.lock`.

---

### Deploy fails with `Required field is not set`

**Cause**: A specialist URL in `.env` is blank.  
**Diagnosis**:
```bash
grep -E '^(GITHUB_AGENT_URL|STACKEXCHANGE_AGENT_URL|BQ_AGENT_URL|SALESFORCE_AGENT_URL)=$' .env
```
**Fix**: Re-export the missing URL from the value you recorded, then recreate `.env`.

---

### Cloud Run service returns `503` after deploy

**Cause**: Container hasn't finished starting (cold start on first request).  
**Fix**: Wait 30 seconds and retry.

---

## A2A Protocol Issues

### ADK root can't parse the Stack Exchange agent card

**Cause**: Stack Exchange server uses `a2a-sdk 1.x` but ADK client uses `a2a-sdk 0.3` â€” card shapes differ.  
**Fix**: Ensure `server.py` advertises **both** 1.0 and 0.3 interfaces in `supported_interfaces`.

---

### Remote agent returns empty results / no output in synthesizer

**Cause**: `RemoteA2aAgent` doesn't set `output_key`, so its reply isn't in session state.  
**Fix**: Ensure each remote agent node has `after_agent_callback=capture_response_to_state("key_name")`.

---

## BigQuery / Vector Search Issues

### Vector search returns wrong/unrelated results

**Cause**: Query and stored vectors use different embedding models or dimensions.  
**Fix**: Both must use `text-embedding-004` (768 dimensions). Check the `tools.py` embedding call.

---

### `VECTOR_SEARCH` returns no results

**Cause**: The table may be empty, or the query vector is malformed.  
**Diagnosis**: Check the BigQuery table directly in the Cloud Console â†’ BigQuery â†’ `code_assist_bugs.known_bugs`.

---

## Memory Bank Issues

### Memory doesn't persist between conversations

**Possible causes**:
1. `MEMORY_BANK_ID` not set in `.env`
2. Using wrong location (`global` instead of `us-central1`)
3. User ID forwarding not implemented (root doesn't pass `user_id`)

**Fix**:
```bash
# Verify .env has:
grep MEMORY_BANK .env
# Should show:
# MEMORY_BANK_ID=<numeric-id>
# MEMORY_BANK_LOCATION=us-central1
```

---

### Memory generation is too slow

**Expected behavior**: Memory Bank processes sessions **asynchronously** after the turn ends. Wait **~1 minute** before testing recall.

---

### `Memory Bank endpoint not found` error

**Cause**: Using `global` location for Memory Bank.  
**Fix**: Always use `MEMORY_BANK_LOCATION=us-central1`.

---

## Salesforce Issues

### `401 Unauthorized` from Salesforce API

**Cause**: Client credentials flow not properly enabled, or Run-As user not set.  
**Fix**: Check the External Client App â†’ Edit Policies â†’ confirm:
- `Enable Client Credentials Flow` is checked
- Run-As user is set to your Salesforce username

---

### `invalid_client` error during authentication

**Cause**: Wrong Consumer Key or Consumer Secret in `.env`.  
**Fix**: Re-copy the credentials from Salesforce: External Client App Manager â†’ L400 Lab â†’ Settings â†’ Consumer Key and Secret.

---

### `IP restrictions` / connection refused

**Cause**: Salesforce IP restrictions block the Cloud Run egress IP.  
**Fix**: In External Client App Manager â†’ L400 Lab â†’ Edit Policies â†’ set **IP Relaxation** to `Relax IP restrictions`.

---

## Environment Variable Reference

Run this to check all key variables are set:

```bash
echo "PROJECT_ID: ${PROJECT_ID}"
echo "REGION: ${REGION}"
echo "GITHUB_AGENT_URL: ${GITHUB_AGENT_URL}"
echo "STACKEXCHANGE_AGENT_URL: ${STACKEXCHANGE_AGENT_URL}"
echo "BQ_AGENT_URL: ${BQ_AGENT_URL}"
echo "SALESFORCE_AGENT_URL: ${SALESFORCE_AGENT_URL}"
```

To reload from `.bashrc` (e.g., after Cloud Shell reconnect):
```bash
source ~/.bashrc
```
