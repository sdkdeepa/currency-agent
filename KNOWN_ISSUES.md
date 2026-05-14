# Currency Agent Codelab — Known Issues & Fixes

A reference guide for issues encountered while running through the [Getting Started with MCP, ADK and A2A](https://codelabs.developers.google.com/codelabs/currency-agent) codelab. Keep this open during the workshop.

## Table of Contents

- [Issue 1: Frankfurter API moved domains](#issue-1-frankfurter-api-moved-domains-step-4)
- [Issue 2: Edits not taking effect](#issue-2-edits-not-taking-effect-step-4)
- [Issue 3: "Forbidden" error in browser](#issue-3-forbidden-when-visiting-cloud-run-url-in-browser-step-5)
- [Issue 4: Test script fails after deploying](#issue-4-test-script-fails-after-deploying-to-cloud-run-step-5)
- [Issue 5: ADC error in agent UI](#issue-5-could-not-resolve-project-using-application-default-credentials-step-6)
- [Issue 6: Agent silently fails with 401](#issue-6-agent-returns-nothing-logs-show-401-unauthorized-step-6)
- [Required Tab Setup](#required-tab-setup)
- [Pre-flight Checklist](#pre-flight-checklist)

---

## Issue 1: Frankfurter API moved domains (Step 4)

### Symptom

When running `uv run mcp-server/test_server.py` after starting the local MCP server, you see:

```
✅ Success: {"error":"API request failed: Redirect response '301 Moved Permanently'
for url 'https://api.frankfurter.app/latest?from=USD&to=EUR'
Redirect location: 'https://api.frankfurter.dev/v1/latest?from=USD&to=EUR'"}
```

### Cause

The Frankfurter API migrated from `api.frankfurter.app` to `api.frankfurter.dev/v1/`. The codelab still references the old URL.

### Fix

Edit `mcp-server/server.py` and update the URL:

```python
# Before
response = httpx.get(
    f"https://api.frankfurter.app/{currency_date}",
    params={"from": currency_from, "to": currency_to},
)

# After
response = httpx.get(
    f"https://api.frankfurter.dev/v1/{currency_date}",
    params={"from": currency_from, "to": currency_to},
)
```

Or apply the fix in one shot:

```bash
sed -i 's|api.frankfurter.app|api.frankfurter.dev/v1|g' mcp-server/server.py
```

Save the file, then restart the MCP server (Ctrl+C in the server tab, then re-run `uv run mcp-server/server.py`).

---

## Issue 2: Edits not taking effect (Step 4)

### Symptom

You edited `server.py` but the test still shows the old URL in the error.

### Cause

The running MCP server process doesn't auto-reload on file changes. It's still serving the old code from memory.

### Fix

Restart the server.

```bash
# In the tab running the MCP server, press Ctrl+C
# Then re-run:
uv run mcp-server/server.py
```

Verify the file was actually saved before restarting:

```bash
grep -n "frankfurter" mcp-server/server.py
```

You should see `frankfurter.dev/v1/`. If you still see `frankfurter.app`, the save didn't go through — re-edit and save again.

---

## Issue 3: "Forbidden" when visiting Cloud Run URL in browser (Step 5)

### Symptom

After deploying with `gcloud run deploy`, you visit the service URL in your browser and see:

```
Error: Forbidden
Your client does not have permission to get URL / from this server.
```

### Cause

This is **expected behavior**, not a bug. You deployed with `--no-allow-unauthenticated`, which requires every request to include a valid auth token. Your browser doesn't send one, so Cloud Run rejects it.

### Fix

No fix needed. The service is working correctly. Test it using the Cloud Run proxy (see Issue 4) instead of the browser.

---

## Issue 4: Test script fails after deploying to Cloud Run (Step 5)

### Symptom

After deploying to Cloud Run, running `uv run mcp-server/test_server.py` throws a long traceback ending in connection errors or redirects.

### Cause

The test script is hardcoded to `http://localhost:8080/mcp`. After deploying, nothing is running on localhost — your server is now on Cloud Run, which requires auth. The Cloud Run proxy bridges this gap.

### Fix

Start the Cloud Run proxy in a new terminal tab. Leave it running for the rest of the codelab.

```bash
# In a new Cloud Shell tab:
gcloud run services proxy mcp-server --region=us-central1
```

You should see:

```
Proxying to Cloud Run service [mcp-server] in project [PROJECT_ID] region [us-central1]
http://127.0.0.1:8080 proxies to https://mcp-server-XXXXX.us-central1.run.app
```

> ⚠️ **Do not Ctrl+C this.** It needs to run for the rest of the workshop.

In another tab, run the test:

```bash
cd ~/currency-agent
uv run mcp-server/test_server.py
```

You should see real exchange rate data come back.

---

## Issue 5: "Could not resolve project using application default credentials" (Step 6)

### Symptom

When asking the agent a question in the `adk web` UI, an error toast appears:

```
{"error": "Could not resolve project using application default credentials."}
```

### Cause

ADK uses Vertex AI to call Gemini. Vertex AI needs Application Default Credentials (ADC) to know which Google Cloud project to bill. Cloud Shell usually has these set, but not always.

### Fix

Set up ADC explicitly:

```bash
gcloud auth application-default login
```

Follow the URL it prints, sign in with the same Google account, paste the verification code back into the terminal.

Also verify your `.env` file has actual values, not unexpanded variables:

```bash
cat .env
```

You should see literal values, not `$PROJECT_ID`:

```
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=currency-496005
GOOGLE_CLOUD_LOCATION=us-central1
```

If you see `$PROJECT_ID` instead of the resolved value, recreate the file:

```bash
cd ~/currency-agent
rm -f .env
echo "GOOGLE_GENAI_USE_VERTEXAI=TRUE" >> .env
echo "GOOGLE_CLOUD_PROJECT=$PROJECT_ID" >> .env
echo "GOOGLE_CLOUD_LOCATION=us-central1" >> .env
```

Restart `adk web` (Ctrl+C, then `uv run adk web`).

---

## Issue 6: Agent returns nothing, logs show "401 Unauthorized" (Step 6)

### Symptom

The agent UI responds with nothing visible, but the `adk web` logs show:

```
HTTP Request: POST http://localhost:8080/mcp "HTTP/1.1 401 Unauthorized"
ERROR: ASGI callable returned without completing response.
```

### Cause

The Cloud Run proxy is no longer running. The agent is trying to call MCP at `localhost:8080`, but the proxy that bridges to Cloud Run was closed.

### Fix

Restart the Cloud Run proxy in its own tab:

```bash
gcloud run services proxy mcp-server --region=us-central1
```

Then refresh the agent UI or click "New Session" and ask the question again. The proxy must stay running for any agent invocation that uses MCP tools.

---

## Required Tab Setup

By the end of the codelab, you need **three terminals running simultaneously**, plus a fourth for ad-hoc commands. Don't close any of them until you've completed all the tests.

| Tab | Step Introduced | Command | Purpose |
|---|---|---|---|
| 1 | Step 4, reused at Step 5 | First: `uv run mcp-server/server.py` <br/> Then: `gcloud run services proxy mcp-server --region=us-central1` | Local MCP server first, then the Cloud Run proxy after deploy |
| 2 | Step 4 | `uv run mcp-server/test_server.py` | One-off test runs |
| 3 | Step 6, reused at Step 8 | First: `uv run adk web` <br/> Then: `uv run uvicorn currency_agent.agent:a2a_app --host localhost --port 10000` | Agent dev UI first, then the A2A server |
| 4 | Step 8 | `uv run currency_agent/test_client.py` | A2A test client |

### Critical handoffs

- **Between Step 4 and Step 5:** Stop the local MCP server (Tab 1) before starting the Cloud Run proxy. Both want port 8080.
- **Between Step 6 and Step 8:** Stop `adk web` (Tab 3) before starting the A2A server.

---

## Pre-flight Checklist

Run these before starting Step 4 to avoid problems:

```bash
# 1. Confirm you're authenticated
gcloud auth list

# 2. Confirm the project is set
gcloud config get-value project

# 3. Set up application default credentials
gcloud auth application-default login

# 4. Verify .env has real values
cat ~/currency-agent/.env

# 5. Apply the Frankfurter URL fix BEFORE running the local server
sed -i 's|api.frankfurter.app|api.frankfurter.dev/v1|g' ~/currency-agent/mcp-server/server.py
grep "frankfurter" ~/currency-agent/mcp-server/server.py
```

