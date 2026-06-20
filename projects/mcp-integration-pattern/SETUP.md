# Setup

A generic, copy-paste quickstart for implementing this MCP pattern against your own REST backend.
This repository documents the pattern rather than shipping server source, so the steps below
describe how to wire up *your* server — the commands are illustrative and language-agnostic.
**Every value below is a placeholder.** Never commit real keys, hosts, or paths — substitute
your own via the secret store and environment variables described here.

## Prerequisites

- A runtime for your server (e.g. Python 3.11+ or Node 18+, depending on the stack you build it on).
- An MCP server library for that runtime.
- An MCP-capable client (any agent that supports MCP).
- An API key for your backend platform.

## 1. Provision the external secret store

Secrets live **outside** the repository. The server resolves them from a JSON file whose
location is given by an environment variable, with a default under your user profile.

Create the store (example shape — keys are illustrative):

```jsonc
// $APP_CONFIG_PATH  (e.g. ~/.config/<app>/config.json)
{
  "platform": {
    "api_key":  "${PLATFORM_API_KEY}",
    "base_url": "${PLATFORM_BASE_URL}",        // e.g. https://api.example.com
    "assets_dir_unix":    "/path/to/local/assets",
    "assets_dir_windows": "C:\\path\\to\\local\\assets"
  }
}
```

Lock it down with filesystem permissions so only the running user can read it:

```bash
# POSIX
chmod 600 "$APP_CONFIG_PATH"
```

```powershell
# Windows: restrict to the current user
icacls "$env:APP_CONFIG_PATH" /inheritance:r /grant:r "$($env:USERNAME):(R)"
```

## 2. Point the server at the store

```bash
# POSIX shells
export APP_CONFIG_PATH="$HOME/.config/<app>/config.json"
```

```powershell
# PowerShell
$env:APP_CONFIG_PATH = "$HOME\.config\<app>\config.json"
```

If `APP_CONFIG_PATH` is unset, the server falls back to its documented default location.
It will **fail loudly** if the file is missing or the API key is empty — by design, rather
than silently using an insecure default.

## 3. Install dependencies

Install the MCP server library for whichever runtime you build on. For example:

```bash
# Python
python -m pip install -r requirements.txt
```

```bash
# TypeScript
npm install && npm run build
```

## 4a. Run over stdio (local, default)

This is the recommended default: the client spawns the server, and there is no open port.
Invoke your server's stdio entry point — for example:

```bash
# Python
python server.py

# TypeScript
node dist/index.js
```

Register it with your MCP client (generic form):

```bash
# Example: a CLI agent that takes "mcp add <name> -- <command>"
<your-agent> mcp add my-platform -- python /abs/path/to/server.py
```

Or via a client config file:

```jsonc
{
  "mcpServers": {
    "my-platform": {
      "command": "python",
      "args": ["/abs/path/to/server.py"],
      "env": { "APP_CONFIG_PATH": "/abs/path/to/config.json" }
    }
  }
}
```

## 4b. Run over Streamable HTTP (hosted, multi-client)

Use this only when you need a shared, hosted instance. The credential is taken from the
per-request `Authorization` header instead of the local file.

```bash
# Bind address/port via env (placeholders)
export HOST="127.0.0.1"
export PORT="${MCP_PORT}"          # e.g. 8080
node dist/http.js                  # or: python server.py --http
```

Health check and a sample authenticated call:

```bash
curl "http://${HOST}:${PORT}/health"

curl -X POST "http://${HOST}:${PORT}/mcp" \
  -H "Authorization: Bearer ${PLATFORM_API_KEY}" \
  -H "Content-Type: application/json" \
  --data '{ "jsonrpc": "2.0", "id": 1, "method": "tools/list" }'
```

Front it with a reverse proxy / tunnel for TLS and external reach. Keep it **stateless**
(a fresh server + transport per request) so it scales horizontally without session affinity.

## 5. Verify the tool catalog

From your MCP client, list tools and confirm the read-vs-destructive split is visible:

```text
list_*      → read-only, safe to call speculatively
create_*    → DESTRUCTIVE (state-changing / cost-incurring) — confirm intent first
```

## Notes

- **Rotation:** because the key is read per call, rotating it in the secret store takes
  effect on the next tool invocation — no restart required.
- **CDN/WAF:** if your backend sits behind a WAF that rejects default HTTP-client
  fingerprints, set a custom `User-Agent` in the HTTP adapter (one line, configured once).
- **Never** pass the API key as a command-line argument — it would be visible to anyone who
  can list processes. Use the secret store or the per-request header only.
