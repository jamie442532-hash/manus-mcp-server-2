# manus-mcp-server

A Remote MCP server that wraps the Manus API and exposes a `generate_image`
tool, so Claude (web/app) can request images and have Manus generate them.

## How it works

Manus's API is a general task API, not a dedicated image endpoint. This
server:

1. Receives a `generate_image` tool call from Claude (prompt, optional style/aspect ratio).
2. Creates a Manus task asking the agent to produce an image and return it as a file.
3. Polls the task until it completes (or times out after 4 minutes).
4. Returns the resulting file URL(s) back to Claude as the tool result.

## 1. Deploy on Railway

1. Push this repo to GitHub.
2. In Railway: **New Project → Deploy from GitHub repo** → select this repo.
3. Railway will detect `railway.json` and use Nixpacks to run
   `npm install && npm run build`, then `npm start`.
4. In the Railway project, go to **Variables** and set:
   - `MANUS_API_KEY` — your Manus API key.
   - `MANUS_API_BASE_URL` — defaults to `https://api.manus.im/v1`; check your Manus API docs/dashboard and adjust if different.
   - `MCP_SERVER_TOKEN` — make up a long random string (e.g. `openssl rand -hex 32`). This protects your server from being called by strangers.
   - `PORT` — Railway sets this automatically; no action needed.
5. Once deployed, go to **Settings → Networking → Generate Domain** to get a public HTTPS URL, e.g. `https://manus-mcp-server-production.up.railway.app`.
6. Confirm it's alive: visit `https://<your-domain>/health` — should return `{"status":"ok"}`.

## 2. Verify the Manus API endpoint shape

The exact paths (`/tasks`, response field names like `task_id` vs `id`) can
vary by account/API version. Before relying on this in production:

- Check `https://manus.im/docs/integrations/manus-api` (or your account's
  API reference) for the current create-task and get-task endpoints and
  response fields.
- If they differ from what's in `src/manusClient.ts`, adjust the constants
  at the top of that file (`CREATE_TASK_PATH`, `GET_TASK_PATH`, and the
  field names read in `createImageTask`/`getTask`) and redeploy.
- Easiest way to check without guessing: make one manual `curl` call to
  create a task and print the raw JSON, then match the field names.

## 3. Connect it to Claude

In Claude (web or app): **Settings → Connectors → Add custom connector**
(the exact label may read "Remote MCP server" or similar).

- **URL**: `https://<your-railway-domain>/mcp`
- **Auth**: if Claude's UI supports a custom header/bearer token field, set
  it to `Bearer <your MCP_SERVER_TOKEN>`. If it only supports OAuth, you'll
  need to either leave `MCP_SERVER_TOKEN` unset (open server — only do this
  if you're comfortable with that) or add an OAuth layer in front (out of
  scope here, ask if you want this built).

Once connected, you can just ask Claude for an image and it will call the
`generate_image` tool automatically.

## Local development

```bash
npm install
cp .env.example .env   # fill in your real values
npm run dev
```

Server runs at `http://localhost:3000/mcp`.
