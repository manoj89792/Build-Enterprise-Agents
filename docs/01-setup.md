# 01 â€” Prerequisites & Initial Setup

## Before You Begin

> âš ï¸ **Do these steps BEFORE starting the timed lab session.**  
> Salesforce signup takes 5â€“10 minutes and requires email verification.

---

## 1. Salesforce Developer Edition Account

### 1.1 Create Account

1. Open [https://developer.salesforce.com/signup](https://developer.salesforce.com/signup)
2. Fill in: First name, Last name, Email, Company, Role, Country, Username (must be email-format, e.g. `yourname@lab.salesforce.com`)
3. Click **Sign me up**
4. Accept the Master Subscription Agreement
5. Open verification email (subject: *"Welcome to Salesforce: Verify your account"*)
6. Click **Verify Account**
7. Set a password (min 8 chars, 1 letter, 1 number), choose security question â†’ **Change Password**

### 1.2 Upload Lab Documents to Salesforce Files

```bash
# In Cloud Shell â€” copy the lab documents ZIP
gcloud storage cp gs://l400-academy-data-science/l400-apex-001/Salesforce_documents.zip .
```

Then:
- Download the ZIP to your local machine (Cloud Shell Editor â†’ right-click â†’ Download)
- Extract the documents
- In Salesforce: **App Launcher (â‹®â‹®â‹®)** â†’ search **Files** â†’ **Upload Files**
- Upload all extracted documents
- Wait a few minutes for Salesforce to index them

> ðŸ’¡ **Remote Browser tip**: If your corporate policies block downloads, use the Remote Browser session URL provided in the lab. The documents are already on the Desktop.

---

## 2. GitHub Personal Access Token (PAT)

1. Go to [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. Click **Generate new token** â†’ **Generate new token (classic)**
3. Set Note: `code-assist-lab`, choose an Expiration
4. Under **Select scopes**, check:
   - âœ… `repo`
   - âœ… `read:org`
5. Click **Generate token**
6. âš ï¸ **Copy the `ghp_...` value immediately** â€” GitHub shows it only once!

Save format:
```
GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

## 3. Credentials Checklist

Save all credentials to a **secure local file** before starting the lab. They are reused in follow-on labs.

| Credential | Where to get it | Variable name |
|-----------|----------------|--------------|
| GitHub PAT | github.com/settings/tokens | `GITHUB_PERSONAL_ACCESS_TOKEN` |
| Salesforce Consumer Key | SF External Client App â†’ Settings | `SALESFORCE_CLIENT_ID` |
| Salesforce Consumer Secret | SF External Client App â†’ Settings | `SALESFORCE_CLIENT_SECRET` |
| Salesforce My Domain URL | SF Setup â†’ My Domain | `SALESFORCE_DOMAIN` |
| GCP Project ID | Lab Setup pane | `PROJECT_ID` |

---

## 4. Browser Setup

- Use **Incognito / Private** mode to avoid account conflicts
- Use **Chrome** (recommended)
- Have two windows side-by-side: Google Cloud Console + Lab instructions

---

## 5. Lab Access

1. Click **Start Lab** button
2. Right-click **Open Google Cloud Console** â†’ Open in new window
3. Use the **lab-provided credentials** (not your personal account)
4. Accept terms, skip recovery/2FA setup, skip free trial offers

---

## 6. System Requirements

- Standard internet browser (Chrome recommended)
- No special local software required (all work done in Cloud Shell)
- ~3 hours uninterrupted time once lab starts
