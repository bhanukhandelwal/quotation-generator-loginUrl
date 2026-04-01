# Gmail integration — what you need to do

This guide assumes **Google Workspace** and that you will **not** push to git from here; deploy and configure in your own Google Cloud / Admin consoles.

## Overview

| Piece | Role |
|--------|------|
| **Gmail Add-on** (`gmail-addon/`) | Shows in Gmail; opens your **webapp** with `threadId`, `messageId`, subject, etc. |
| **Webapp** (`webapp/index.html`) | Opens the quotation generator; user attaches PDF; **Sign in with Google**; calls backend. |
| **Backend** (`backend/`) | Receives PDF + `threadId`; calls **Gmail API** to create a **reply draft** in that thread. |

---

## 1) Google Cloud project

1. Open [Google Cloud Console](https://console.cloud.google.com/) and create or select a project.
2. **APIs & Services → Library** → enable **Gmail API**.

---

## 2) OAuth consent screen

1. **APIs & Services → OAuth consent screen**.
2. Choose **Internal** (Workspace users only) unless you need external users.
3. Add scopes the app will request:
   - `https://www.googleapis.com/auth/gmail.modify` (create drafts with attachments — used by **webapp → backend** flow via user token).

Optional (if you use add-on metadata fetch as written):

- Add-on Apps Script project has its own scopes in `appsscript.json` (e.g. `gmail.addons.execute`, `gmail.readonly`).

---

## 3) OAuth 2.0 Client ID (Web application)

1. **APIs & Services → Credentials → Create credentials → OAuth client ID**.
2. Application type: **Web application**.
3. **Authorized JavaScript origins**: URLs where `webapp/index.html` is hosted, e.g.
   - `https://your-domain.com`
   - `http://localhost:8080` (for local testing only)
4. **Authorized redirect URIs**: not required for the **Google Identity Services** token flow used in `index.html` (popup/redirect-less token client). If you change to a redirect-based flow, add URIs here.

Copy the **Client ID** — you will paste it into `webapp/index.html` as `OAUTH_CLIENT_ID`.

---

## 4) Deploy the backend

1. On a machine with Node 20+:

   ```bash
   cd gmail-integration/backend
   npm install
   npm run dev
   ```

2. For production, use the included `Dockerfile` or host on **Cloud Run** / similar with HTTPS and env `PORT=8080`.
3. Note the public URL, e.g. `https://qg-backend-xxxxx.run.app`.
4. In `webapp/index.html`, set `BACKEND_URL` to `https://qg-backend-xxxxx.run.app/api/gmail/reply-draft`.

**CORS:** the backend allows any origin by default. For production, restrict `cors` in `backend/src/index.js` to your webapp origin only.

---

## 5) Host the webapp

1. Upload or deploy `gmail-integration/webapp/index.html` (same origin as in step 3 origins), or open locally for dev with a static server.
2. Set in the file:
   - `OAUTH_CLIENT_ID` — from step 3
   - `GENERATOR_URL` — your hosted quotation app (e.g. GitHub Pages URL to `index.html`)
   - `BACKEND_URL` — from step 4

3. Test: open webapp → **Connect Google** → pick PDF → **Create Gmail reply draft** → check Gmail **Drafts** / thread.

---

## 6) Gmail Add-on (Apps Script)

1. Go to [script.google.com](https://script.google.com) → New project.
2. Replace `Code.gs` with `gmail-integration/gmail-addon/Code.gs`.
3. Add `appsscript.json` via **Project Settings → manifest** (or include manifest in clasp project).
4. Set `WEBAPP_BASE_URL` in `Code.gs` to the **base URL of your hosted webapp** (no trailing slash issues — the code appends `?gmail=1&...`).
5. **Deploy → New deployment** → type **Add-on** (or follow [Workspace Add-on deploy](https://developers.google.com/workspace/add-ons/how-tos/publish-add-on-overview)).
6. As Workspace **Admin**: install / allow the add-on for your domain (Marketplace or internal deployment per Google’s current flow).

---

## 7) End-to-end check

Use `gmail-integration/end-to-end-test-checklist.md` and verify:

- Add-on opens webapp with `threadId` filled.
- Draft appears **in the same thread** with PDF attached.
- Token: user must click **Connect Google** when scopes or session require it (access tokens expire; user can connect again).

---

## Troubleshooting

| Symptom | What to check |
|--------|----------------|
| `redirect_uri_mismatch` / GIS errors | Authorized **JavaScript origins** must match the page origin exactly (scheme + host + port). |
| `403` / insufficient permissions | Consent screen includes `gmail.modify`; user completed consent. |
| Draft not in thread | `threadId` must be the Gmail thread id (add-on passes it; do not truncate). |
| CORS errors | Backend URL correct; production: allow only your webapp origin. |

---

## Security notes

- Do **not** commit OAuth **client secret** into public repos for a public SPA; the webapp uses **client ID** only (GIS token client).
- The backend trusts the **Bearer** access token; deploy backend on HTTPS and restrict CORS in production.
- For stricter control, add server-side session or Google ID token verification + allowlist of Workspace domains.
