# Deploying this fork to Railway (org-restricted)

Self-host the Google Workspace MCP as a remote server, with sign-in locked to your
`construct.sg` Google Workspace. Docker-based, persistent, HTTPS, auto-redeploy on push.

The MCP endpoint will be: `https://<your-domain>/mcp`

---

## 1. Google Cloud — OAuth (the org restriction lives here)

1. [console.cloud.google.com](https://console.cloud.google.com/) → create/select a project **inside the construct.sg org**.
2. **APIs & Services → OAuth consent screen → User Type = Internal.** ← restricts login to construct.sg. Fill app name + support email.
3. **Enabled APIs & services → Enable APIs** for what you'll use: Gmail, Drive, Calendar, Docs, Sheets, Slides, Forms, Tasks, People. (Custom Search / Chat / Apps Script only if needed — Chat needs extra app config.)
4. **Credentials → Create credentials → OAuth client ID → Web application.**
   - Leave redirect URI empty for now (added in step 3 below).
   - Save the **Client ID** and **Client secret**.

## 2. Railway — create the service

1. [railway.com](https://railway.com/) → **New Project → Deploy from GitHub repo** → `edisonb24/google_workspace_mcp`. It builds from the `Dockerfile` (via `railway.json`).
2. **Add a Volume** on the service, mount path **`/data`**.
3. **Variables** → add the ones below (Raw Editor makes this a single paste).
   - Fill `GOOGLE_OAUTH_CLIENT_ID` / `GOOGLE_OAUTH_CLIENT_SECRET` from step 1.
   - Generate the signing key once:
     `python3 -c "import secrets;print(secrets.token_urlsafe(48))"`
   - Leave `WORKSPACE_EXTERNAL_URL` / `GOOGLE_OAUTH_REDIRECT_URI` as placeholders for now.
4. **Settings → Networking → Generate Domain.** Copy the `https://<name>.up.railway.app`.

```dotenv
# Google OAuth (from Google Cloud → Credentials → Web application)
GOOGLE_OAUTH_CLIENT_ID=<your-client-id>.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=<your-client-secret>

# Transport / OAuth 2.1 multi-user mode
MCP_ENABLE_OAUTH21=true
WORKSPACE_MCP_TRANSPORT=streamable-http
WORKSPACE_MCP_HOST=0.0.0.0
# Do NOT set WORKSPACE_MCP_PORT — Railway injects PORT and the server reads it.

# Public URL — set to the real domain AFTER step 4 (no trailing slash)
WORKSPACE_EXTERNAL_URL=https://<name>.up.railway.app
GOOGLE_OAUTH_REDIRECT_URI=https://<name>.up.railway.app/oauth2callback

# Persistence — one volume mounted at /data
WORKSPACE_MCP_CREDENTIALS_DIR=/data/creds
WORKSPACE_MCP_OAUTH_PROXY_STORAGE_BACKEND=disk
WORKSPACE_MCP_OAUTH_PROXY_DISK_DIRECTORY=/data/oauth-proxy
FASTMCP_SERVER_AUTH_GOOGLE_JWT_SIGNING_KEY=<paste-generated-key>

# Let the non-root image write to the root-owned Railway volume
RAILWAY_RUN_UID=0
# Leave OAUTHLIB_INSECURE_TRANSPORT / OAUTH2_ALLOW_INSECURE_TRANSPORT unset in prod.
```

## 3. Wire the domain back

1. Railway → Variables → set the two URL vars to the real domain (no trailing slash):
   - `WORKSPACE_EXTERNAL_URL=https://<name>.up.railway.app`
   - `GOOGLE_OAUTH_REDIRECT_URI=https://<name>.up.railway.app/oauth2callback`
2. Google Cloud → your OAuth client → **Authorized redirect URIs** → add
   `https://<name>.up.railway.app/oauth2callback` → Save.
3. Railway redeploys automatically on the variable change.

## 4. Verify

```bash
curl https://<name>.up.railway.app/health          # → 200 OK
curl https://<name>.up.railway.app/.well-known/oauth-protected-resource   # → JSON metadata
```

Then add the server to your MCP client as a **streamable-http / remote** server at
`https://<name>.up.railway.app/mcp`, run the OAuth flow, and sign in with your
`construct.sg` account. A non-org Google account should be rejected at the consent screen.

## Updating later

`git fetch upstream && git merge upstream/main && git push` → Railway auto-redeploys.

## Custom domain (optional)

Railway → add domain `mcp.construct.sg` (set the CNAME) → then update
`WORKSPACE_EXTERNAL_URL`, `GOOGLE_OAUTH_REDIRECT_URI`, and the GCP redirect URI to match.
