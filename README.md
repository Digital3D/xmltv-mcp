# xmltv-mcp

An MCP server that gives an assistant read-only access to the French TV guide of
<https://xmltv.digital3d.com>: channels, schedules by channel and day, what is on air now, search by title,
description or credits, and the best rated programs.

## Pick your way in

| Your client | The easiest way | What you need |
|---|---|---|
| Claude Code | [Plugin](#claude-code-plugin): two commands | nothing |
| Claude Code, without a plugin | [Remote server](#claude-code-remote-server): one command | nothing |
| Claude Desktop | [Extension](#claude-desktop-extension): one file to open | Windows x64 or a Mac with Apple silicon |
| Any MCP client that sends HTTP headers | [Remote server](#any-client-remote-server): a URL and a header | nothing |
| Codex (OpenAI) | [Remote server](#codex): three lines of configuration | nothing |
| OpenAI API (Responses) | [Remote server as a tool](#openai-api): one `tools` entry | nothing |
| ChatGPT | [Only the two free lists](#chatgpt): it cannot send a key | nothing |
| Any MCP client that starts a local command | [Local server](#any-client-local-server): one block to paste | the .NET 10 SDK, or the [standalone file for Linux](#linux-without-net) |

All of them give the same seven [tools](#tools) and use the same API key. The remote server
(`https://xmltv.digital3d.com/mcp`) needs no installation; the local server runs on your machine and keeps a
cache that saves requests (see [Local or remote](#local-or-remote)).

## Getting a key

1. Create an account at <https://digital3d.com/Register>.
2. Copy your API key (a GUID) from <https://digital3d.com/MyAccount>.
3. Give it to the server the way your client asks for it (sections below).

**The free key gives 10 requests per day** (`get_quota` tells the exact limit, for free). To go beyond, take the
XMLTV subscription at <https://digital3d.com/XmltvAbo> (the API only), or the PRO subscription at
<https://digital3d.com/StripeAbo> (a higher limit plus all Digital3D desktop apps). A subscription applies from
the next request, without waiting for the daily reset.

Two tools (`list_channels`, `list_categories`) work without a key, so you can start without one and add it
later. When a tool needs the key, or when the daily quota runs out, its answer says what to do and the assistant
relays it.

## Claude Code: plugin

```
/plugin marketplace add digital3d/xmltv-mcp
/plugin install xmltv@digital3d
```

Claude Code asks for the API key when the plugin is enabled and keeps it in its secure storage; leave it empty
to use only the two free lists. The plugin connects to the remote server: nothing else to install.

## Claude Code: remote server

```bash
claude mcp add --transport http xmltv https://xmltv.digital3d.com/mcp --header "Authorization: Bearer <your-api-key>"
```

Without `--header`, only the two free lists work. To change the key: `claude mcp remove xmltv`, then add it
again.

## Claude Desktop: extension

1. Download `xmltv-mcp-<version>-win-x64.mcpb` (Windows) or `xmltv-mcp-<version>-osx-arm64.mcpb` (Mac with Apple
   silicon) from <https://github.com/Digital3D/xmltv-mcp/releases>.
2. Open it (or Settings → Extensions → Install extension), then type your API key in the form.

The extension carries the local server: neither .NET nor a terminal is needed. Claude Desktop exists for Windows
and macOS only, hence no Linux file here. On an Intel Mac, use
the [local server](#any-client-local-server) block in `claude_desktop_config.json` (Settings → Developer → Edit
Config) and restart Claude Desktop.

A connector added by URL in Claude Desktop or on claude.ai cannot send your key (those connectors sign in with
OAuth, which this server does not offer): it would only give the two free lists.

## Any client: remote server

Streamable HTTP, no session, nothing to install:

```json
{
  "mcpServers": {
    "xmltv": {
      "type": "http",
      "url": "https://xmltv.digital3d.com/mcp",
      "headers": { "Authorization": "Bearer <your-api-key>" }
    }
  }
}
```

The names of the fields depend on the client; what matters is the URL and the `Authorization: Bearer` header,
the same key as the REST API. Without the header, only the two free lists work. The server never answers 401:
a missing or refused key is explained in the result of the tool.

## Codex

In `~/.codex/config.toml`:

```toml
[mcp_servers.xmltv]
url = "https://xmltv.digital3d.com/mcp"
bearer_token_env_var = "XMLTV_API_KEY"
```

`bearer_token_env_var` is the name of an environment variable, not the key: set `XMLTV_API_KEY` to your key
before starting Codex. The local server works too (requires the .NET 10 SDK, keeps a cache):

```toml
[mcp_servers.xmltv]
command = "dotnet"
args = ["dnx", "XmltvDigital3d.Mcp", "--yes"]
env_vars = ["XMLTV_API_KEY"]
```

Both checked with Codex CLI 0.162.1 on 2026-10-11.

## OpenAI API

MCP is an open standard: the Responses API of OpenAI takes the remote server as a tool, with your key in
`headers`.

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.4-mini",
    "tools": [{
      "type": "mcp",
      "server_label": "xmltv",
      "server_url": "https://xmltv.digital3d.com/mcp",
      "headers": { "Authorization": "Bearer <your-api-key>" },
      "require_approval": "never"
    }],
    "input": "What is on air now on TF1?"
  }'
```

Two different keys: the one of OpenAI pays for the model, yours opens the TV guide. Without `headers`, the
model still gets the two free lists and is told how to get a key. Checked with this request body on
2026-10-11.

## ChatGPT

ChatGPT can add the server (<https://chatgpt.com/plugins>, "Add custom MCP server", server URL
`https://xmltv.digital3d.com/mcp`, "No authentication"), but it has no place for a personal key: OpenAI only
offers OAuth or no authentication there, and this server has no OAuth. So `list_channels` and `list_categories`
work, and the tools that need the key answer that it is missing. For programs in an OpenAI tool, use
[Codex](#codex) or the [API](#openai-api). The same holds for a connector added by URL on claude.ai.

## Any client: local server

Requires the [.NET 10 SDK](https://dotnet.microsoft.com/download) (the SDK, not the runtime alone). Nothing
else to install: `dotnet dnx` downloads the package from nuget.org on first use and runs it. The block is the
same on Windows, macOS and Linux:

```json
{
  "mcpServers": {
    "xmltv": {
      "command": "dotnet",
      "args": ["dnx", "XmltvDigital3d.Mcp", "--yes"],
      "env": { "XMLTV_API_KEY": "<your-api-key>" }
    }
  }
}
```

With Claude Code:

```bash
claude mcp add xmltv --env XMLTV_API_KEY=<your-api-key> -- dotnet dnx XmltvDigital3d.Mcp --yes
```

`--yes` accepts the download without a prompt, which a server started by an MCP client could not answer.
`dotnet dnx` runs the latest version; pin one with `XmltvDigital3d.Mcp@<version>`. Where the SDK put the
shorter `dnx` command on the `PATH` (Windows and macOS installers do, some Linux packages only link
`dotnet`), `dnx XmltvDigital3d.Mcp --yes` is the same thing. Restart the client (or reconnect the server) after
changing the key.

### Linux without .NET

Claude Desktop has no Linux version, so there is no `.mcpb` for Linux. For a local server without the .NET SDK,
take `xmltv-mcp-<version>-linux-x64.tar.gz` from <https://github.com/Digital3D/xmltv-mcp/releases> (from version
1.0.10): one self-contained executable. It is built for x64 Linux with glibc (not Alpine, not ARM) and needs the
`libicu` and OpenSSL libraries, present on desktop distributions.

```bash
tar -xzf xmltv-mcp-<version>-linux-x64.tar.gz
```

Then point your client to the extracted file: `"command": "/path/to/xmltv-mcp"` without `args` in the block
above, or `claude mcp add xmltv --env XMLTV_API_KEY=<your-api-key> -- /path/to/xmltv-mcp`. On Linux the
[Claude Code plugin](#claude-code-plugin) and the [remote server](#any-client-remote-server) need nothing at all.

### Installed as a dotnet tool

Instead of `dotnet dnx`, the package can be installed once as a global tool:

```bash
dotnet tool install -g XmltvDigital3d.Mcp
```

The command is then `xmltv-mcp`: use `"command": "xmltv-mcp"` without `args` in the block above, or
`claude mcp add xmltv --env XMLTV_API_KEY=<your-api-key> -- xmltv-mcp`. Update with
`dotnet tool update -g XmltvDigital3d.Mcp`, remove with `dotnet tool uninstall -g XmltvDigital3d.Mcp`.

### Variables of the local server

| Variable | Required | Meaning |
|---|---|---|
| `XMLTV_API_KEY` | for everything but the two reference lists | Your API key (a GUID). Missing or malformed: the server still starts, and the tools that need it explain what to set. |
| `XMLTV_API_BASE_URL` | no | Base URL of the API, `https://xmltv.digital3d.com` by default. Must be `https`, except on loopback; anything else stops the server at startup. |

The key is sent only to the configured host, only to the endpoints that require it, and is never written to a
log or to a tool result. The local server writes the protocol on stdout and its logs (warnings only) on stderr,
and stores nothing on disk.

## Local or remote

| | Local server (`dotnet dnx`, dotnet tool, Claude Desktop extension) | Remote server (`/mcp`, Claude Code plugin) |
|---|---|---|
| Installation | .NET 10 SDK, or the extension | none |
| An identical call repeated within a few minutes | free, answered from the local cache | one request each time |
| A response that did not change since | free (`"source": "revalidated"`) | one request |
| Quota exhausted | a response still in the cache is returned, marked `stale` | an error with the reset time |
| Per-IP limit of the API (120 requests per minute) | one per request sent to the API | up to two per tool call |

On the free key (10 requests per day), the local server makes the requests last longer.

## Tools

| Tool | What it returns | Cost |
|---|---|---|
| `list_channels` | Channels (name, id, IPTV id), optionally filtered by `search` | free, no key needed |
| `list_categories` | Program categories | free, no key needed |
| `search_programs` | Programs not started yet matching `query` in titles and descriptions (`field: "title"`) or in credits (`field: "credits"`) | one request |
| `get_channel_schedule` | Schedule of one `channel`: the program on air and the following ones across the coming days, or only those starting on `date`. Programs that are over are never returned | one request |
| `get_programs_on_air` | What is on air now, on every channel or on one | one request |
| `get_top_rated_programs` | Best rated programs, optionally by `channel`, `date`, `category`, `year` | one request |
| `get_quota` | Daily quota of the key: limit, used, remaining, reset time | free, key needed |

The program tools take `limit` (1 to 100, default 25), `offset`, and `full_description` (descriptions are cut to
300 characters unless it is `true`). Dates are days in Europe/Paris local time (`yyyy-MM-dd`); program times
carry their offset (`+01:00` or `+02:00`).

A result is one JSON text:

```json
{
  "items": [ { "title": "…", "channelId": "…", "startDateTime": "2026-01-15T20:55:00+01:00", "…": "…" } ],
  "limit": 25,
  "offset": 0,
  "hasMore": true,
  "nextOffset": 25,
  "quota": { "limit": 10, "remaining": 2, "resetAt": "2026-01-16T00:00:00Z" },
  "notice": "2 of the 10 free requests of today are left. A subscription at https://digital3d.com/XmltvAbo?src=mcp raises the daily limit.",
  "source": "api",
  "stale": false
}
```

`nextOffset` is present only when another page exists. `quota` is present only when the API answered this
call. `notice` is present only on a free key (limit of 10 or less) with 2 requests or fewer left; the assistant
relays it. `get_top_rated_programs` returns one item per work, with its next airings in `programs`.

## Cache and quota

The API counts requests per day and per key; the limit depends on your plan (`get_quota` tells, for free).
The remote server keeps no paid response: every call of a paid tool costs one request. To spend as few requests as
possible, the local server keeps the responses in memory (200 at most, lost when the server stops):

- An identical call repeated within a short window is answered from the cache without any request
  (`"source": "cache"`): 45 seconds for programs on air, 5 minutes for a search, 10 minutes for a schedule,
  15 minutes for the best rated programs, 30 minutes for the two lists.
- After that window, the server asks the API whether the response changed. An unchanged response is free
  (`"source": "revalidated"`); a changed one costs one request.
- When the quota is exhausted, a response still in the cache is returned with `"stale": true` and a message;
  anything else is an error telling when the quota resets. The free tools keep working.

On both servers:

- On the free tier,
  (10 requests per day) a result warns when 2 requests or fewer are left (`notice`), and the error of an
  exhausted quota points to the XMLTV subscription at <https://digital3d.com/XmltvAbo>, which raises the limit
  from the next request, without waiting for the reset.
- An unknown channel or category is refused by the API for free. A valid request that matches nothing costs
  one request and returns an empty `items` array.
- Neither server retries by itself. When the guide is temporarily unavailable, the error tells how long to
  wait; retrying costs one request.

## From a local .nupkg

To run a build that is not on nuget.org, install it from a local `.nupkg`:

```bash
# from a clone of the repository: pack into a folder outside of it
dotnet pack XmltvDigital3dCom.Mcp -c Release -o <folder>

# then, or from a .nupkg you were given and placed in <folder>
cd <folder>
dotnet tool install -g XmltvDigital3d.Mcp --add-source .
```

Run `dotnet tool install` from outside the clone: the `nuget.config` of the repository maps its package
sources, and `--add-source` is refused wherever such a mapping applies.

The command is then `xmltv-mcp`, as in [Installed as a dotnet tool](#installed-as-a-dotnet-tool).

## Contact

A question, a problem or a request: <https://digital3d.com/contact>.
