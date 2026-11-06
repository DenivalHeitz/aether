# Aether

A local, self-hosted web search and page-fetching [MCP](https://modelcontextprotocol.io/) server.

Aether exposes two tools to any MCP-compatible client:

- **`web_search`** — query a self-hosted [SearXNG](https://docs.searxng.org/) instance and return titles, URLs, snippets, and publication dates.
- **`fetch_page`** — download a public webpage and extract its readable text.

Everything runs on your machine: SearXNG handles the searching, and Aether talks to it over `localhost`. No API keys, no third-party search subscriptions.

## Features

- Self-hosted meta-search via SearXNG (Docker)
- Plain-text extraction from HTML pages with `BeautifulSoup`
- Search results capped at 10 per query; page text capped at 12,000 characters
- `fetch_page` refuses non-public hosts (loopback, private, link-local, etc.), credentials in URLs, and redirects
- Hard 1 MB response limit for fetched pages
- Returned page content is clearly flagged as untrusted data

## Architecture

```
MCP client  ──stdio──▶  server.py (Aether)  ──HTTP──▶  SearXNG (Docker)
                                                          │
                                                          ▼
                                                   upstream search
                                                      engines
```

`server.py` is both the MCP server and the SearXNG client. It runs locally over the stdio transport, so your MCP client launches it as a child process.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with the Docker Compose plugin
- Python **3.10+**
- `git`
- An MCP-compatible client (Claude Desktop, Claude Code, OpenCode, etc.)

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/DenivalHeitz/aether.git
cd aether
```

### 2. Create the environment file

```bash
cp .env.example .env
```

Set `SEARXNG_SECRET` to a random value:

```bash
sed -i "s/change-me/$(openssl rand -hex 32)/" .env
```

Or edit `.env` by hand:

```dotenv
SEARXNG_SECRET=your-random-secret-here
```

### 3. Start SearXNG

```bash
docker compose up -d
```

Check that it is running:

```bash
docker compose ps
```

### 4. Verify the SearXNG JSON API

Aether requires SearXNG's JSON output format, which is already enabled in [searxng/settings.yml](searxng/settings.yml).

```bash
curl "http://127.0.0.1:8080/search?q=test&format=json" | head -c 300
```

You should get back JSON containing a `results` array. If you get HTML or a 403, confirm the `search.formats` list in `searxng/settings.yml` includes `json` and restart the container:

```bash
docker compose restart
```

### 5. Install the Python dependencies

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

On Windows, activate with `.venv\Scripts\activate` instead.

If you only need to run the server and don't need the MCP Inspector, `pip install -e .` is enough.

`pyproject.toml` pins the MCP SDK to the **1.x** line (`mcp>=1.30,<2`), which is the version that provides the FastMCP 1.x API used by `server.py`. The `dev` extra adds the `mcp` CLI for `mcp dev`. Do not upgrade to `mcp` 2.x without updating the code.

### 6. Test the server (optional)

`server.py` uses stdio, so running it directly just waits for an MCP client. To poke at the tools interactively, use the MCP Inspector:

```bash
mcp dev server.py
```

### 7. Connect your MCP client

Point the client at the **absolute path** of the virtualenv's Python and `server.py`.

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "aether": {
      "command": "/absolute/path/to/aether/.venv/bin/python",
      "args": ["/absolute/path/to/aether/server.py"]
    }
  }
}
```

**Claude Code**:

```bash
claude mcp add aether -- /absolute/path/to/aether/.venv/bin/python /absolute/path/to/aether/server.py
```

**OpenCode** (`opencode.json`):


```json
{
  "mcp": {
    "servers": {
      "local-web-search": {
        "type": "local",
        "command": [
          "/absolute/path/to/aether/.venv/bin/python",
          "/absolute/path/to/aether/server.py"
        ],
        "environment": {
          "SEARXNG_URL": "http://127.0.0.1:8080"
        },
        "codemode": false // codemode not needed for this mcp tool
      }
    }
  }
}
```

```
```
Restart your client after saving the config.

## Usage

Once connected, ask your client something that needs the live web, for example:

> Search the web for the latest Python release notes.

The client will call `web_search`, then optionally call `fetch_page` on a promising result to read it in full.

### Tools

#### `web_search`

| Parameter | Type   | Default | Description                                   |
| --------- | ------ | ------- | --------------------------------------------- |
| `query`   | string | —       | Search terms. Empty queries are rejected.     |
| `limit`   | int    | `5`     | Number of results, clamped to the range 1–10.  |

Returns:

```json
{
  "query": "latest python release",
  "count": 2,
  "results": [
    {
      "title": "Python 3.14 release notes",
      "url": "https://docs.python.org/3/whatsnew/3.14.html",
      "snippet": "...",
      "published_date": null
    }
  ]
}
```

#### `fetch_page`

| Parameter | Type   | Description                        |
| --------- | ------ | ---------------------------------- |
| `url`     | string | Public `http`/`https` webpage URL. |

Returns `url`, `title`, `text`, `truncated`, and a `warning` noting that page content is untrusted. Failures return `{"error": "..."}`.

## Configuration

| Variable       | Default                 | Used by        | Description                                                        |
| -------------- | ----------------------- | -------------- | ------------------------------------------------------------------ |
| `SEARXNG_URL`  | `http://127.0.0.1:8080` | `server.py`    | Base URL of the SearXNG instance. Set it if you changed the port.  |
| `SEARXNG_SECRET` | —                     | Docker Compose | Secret key for SearXNG. Required; Compose fails without it.        |

If SearXNG runs elsewhere, pass `SEARXNG_URL` through your client's MCP config, for example:

```json
{
  "mcpServers": {
    "aether": {
      "command": "/absolute/path/to/aether/.venv/bin/python",
      "args": ["/absolute/path/to/aether/server.py"],
      "env": { "SEARXNG_URL": "http://127.0.0.1:8080" }
    }
  }
}
```

Tunable limits live at the top of `server.py` (`TIMEOUT`, `MAX_PAGE_BYTES`, `MAX_TEXT_CHARS`, `MAX_CONNECT_ADDRESSES`).

## Security notes

- SearXNG is bound to `127.0.0.1` only and is not exposed to your network.
- `.env` is git-ignored; only `.env.example` is committed.
- `fetch_page` resolves the hostname once, rejects every non-public address, and then connects to that validated IP while preserving the original `Host` header and TLS SNI. This prevents DNS rebinding between the check and the request.
- IPv6 addresses must be in global unicast space (`2000::/3`), which also blocks transition ranges such as NAT64 that embed an IPv4 address.
- Redirects are not followed, so a public URL cannot bounce to a private one.
- HTTP proxy environment variables are ignored (`trust_env=False`), so searches and fetches do not silently leave through a configured proxy.
- Fetched page text is untrusted input and is returned with an explicit warning. Do not treat it as agent instructions.

## Stopping and cleaning up

```bash
docker compose down       # stop SearXNG
docker compose down -v    # also remove Docker volumes
```

## Troubleshooting

| Symptom | Fix |
| ------- | --- |
| `set SEARXNG_SECRET in .env` | Create `.env` from `.env.example` and set a value. |
| `Search failed: HTTPStatusError` | SearXNG is not reachable or returned an error. Check `docker compose ps` and `docker compose logs searxng`. |
| Search returns `403` | JSON format is disabled. Ensure `json` is in `search.formats` in `searxng/settings.yml`, then restart the container. |
| Agent can't see the tools | The client is probably pointing at the wrong Python or `server.py` path. Use absolute paths and restart the client. |
| `Fetch failed` / `Host is not public` | The URL is unreachable, redirects, or resolves to a private address — all rejected by design. |

## Project structure

```
.
├── docker-compose.yml     # SearXNG service definition
├── pyproject.toml         # Project metadata and deps (MCP pinned to 1.x)
├── server.py              # MCP server with web_search and fetch_page
├── searxng/
│   └── settings.yml       # SearXNG config (enables JSON output)
├── .env.example           # Template for local secrets
└── README.md
```

## License

MIT — see [LICENSE](LICENSE).
