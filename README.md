# mcp-grafana-unraid

<p align="center">
  <img src="icons/mcp-grafana.png" alt="Grafana MCP" width="200" />
</p>

[![Lint](https://github.com/paulmeier/mcp-grafana-unraid/actions/workflows/lint.yml/badge.svg)](https://github.com/paulmeier/mcp-grafana-unraid/actions/workflows/lint.yml)
[![Release Please](https://github.com/paulmeier/mcp-grafana-unraid/actions/workflows/release-please.yml/badge.svg)](https://github.com/paulmeier/mcp-grafana-unraid/actions/workflows/release-please.yml)
[![Release](https://img.shields.io/github/v/release/paulmeier/mcp-grafana-unraid)](https://github.com/paulmeier/mcp-grafana-unraid/releases)

Unraid Community Applications template for [Grafana MCP](https://github.com/grafana/mcp-grafana) — the official Model Context Protocol server from Grafana Labs. It exposes your Grafana instance to AI assistants and other MCP clients (such as Claude) so they can search dashboards, run Prometheus and Loki queries, inspect datasources, and review alerts and incidents through scoped tools.

## Template

- **Container image**: `grafana/mcp-grafana:latest` (multi-arch: amd64, arm64)
- **Project**: https://github.com/grafana/mcp-grafana
- **Container registry**: https://hub.docker.com/r/grafana/mcp-grafana

The container runs in **SSE transport mode on port 8000** (the image's default entrypoint already binds `0.0.0.0:8000`). MCP clients connect to the endpoint at **`/sse`**. This is a **headless server — there is no web UI**.

## Requirements

> [!IMPORTANT]
> You need a **reachable Grafana instance** and a **Grafana service account token**. The MCP server is a client of your Grafana — it does not bundle one. Create the token *before* starting the container. See [Set up a service account token](#1-set-up-a-grafana-service-account-token) below.

## Installation

### 1. Set up a Grafana service account token

1. In Grafana, go to **Administration → Users and access → Service accounts**.
2. **Add a service account** and give it only the roles you want the MCP server to be able to use (for example, a read-only viewer role if you only want querying).
3. Open the service account and **Add a token**. Copy the token — you will not be able to see it again.

> The token replaces the deprecated Grafana API key. Basic-auth (`GRAFANA_USERNAME` / `GRAFANA_PASSWORD`) is also supported as a fallback if you prefer — add those as extra variables in the Docker editor.

### 2. Install the template

#### Via Community Applications (recommended)

Search for **Grafana-MCP** in the Unraid **Apps** tab.

To add this repository as a template source manually:

1. In Unraid, open the **Apps** tab and go to **Settings**.
2. Under **Template Repositories**, add:
   ```
   https://github.com/paulmeier/mcp-grafana-unraid
   ```
3. Click **Check for Updates**, then search for **Grafana-MCP**.

#### Manual installation

1. In Unraid, go to **Docker** → **Add Container**.
2. Click **Load template from URL** and paste:
   ```
   https://raw.githubusercontent.com/paulmeier/mcp-grafana-unraid/main/templates/mcp-grafana.xml
   ```
3. Click **Load**.

### 3. Configure and start

Fill in the two required fields, then **Apply**:

| Field | What to enter |
| ----- | ------------- |
| **Grafana URL** | Full base URL of your Grafana, e.g. `http://192.168.1.10:3000`. See the note below about `localhost`. |
| **Grafana Service Account Token** | The token from step 1. Stored masked. |

> [!WARNING]
> **Do not use `localhost` or `127.0.0.1` in the Grafana URL.** Inside the container those point back at the MCP server itself, not your Grafana. If Grafana is another Docker container on the same Unraid host, use the **Unraid host LAN IP** with Grafana's published port (e.g. `http://192.168.1.10:3000`), or put both containers on the same **custom Docker network** and use the **Grafana container name** as the host.

## Connecting an MCP client

The server speaks the **SSE** transport on port 8000. Point your MCP client at:

```
http://<UNRAID-IP>:8000/sse
```

A liveness check is available at `http://<UNRAID-IP>:8000/healthz` (this is what the container's **WebUI** button opens — there is no browsable interface).

**Example client config** (Claude Desktop / any client that supports remote SSE MCP servers):

```json
{
  "mcpServers": {
    "grafana": {
      "url": "http://192.168.1.10:8000/sse"
    }
  }
}
```

### Streamable HTTP instead of SSE

The image defaults to SSE. To use the newer **streamable HTTP** transport, add a **Post Argument** of `--transport streamable-http` in the Docker editor (Advanced View). The endpoint then moves to:

```
http://<UNRAID-IP>:8000/mcp
```

## Configuration

Settings are passed as environment variables. The template exposes the common ones; add the rest with **Add another Path, Port, Variable, Label or Device** in the Docker editor.

| Variable | Purpose |
| -------- | ------- |
| `GRAFANA_URL` | Base URL of your Grafana instance (**required**). |
| `GRAFANA_SERVICE_ACCOUNT_TOKEN` | Service account token for auth (**required**, recommended). Stored masked. |
| `GRAFANA_USERNAME` / `GRAFANA_PASSWORD` | Basic-auth alternative to the token. |
| `GRAFANA_ORG_ID` | Organization ID, for multi-org instances. |
| `GRAFANA_EXTRA_HEADERS` | Extra HTTP headers as a JSON object (e.g. `{"X-Scope-OrgID":"1"}` for Mimir/Grafana Cloud tenants). |
| `TZ` | Container timezone for log timestamps. |

See the [Grafana MCP documentation](https://github.com/grafana/mcp-grafana) for the full list, including per-tool enable/disable flags and TLS options.

> [!CAUTION]
> The MCP server can do anything the service account token allows. **Scope the token to least privilege**, and because the SSE endpoint has no authentication of its own, keep it on your LAN or behind Tailscale rather than exposing it directly to the internet.

## Tailscale

This template supports Unraid's built-in (Unraid 7+) Tailscale Docker integration, so the Grafana MCP container can join your tailnet as its own device — reach it privately from anywhere (e.g. Claude Desktop on a laptop) with no port forwarding.

**Enable it:**

1. Open the **Grafana-MCP** container in **Docker** and switch to **Advanced View**.
2. Toggle **Use Tailscale** on, set a unique **Tailscale hostname**, and configure any extras as needed.
3. **Apply**.
4. Open the container log and click the **authentication link** to approve the container on your tailnet.

**How it persists:** the MCP server is stateless, so this template maps a small **Appdata** volume for the sole purpose of holding Tailscale state, and declares:

```xml
<TailscaleStateDir>/appdata/.tailscale_state</TailscaleStateDir>
```

Because that path is inside a mapped persistent volume, Tailscale's machine key and node identity survive container recreation and image updates — and you avoid the common Unraid error:

```
ERROR: Couldn't detect persistent Docker directory for .tailscale_state!
```

The upstream image already ships `ca-certificates`, which the Unraid Tailscale hook needs to download the Tailscale binary — so the injection works without any changes to the image.

> [!TIP]
> Once on your tailnet, point your MCP client at the Tailscale hostname instead of the LAN IP, e.g. `http://grafana-mcp.<your-tailnet>.ts.net:8000/sse`.

## How this repo is built

- **`templates/mcp-grafana.xml`** — the Community Applications template.
- **`scripts/validate_unraid_ca_templates.py`** — heuristic CA-policy validator. It checks for shell-injection characters, HTML in `Overview`/`Description`, affiliate links, and that `<TailscaleStateDir>` is declared inside a mapped volume. Run locally with `python3 scripts/validate_unraid_ca_templates.py`.
- **`.github/workflows/lint.yml`** — runs the validator on every push and PR.
- **`.github/workflows/release-please.yml`** — opens a release PR on conventional-commit `feat:`/`fix:` and cuts a tagged GitHub Release when merged.

## Support

Open an issue: https://github.com/paulmeier/mcp-grafana-unraid/issues
