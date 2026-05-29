# Zero-Touch DevLog Pipeline — Setup Guide

Automatically turns every `[Milestone]` git commit into a polished AI-written changelog entry stored in Supabase — no manual writing required.

**Flow:** GitHub push → n8n webhook → extract commit → fetch git diff → Google Gemini AI → Supabase devlogs table

---

> **Security rule — read this first.**
> Your API keys (GitHub token, Gemini key, Supabase service role) are secrets.
> Enter them **only** inside n8n's Credential Manager (encrypted at rest).
> **Never** paste them into a chat, email, terminal history, or any document —
> including the Claude Code prompt in Option A below.

---

## Table of Contents

- [What You Need](#what-you-need)
- [Option A — Claude Code (Guided Setup)](#option-a--claude-code-guided-setup)
  - [Fastest path — rename to CLAUDE.md](#fastest-path--rename-this-file-to-claudemd)
  - [Alternative — paste the prompt manually](#alternative--paste-the-prompt-manually)
- [Option B — Manual Step-by-Step](#option-b--manual-step-by-step)
  - [Step 1 — Download the Workflow](#step-1--download-the-workflow)
  - [Step 2 — Import into n8n](#step-2--import-into-n8n)
  - [Step 3 — Set Up Supabase](#step-3--set-up-supabase)
  - [Step 4 — Generate a GitHub Token](#step-4--generate-a-github-token)
  - [Step 5 — Get a Google Gemini API Key](#step-5--get-a-google-gemini-api-key)
  - [Step 6 — Enter Credentials in n8n](#step-6--enter-credentials-in-n8n)
  - [Step 7 — Activate the Workflow & Copy the Webhook URL](#step-7--activate-the-workflow--copy-the-webhook-url)
  - [Step 8 — Add the Webhook to GitHub](#step-8--add-the-webhook-to-github)
  - [Step 9 — Test the Pipeline](#step-9--test-the-pipeline)
- [How to Trigger a Devlog Entry](#how-to-trigger-a-devlog-entry)
- [Troubleshooting](#troubleshooting)

---

## What You Need

| Service | Purpose | Free tier |
|---|---|---|
| **n8n** (self-hosted or n8n.cloud) | Workflow engine | Self-host is free |
| **Supabase** | Database storage for devlogs | Yes |
| **GitHub** | Source repository + webhook trigger | Yes |
| **Google Gemini API** | AI-generated release notes | Yes (generous free tier) |

> **n8n hosting:** You can run n8n locally with Docker, on a VPS (Hostinger, DigitalOcean, etc.), or use [n8n.cloud](https://n8n.cloud). A 24/7 server is recommended so the webhook is always reachable.

---

## Option A — Claude Code (Guided Setup)

If you have [Claude Code](https://claude.ai/code) installed, you can let it read this file as its instructions and drive the entire setup for you.

### Fastest path — rename this file to `CLAUDE.md`

1. Rename `README.md` to `CLAUDE.md` in the `Devlog_Pipeline` folder.
2. Open Claude Code in that folder.
3. Type this short prompt — Claude Code will already have the full setup instructions loaded from `CLAUDE.md`:

```
Set up the DevLog Pipeline on my n8n instance.
My n8n URL is: <YOUR_N8N_URL>
GitHub repository to track: <OWNER/REPO>
```

That's it. Claude Code reads `CLAUDE.md` automatically as project context, so it already knows every step, credential name, and configuration detail without you having to explain anything.

### Alternative — paste the prompt manually

If you prefer to keep the file as `README.md`, paste this into a Claude Code session:

```
I want to set up the Zero-Touch DevLog Pipeline from baiplot.com on my n8n instance.

- n8n URL: <YOUR_N8N_URL>                  ← your n8n address only, no secrets
- GitHub repository to track: <OWNER/REPO> ← e.g. Darek027/my-project

Please:
1. Help me import DevLog Pipeline.json into n8n
2. Open the n8n Credential Manager for each required credential so I can
   type my keys directly into n8n's encrypted form — do not ask me to
   share any API keys in this chat
3. Confirm all three credentials are attached to the correct nodes
4. Activate the workflow and give me the webhook URL
5. Walk me through adding the webhook on GitHub with a webhook secret
6. Guide me through sending a test [Milestone] commit to verify the full pipeline
```

> **Security note:** Both prompts above contain only your n8n URL and repo name — no secrets. Claude Code will open the n8n Credential Manager in your browser for each key so you type them directly into n8n's encrypted form. Never add API keys to the prompt.

---

## Option B — Manual Step-by-Step

### Step 1 — Download the Workflow

1. Go to **baiplot.com** and navigate to the DevLog Pipeline product page.
2. Click **Download** — you will get a file named `DevLog Pipeline.json`.
3. Save it somewhere accessible (e.g. Desktop or Downloads).

---

### Step 2 — Import into n8n

1. Open your n8n instance in a browser.
2. In the left sidebar click **Workflows**.
3. Click the **+** button (top right) → **Import from file**.
4. Select `DevLog Pipeline.json`.
5. The workflow opens with 5 nodes in a horizontal chain:
   ```
   Webhook → Code in JavaScript → Github → Google_Gemini → Supabase_Entry
   ```
6. Do **not** activate it yet — enter credentials first.

---

### Step 3 — Set Up Supabase

#### 3a — Create a project

1. Go to [supabase.com](https://supabase.com) → **New project**.
2. Set a name (e.g. `devlog-db`), choose a nearby region, set a database password.
3. Wait ~1 minute for provisioning to complete.

#### 3b — Create the `devlogs` table

1. Go to **Table Editor** → **New table**.
2. Name it exactly: `devlogs`
3. Add these columns (the default `id` and `created_at` are created automatically):

| Column name | Type | Nullable |
|---|---|---|
| `commit_hash` | `text` | No |
| `project_name` | `text` | No |
| `message` | `text` | Yes |
| `published` | `boolean` | No — default `false` |

4. Click **Save**.

#### 3c — Enable Row Level Security

1. Go to **Authentication → Policies** → select the `devlogs` table.
2. Enable RLS.
3. Add a `SELECT` policy restricted to rows where `published = true`. This ensures your public frontend can only read published entries — all draft rows remain private.

#### 3d — Locate your Supabase credentials

1. In Supabase go to **Project Settings → API**.
2. Note the **Project URL** and the **service_role** key.

> These values will be entered directly into n8n's Credential Manager in Step 6.
> Do not copy them into any notes, chat, or document.

---

### Step 4 — Generate a GitHub Token

1. On GitHub go to **Settings → Developer settings → Personal access tokens → Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Name it `devlog-pipeline` and set an expiry that fits your security policy.
4. Under **Scopes** check only `repo` — nothing else is needed.
5. Click **Generate token**.

> The token is shown only once. Switch directly to your n8n tab and enter it there (Step 6).
> If you navigate away without saving it to n8n, revoke it and generate a new one.

---

### Step 5 — Get a Google Gemini API Key

1. Go to [aistudio.google.com](https://aistudio.google.com).
2. Click **Get API key → Create API key**.
3. Select or create a Google Cloud project.
4. The key is generated immediately.

> Switch directly to your n8n tab and enter it there (Step 6). Do not store it anywhere else.

---

### Step 6 — Enter Credentials in n8n

All secrets are entered here, inside n8n's encrypted Credential Manager. **This is the only place they should ever exist.**

#### 6a — GitHub Token

1. In n8n go to **Credentials** (left sidebar) → **+ Add credential**.
2. Search for **HTTP Request (Multiple Headers Auth)** and select it.
3. Name it exactly: `GitHub Token`
4. Add these two headers — the **Name** column is the header name, the **Value** column is where you type your actual secret:

| Name | Value |
|---|---|
| `Authorization` | Type: `Bearer ` followed immediately by your GitHub token |
| `X-GitHub-Api-Version` | `2022-11-28` |

5. Click **Save**.

#### 6b — Google Gemini API Key

1. **+ Add credential** → **HTTP Request (Multiple Headers Auth)**.
2. Name it exactly: `Google Gemini API Key`
3. Add one header:

| Name | Value |
|---|---|
| `x-goog-api-key` | Type your Gemini API key here |

4. Click **Save**.

#### 6c — Supabase

1. **+ Add credential** → search **Supabase** and select it.
2. Name it: `Supabase account`
3. Fill in:
   - **Host**: your Supabase Project URL
   - **Service Role Secret**: your Supabase service_role key
4. Click **Save**.

> Use the **service_role** key (not `anon`) — n8n needs write access to insert rows.

#### 6d — Attach credentials to nodes

Back in your workflow:

1. Click the **Github** node → under Authentication select `GitHub Token`.
2. Click the **Google_Gemini** node → select `Google Gemini API Key`.
3. Click the **Supabase_Entry** node → select `Supabase account`.
4. Click **Save** (top right).

---

### Step 7 — Activate the Workflow & Copy the Webhook URL

1. Click the **Webhook** node in the editor.
2. Copy the **Production URL** — it looks like:
   ```
   https://YOUR-N8N-DOMAIN/webhook/github-push
   ```
3. Toggle the workflow **Active** switch (top right) to turn it on.

> **Running locally?** GitHub cannot reach `localhost`. Either deploy to a public server, or use [ngrok](https://ngrok.com): run `ngrok http 5678` and use the resulting HTTPS URL instead.

---

### Step 8 — Add the Webhook to GitHub

Do this for every repository you want the pipeline to track.

1. Go to the GitHub repository → **Settings → Webhooks → Add webhook**.
2. Fill in:
   - **Payload URL**: the webhook URL from Step 7
   - **Content type**: `application/json`
   - **Secret**: generate a random string (e.g. use a password manager or `openssl rand -hex 32`) and save it in your password manager — this proves to n8n that payloads genuinely come from GitHub
   - **Which events?**: **Just the push event**
3. Make sure **Active** is checked.
4. Click **Add webhook**.

GitHub sends an immediate ping. Check your n8n **Executions** tab to confirm it was received.

---

### Step 9 — Test the Pipeline

1. In a tracked repository make a commit whose message starts with `[Milestone]`:
   ```bash
   git commit -m "[Milestone] Initial release of authentication module"
   git push
   ```
2. In n8n, open the **Executions** tab — a successful run should appear within seconds.
3. In Supabase go to **Table Editor → devlogs** — confirm a new row with:
   - `commit_hash` — the SHA of your commit
   - `project_name` — your repository name
   - `message` — an AI-written release note from Gemini

> Commits that do **not** start with `[Milestone]` are silently filtered out.

---

## How to Trigger a Devlog Entry

The only daily action required is prefixing a commit message with `[Milestone]`:

```bash
# Creates a devlog entry
git commit -m "[Milestone] Added dark mode support across all dashboard pages"

# Ignored by the pipeline
git commit -m "fix typo in footer"
git commit -m "wip: auth refactor"
```

The AI reads the full git diff and writes a concise, professional changelog entry automatically.

---

## Troubleshooting

**Webhook not firing**
- Confirm the workflow is **Active** in n8n.
- In GitHub go to **Settings → Webhooks → Recent Deliveries** — check for a green checkmark.
- If running locally, confirm ngrok is running and the URL in GitHub matches your current ngrok HTTPS address.

**Execution fails at the Github node**
- Verify the GitHub token has `repo` scope.
- The `Authorization` header value must start with `Bearer ` (with a space) followed by the token.
- Confirm the token has not expired.

**Execution fails at Google_Gemini node**
- Verify the Gemini API key is active in Google Cloud and the Generative Language API is enabled for your project.
- The node retries automatically — check the execution log for the HTTP status code returned.

**No row in Supabase**
- Confirm the credential uses the **service_role** key, not `anon`.
- The table must be named exactly `devlogs` (lowercase) with columns `commit_hash`, `project_name`, and `message`.

**Commit was ignored**
- The filter requires the message to start with exactly `[Milestone]` — check for extra spaces or different capitalisation.
