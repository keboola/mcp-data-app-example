# Keboola MCP Server as a Keboola Data App

> A reference template for hosting the [Keboola MCP Server](https://github.com/keboola/keboola-mcp-server) as a **private, single-tenant Keboola data app** in your own project. Fork it, drop in your secrets, deploy, and you have a dedicated MCP endpoint that Claude, Cursor, Windsurf, and other MCP clients can connect to.

Keboola already runs a shared multi-tenant remote MCP server at `https://mcp.<your-region>.keboola.com/mcp` — for most users that's the right starting point. Pick this template when you want any of:

- A **dedicated, isolated instance** running on your own Keboola compute, with its own URL and rotation cadence.
- The ability to **fork the upstream MCP server** and ship custom tools.
- A **worked example** of "host any MCP server as a data app" — copy this directory layout to wrap a different MCP server (the [`ghost-mcp` companion repo](#related) does the same for Ghost CMS).

## What you get

| Feature | How |
|---|---|
| All upstream Keboola MCP tools (storage, components, SQL, jobs, flows, …) | The data-app entry point mounts the upstream `keboola-mcp-server` package at `/mcp`. |
| Public HTTPS URL with no DNS/TLS work | Keboola's data-app proxy. |
| Two ways to authenticate clients | Static bearer for programmatic agents; OAuth-shape stubs for Claude Desktop / claude.ai's "Add custom connector" GUI flow. Same shared secret backs both. |
| Process management, auto-restart, logs | Supervisord, visible in the Keboola data-app Terminal Log tab. |
| Secrets, not in code | All credentials live as `#`-prefixed Keboola data-app secrets. |

## Repository layout

```
keboola-mcp-data-app-example/
├── server.py                                   # Entry point: mounts upstream MCP + auth + OAuth-shape stubs
├── pyproject.toml                              # Deps; pins upstream keboola-mcp-server to a tag
├── keboola-config/
│   ├── nginx/sites/default.conf                # 8888 → 5000 reverse proxy, SSE-safe, Host-rewrite trick
│   ├── supervisord/services/mcp-server.conf    # `uv run python server.py` on port 5000
│   └── setup.sh                                # `uv sync` at container start
├── README.md
└── LICENSE                                     # MIT
```

## Required configuration

Set these as **secrets** on the Keboola data-app config (prefix with `#` so they're encrypted).

| Secret | What | Where to get it |
|---|---|---|
| `#KBC_STORAGE_API_URL` | Your Keboola stack URL, no trailing slash. | E.g. `https://connection.us-east4.gcp.keboola.com`. Check your Keboola UI URL bar. |
| `#KBC_STORAGE_TOKEN` | Storage API token. All MCP tools run with this token's permissions; scope it to what your AI agent needs. | Project Settings → API tokens → New token. |
| `#MCP_API_KEY` | The client-auth secret you generate. Doubles as the **static bearer** and the **OAuth `client_secret`**. | `openssl rand -hex 32` |

Optional:

| Var | Default | What |
|---|---|---|
| `KBC_WORKSPACE_SCHEMA` | unset | Snowflake/BigQuery schema for the SQL-transformation tools. Find it in your project's workspace settings. |
| `LOG_LEVEL` | `INFO` | `DEBUG` while iterating, `INFO` in prod. |
| `PORT` | `5000` | Don't change; the Nginx config expects 5000. |
| `#MCP_PUBLIC_URL` | `KBC_APP_PUBLIC_URL` | Override for the origin advertised by the OAuth discovery documents. Keboola injects `KBC_APP_PUBLIC_URL` with the app's own URL, so leave this unset unless the app is reached at a different origin (custom domain, reverse proxy). No trailing slash, no `/mcp`. |

## Deploy

1. **Fork this repo** to a GitHub account Keboola can clone (private or public, either works).
2. In your Keboola project: **Apps → Create App → Python/JS Data App**.
3. Point at your fork (repo URL + branch).
4. Add the three required `#`-prefixed secrets above.
5. **App-level auth: set to "No auth".** Keboola's app-level OIDC strips the `Authorization` header before it reaches the container, which breaks both auth patterns below. The MCP_API_KEY is the security boundary.
6. **Auto-suspend window**: set to **24 hours** or longer so the first call of each day doesn't pay cold-start.
7. **Click Deploy.** Wait for the container to come up.
8. Verify: `curl https://<your-app-url>/healthz` → `{"status":"ok",...}`.

No second pass is needed. The discovery documents pick up the app's own URL from
`KBC_APP_PUBLIC_URL`, which Keboola injects into every data-app container.

## Connect a client

### Pattern A — Static bearer (managed agents, scripts, Claude Desktop config)

In a Claude Desktop `claude_desktop_config.json` (or any MCP client that supports a JSON config), wire it with `mcp-remote`:

```json
{
  "mcpServers": {
    "keboola": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "https://<your-app-url>/mcp",
        "--header",
        "Authorization:Bearer ${MCP_API_KEY}"
      ],
      "env": { "MCP_API_KEY": "<paste-MCP_API_KEY>" }
    }
  }
}
```

For an Anthropic Managed Agent: pick "Static bearer token" in the credential-vault UI and paste `MCP_API_KEY`.

### Pattern B — OAuth-shape GUI flow (Claude Desktop / claude.ai "Add custom connector")

The custom-connector UI only speaks OAuth. This server impersonates an OAuth 2.1 authorization server but funnels everything through the same `MCP_API_KEY`:

1. **Add custom connector** → Remote MCP server URL: `https://<your-app-url>/mcp`
2. Open **Advanced settings**:
   - **OAuth Client ID**: any non-empty string, e.g. `claude`
   - **OAuth Client Secret**: paste the `MCP_API_KEY`
3. Click **Add** → a browser tab flashes through `/authorize` and closes; connector goes live and lists the upstream Keboola MCP tools.

Smoke-test from your shell first:
```bash
curl -sS https://<your-app-url>/.well-known/oauth-protected-resource
curl -sS https://<your-app-url>/.well-known/oauth-authorization-server
curl -sS -i https://<your-app-url>/mcp | head -5   # expect 401 + WWW-Authenticate
```

The discovery JSON should show this app's own URL. If it shows some *other* app's
URL, this config was duplicated from one that sets the `#MCP_PUBLIC_URL` override —
duplication copies secrets verbatim, and the override wins over the platform value.
Remove `#MCP_PUBLIC_URL` from the copy and redeploy. If it shows `127.0.0.1:5000`,
the app is running somewhere that does not inject `KBC_APP_PUBLIC_URL`; set the
override explicitly.

## How auth works under the hood

`server.py` adds one middleware and five extra routes to the Starlette app that hosts the upstream MCP:

```
                            ┌──────────────────────────────────────┐
                            │  Keboola data-app container           │
                            │                                        │
 Claude client ── HTTPS ──▶ │ Nginx :8888 ──▶ server.py :5000        │
                            │   │                │                   │
                            │   │  BearerAuthMiddleware (all paths   │
                            │   │  except ANON_PATHS)                │
                            │   │                │                   │
                            │   │  ANON_PATHS: /healthz,             │
                            │   │   /.well-known/oauth-*, /register, │
                            │   │   /authorize, /token               │
                            │   │                │                   │
                            │   └────────────────┼───▶ Mount("/mcp", │
                            │                    │      upstream     │
                            │                    │      Keboola      │
                            │                    │      FastMCP app) │
                            └────────────────────┴───────────────────┘
```

The OAuth-shape endpoints are **stubs** — `/token` simply requires `client_secret == MCP_API_KEY` and returns `MCP_API_KEY` itself as the `access_token`. That access_token is what the bearer middleware then validates on `/mcp` calls. Net effect: clients that speak OAuth can connect *as if* there were a real authorization server, but the security profile is identical to the static-bearer flow.

A longer write-up of the pattern (including nginx specifics and a comparison with real OAuth) lives in [`hosting-remote-mcp-server-as-keboola-data-app.md` upstream](https://github.com/keboola/changelog-agent-ghost-mcp/blob/main/README.md) — this repo is its concrete Keboola-flavored worked example.

## Customizing

- **Pin a different upstream version**: edit `pyproject.toml` and change the `@v1.60.2` tag at the end of the `keboola-mcp-server @ git+...` URL.
- **Add custom tools**: import `mcp_server` from `server.py` and decorate functions with `@mcp_server.tool()` before the `app.add_middleware(...)` line.
- **Use a real OAuth AS** instead of the stubs: delete the five stub routes and `BearerAuthMiddleware`, then follow the FastMCP `AuthSettings` / `TokenVerifier` pattern.
- **Rotate the bearer**: bump `#MCP_API_KEY` on the data app, redeploy, re-paste in every client. The OAuth-shape "access token" is the same value so it rotates with it.

## Related

- [`keboola/keboola-mcp-server`](https://github.com/keboola/keboola-mcp-server) — the upstream MCP server this template wraps.
- [`keboola/changelog-agent-ghost-mcp`](https://github.com/keboola-rnd/changelog-agent-ghost-mcp) — the same data-app + bearer + OAuth-shape pattern wrapping Ghost CMS instead.

## License

MIT. See [LICENSE](LICENSE).
