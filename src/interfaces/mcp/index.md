# MCP

Apex Recon ships a [Model Context Protocol][mcp] server so an AI client
(Claude Desktop / Code, Cursor, Continue — anything that speaks MCP) can
drive scans directly: spawn an _Instance_, watch its progress, fetch
entries and reports, and tear it down again — over a single HTTP
endpoint.

It is the same conceptual surface as the [REST API](../rest-api/index.md),
but exposed as MCP _tools_, _prompts_, and _resources_, and described to
the client via the protocol's own discovery calls (`tools/list`,
`prompts/list`, `resources/list`). Whatever the model sees in its
context is exactly what the surface advertises — the descriptions _are_
the docs.

This page is the canonical reference. It is the only document an AI
needs to understand and drive the surface end to end; everything else
in this section either complements it or provides language bindings.

[mcp]: https://modelcontextprotocol.io/

## Server

To start the MCP server:

```bash
bin/apex_mcp_server
```

To see CLI options:

```bash
bin/apex_mcp_server -h
```

The transport is [Streamable HTTP][http] — every call is a JSON-RPC
`POST`, optionally upgraded to a Server-Sent Events stream by the
server. Authentication is configured in-application (see _Auth_ below);
there are no `--username` / `--password` flags.

[http]: https://modelcontextprotocol.io/docs/concepts/transports#streamable-http

## Endpoint

A single URL — `http://<host>:<port>/mcp`. There is no per-instance
sub-route; instance scoping is done by passing `instance_id` as an
argument to every per-scan tool. One MCP server, one session per client.

`serverInfo` advertises `{ name: "apex", version: <RKN.version> }`,
derived from the running umbrella's `shortname` / `version` methods.
The brand and version are picked up automatically — there's nothing to
configure on the CLI.

## Tools

The server flattens framework + scan tools into one `tools/list`
response. Every tool that returns structured data declares an
`outputSchema`; the response carries _both_ `content[0].text`
(JSON-encoded, for clients that don't speak typed outputs) _and_
`structuredContent` matching the schema (for clients that do).

### Framework tools

| Tool             | Required        | Optional                | Returns (`structuredContent`) |
|------------------|-----------------|-------------------------|-------------------------------|
| `list_instances` | —               | —                       | `{ instances: { <id>: { url } } }` |
| `spawn_instance` | —               | `options`, `start=true` | `{ instance_id, url }`        |
| `kill_instance`  | `instance_id`   | —                       | `{ killed: <id> }`            |

`spawn_instance.options` is forwarded to `instance.run(...)` — same
shape as the [REST API](../rest-api/index.md) `POST /instances` body.
To spawn an _Instance_ without running anything, pass `start: false`;
passing `options: {}` does **not** skip the run.

For the full options surface, read the
[`apex://options/reference`](#resources) resource (covered below) or
the [REST API options reference](../rest-api/index.md#scan-options).

### Per-scan tools

Every per-scan tool requires `instance_id`. Two delta-arg shapes:

- `*_seen` — array of entry digests already processed; the response
  excludes those.
- `*_since` — integer offset; the response is the tail past that index.

| Tool            | Required        | Optional                                                                                       | Returns                              |
|-----------------|-----------------|------------------------------------------------------------------------------------------------|--------------------------------------|
| `scan_progress` | `instance_id`   | `entries_seen`, `errors_since`, `sitemap_since`, `without_entries`, `without_errors`, `without_sitemap`, `without_statistics` | `{ status, running, seed, statistics?, entries?, errors?, sitemap?, messages }` |
| `scan_report`   | `instance_id`   | —                                                                                              | `{ entries, sitemap, statistics }`   |
| `scan_sitemap`  | `instance_id`   | `sitemap_since=0`                                                                              | `{ sitemap: { <url>: <code> } }`     |
| `scan_entries`  | `instance_id`   | `entries_seen=[]`                                                                              | `{ entries: { <digest>: <entry> } }` |
| `scan_errors`   | `instance_id`   | `errors_since=0`                                                                               | `{ errors: [string] }`               |
| `scan_pause`    | `instance_id`   | —                                                                                              | `{ status: 'paused' }`               |
| `scan_resume`   | `instance_id`   | —                                                                                              | `{ status: 'resumed' }`              |
| `scan_abort`    | `instance_id`   | —                                                                                              | `{ status: 'aborted' }`              |

### Entry digests

Entry `digest` values are the **keys** of the returned `entries` hash
(NOT a field nested inside the value) — 32-bit integers. Both
`scan_progress` and `scan_entries` accept the digest array as integers
or numeric strings (some JSON-RPC clients stringify large numbers); the
server coerces. If you ever see the same entry stream back unchanged
after passing it as `entries_seen`, a stringified-vs-int mismatch is
the first thing to check.

## Prompts

| Prompt               | Required | Description                                                                                  |
|----------------------|----------|----------------------------------------------------------------------------------------------|
| `quick_scan(url)`    | `url`    | Canned operator workflow for the **bounded smoke test** — expands into a 6-step user message that walks the AI through reading the options reference, building `options` from the quick-scan preset (`scope.page_limit: 50` baked in), `spawn_instance`, polling `scan_progress` every 5 s using deltas, fetching `scan_entries` when status reaches `done`, and `kill_instance`-ing afterwards. Optional args: `page_limit` (override the cap), `authorized_by`, `extra_options`. |
| `full_scan(url)`     | `url`    | Same shape as `quick_scan` minus the 50-page cap — drives a complete recon using the full-scan preset. Use when you want a thorough run and accept hours of polling. Optional args: `authorized_by`, `extra_options`. |

The expanded prompt body references resources by URI so the model has
a clear pull path for the data — it doesn't need to memorise option
names.

## Resources

| URI                                    | Mime               | Contents                                                                          |
|----------------------------------------|--------------------|-----------------------------------------------------------------------------------|
| `apex://glossary`                      | `text/markdown`    | Domain terms (entry, digest, sink, mutation, action, platforms, status, sitemap, statistics). Read once before driving a scan. |
| `apex://options/reference`             | `text/markdown`    | Concrete keys for `spawn_instance.options` (url, scope, audit, http, browser_cluster, plugins, authorized_by, sinks_filter). Apex auto-loads its sink-trace checks — no `checks` key. |
| `apex://option-presets/quick-scan`     | `application/json` | JSON template — every audit element traced, default plugins, the four-kind `sinks_filter` default (every sink except `blind`), **`scope.page_limit: 50`** so a real-site smoke test finishes in minutes. Bump / drop the cap (or switch to `full-scan`) for a longer recon. |
| `apex://option-presets/full-scan`      | `application/json` | Same shape as `quick-scan` minus the page cap — uncapped recon. Use when you want a complete run and accept a long wait. |

Quick-scan preset:

```json
{
  "url":          "<TARGET URL>",
  "audit":        { "elements": ["links","forms","cookies","headers","ui_inputs","ui_forms","jsons","xmls"] },
  "sinks_filter": ["active","body","header_name","header_value"],
  "scope":        { "page_limit": 50 }
}
```

Full-scan preset (same minus `scope`):

```json
{
  "url":          "<TARGET URL>",
  "audit":        { "elements": ["links","forms","cookies","headers","ui_inputs","ui_forms","jsons","xmls"] },
  "sinks_filter": ["active","body","header_name","header_value"]
}
```

Pulled in-band, this gives an AI client everything it needs to
schematise `spawn_instance.options` without leaving the protocol.

## Auth

Authentication is opt-in and lives on the application class:

```ruby
RKN::Application.mcp_authenticate_with do |bearer_token|
    User.find_by(api_token: bearer_token)
end
```

When a validator is registered the server requires
`Authorization: Bearer <token>` on every request and returns `401`
otherwise (RFC 6750 — `WWW-Authenticate: Bearer realm="MCP", error=…`).
Without a validator the server accepts unauthenticated traffic — fine
for a loopback bind, dangerous on a public interface.

The resolved principal is stashed at `env['cuboid.mcp.auth']` for any
downstream middleware that wants to look it up.

## Self-discovery flow

If you're an AI seeing this server for the first time, do this once:

1. `initialize` → check `serverInfo.name` (`apex`) and `version`.
2. `resources/list` → you'll see four URIs. **Read all four** — they
   are tiny and answer most of the questions you'd otherwise have to
   ask. The glossary in particular grounds the field names you'll see
   in `scan_progress` / `scan_entries` results (sink, mutation, action,
   platforms).
3. `prompts/list` → you'll see `quick_scan` (capped 50-page smoke
   test) and `full_scan` (uncapped). If the user's intent matches one
   ("recon this URL for active inputs"), use it: `prompts/get` with
   their URL gives you a full operator script.
4. `tools/list` → discover the 11 tools. `outputSchema` on each tells
   you exactly what `structuredContent` to expect.

After that, drive the scan with no further out-of-band knowledge.

## Status semantics

`scan_progress.status` advances roughly:

```
ready ──► preparing ──► scanning ──► auditing ──► cleanup ──► done
                              │           │
                              └─► paused ─┘
                              │
                              └─► aborted (terminal)
```

- `ready` — the _Instance_ has been spawned but `start: true` hasn't yet
  flipped it past `instance.run(...)`. **`scan_progress` called on a
  `:ready` instance returns a minimal payload** (status + running +
  seed only — no statistics yet, no entries hash). Don't trust delta
  arithmetic until status has advanced.
- `preparing` — engine is loading checks/plugins, opening the seed URL,
  and warming the browser cluster. No entries yet, but the sitemap may
  start populating.
- `scanning` — crawl is in flight; new sitemap entries appear, no
  audits running yet.
- `auditing` — the crawl is winding down and sink-tracing is firing
  against discovered inputs. Most entries land here.
- `paused` / `aborted` — `running: false`, but only `aborted` is
  terminal. A paused scan can be resumed with `scan_resume`.
- `cleanup` — engine is finalising state; close to `done`.
- `done` — terminal. `scan_report` is now safe to call;
  `running: false`.

**Treat anything other than `done` / `aborted` as still in flight.**

## Polling cadence

5 seconds is the default cadence the `quick_scan` prompt suggests, and
it's a sensible floor:

- Faster than ~2 s burns context tokens for almost no new state.
- `scan_progress` with `without_statistics: true` is cheap; the
  `statistics` block dwarfs the rest of the payload.
- Use `errors_since` / `sitemap_since` / `entries_seen` from the second
  poll onwards — the engine returns only deltas, keeping each response
  small.
- For very long scans (hours), 30 s is fine.

### Delta-arg shapes — when to use which

| Field          | Shape                          | Why                                                       |
|----------------|--------------------------------|-----------------------------------------------------------|
| `entries_seen` | array of digests (int / string) | Entries are content-addressed; offsets aren't stable across deduplication. |
| `errors_since` | integer offset                 | Engine errors are an append-only log.                     |
| `sitemap_since`| integer offset                 | Sitemap is discovery-ordered, append-only.                |

`scan_progress` accepts all three at once — gives you exactly the
right delta for each block.

## Instance lifetime

Every `spawn_instance` forks a daemonised SCNR engine subprocess (Apex
runs on the same engine as Spectre, configured for sink-trace recon).
The `instance_id` is the engine's RPC token. Things to know:

- The instance survives a client disconnect. If you forget to call
  `kill_instance`, the process keeps running until something kills it
  (host shutdown, OOM, manual signal). Always wire a `kill_instance`
  in your error path.
- The instance does **not** survive an MCP-server restart cleanly. The
  daemonised engine keeps running but the MCP server's in-memory
  `@@instances` map is empty after a restart, so you can't
  `kill_instance` it through MCP any more (you'd need REST or a
  process-level kill). **Don't restart the MCP server while scans are
  mid-flight.**
- Each instance reserves about 2 GB RAM and 4 GB disk by default. On a
  laptop, parallel scans are bounded by RAM; the host won't proactively
  refuse a third spawn if the second one is still warming up.
- `start: false` is rare in practice. It registers an idle instance
  that sits there waiting for a `run`, and the only way to `run` is
  via REST/RPC — MCP's `spawn_instance` doesn't have a separate "start
  now" tool. Use it when something else is going to drive the run.

## Error idiom

Engine exceptions don't crash the MCP server — `MCPProxy.instrumented_call`
wraps every body with `rescue => e`. The wire response is:

```jsonc
{
  "result": {
    "isError": true,
    "content": [
      { "type": "text", "text": "error: SCNR::Engine::Options::Error: …" }
    ]
  }
}
```

Common shapes:

- `error: ArgumentError: Invalid options!` — `instance.run(options)`
  rejected the shape. Read `apex://options/reference` and try again.
- `error: Toq::Exceptions::RemoteException: …` — the inner RPC client
  to the engine subprocess raised. Usually means the engine itself is
  in a bad state. Try `scan_errors` for clues; if that's empty,
  `kill_instance` and respawn.
- `error: JSON::GeneratorError: "\xNN" from ASCII-8BIT to UTF-8` — the
  engine produced binary bytes that aren't valid UTF-8 (a response
  body, HTTP header, etc.). Affects `scan_report` more than the
  streaming tools. Skip the report; `scan_progress` + `scan_entries`
  will still work.
- `unknown instance: …` — the `instance_id` you passed isn't in the
  server's local map. Either the MCP server was restarted (which
  clears `@@instances`), or the id is stale. Re-`spawn_instance`.

Validation errors (missing required arg, type mismatch) come back
through the JSON-RPC error envelope, not as a tool error:

```json
{ "error": { "code": -32602, "message": "Missing required arguments: instance_id" } }
```

## Options trivia

- Apex auto-loads its two sink-tracing checks (`sink_trace_force`,
  `sink_trace_force_dom`) automatically — you do **not** pass `checks`.
  Don't try to override; the recon flow depends on these being the
  active set.
- `plugins: ["defaults/*"]` loads every plugin under the `defaults`
  directory. Empty array (or omitted key) loads none.
- `audit.elements` defaults to all kinds when the key is omitted, which
  is what the CLI does. Pass an explicit list to restrict — e.g.
  `["links", "forms"]` skips cookies, headers, JSON/XML bodies, etc.
- `scope.page_limit` is baked into the quick-scan preset at **50** — a
  real-site smoke test that finishes in minutes. Override the
  `page_limit` prompt arg (or the JSON directly) for a smaller / larger
  cap; switch to the `full-scan` preset (or the `full_scan` prompt) for
  an uncapped recon. Sensible explicit values: 30 (smaller smoke test),
  200 (representative).
- `authorized_by` — set this to the operator's email; it shows up in
  the engine's outbound HTTP `From` header so target-site admins can
  identify the scan. Not required, but polite on third-party targets.
- `sinks_filter` — record-time whitelist of sink kinds (mirrors the
  CLI's `--sink-filter`). Four states:
  - **omit the key** → MCP server applies the default
    `["active", "body", "header_name", "header_value"]` (every sink
    kind except `blind`). Mirrors the `apex_scan` CLI default.
  - `null` → no filter, log every sink kind including `blind`.
  - `[]` (empty array) → log NOTHING. The engine still crawls and
    produces a sitemap; no entries get recorded. Useful for a pure
    discovery pass.
  - populated array (e.g. `["active"]`) → whitelist that subset.
  Hits at filtered-out sinks are skipped at trace time and never
  enter `scan_entries` / `scan_report`. See `apex://options/reference`
  for the on-the-wire description.

## Conventions baked into the descriptions

The tool / prompt / resource descriptions are deliberately
self-grounding:

- Per-property descriptions on every tool argument (no buried-in-text
  args).
- Cross-references use namespaced names (`scan_resume`, not `resume`)
  so the AI can call them verbatim.
- Preconditions are stated where they exist (`scan_pause` "the scan
  must currently be running", `scan_resume` "must have been paused via
  `scan_pause`"). Calling out of order returns an MCP tool error
  rather than a routing failure.
- Domain terms (sink, mutation, action, platforms, digest) are defined
  in `apex://glossary` and cross-referenced from the relevant
  `outputSchema` property descriptions, so a model parsing
  `structuredContent` can resolve any unknown field name back to the
  glossary in one hop.

## Things the protocol doesn't expose yet

For honesty — places where you'd still need out-of-band knowledge:

- **Live progress streaming.** The MCP spec supports
  `notifications/progress` for long-running operations; this server
  doesn't emit them yet. You poll.
- **Structured error codes.** Errors come back as text. If you want to
  branch on "bad option key" vs "engine crashed" vs "auth failed",
  you're parsing the text.
- **Plugin / sink catalogue.** There's no `list_plugins` /
  `list_sinks` tool; if a user asks "which sinks does Apex trace", you
  need out-of-band knowledge or to inspect the engine source.

Each of those is on the roadmap. Until they land, the resources +
prompt expansion are the supported way to ground a model.

## Connecting an MCP client

Most clients accept a Streamable HTTP server entry verbatim:

```jsonc
{
  "mcpServers": {
    "apex": {
      "url": "http://127.0.0.1:7331/mcp"
    }
  }
}
```

That's all. After `initialize`, the client sees:

- 11 tools (3 framework + 8 per-scan), each with input + output
  schema.
- 2 prompts (`quick_scan`, `full_scan`).
- 4 resources.

If your client only speaks stdio (older Claude Desktop builds), use
any community stdio↔HTTP MCP bridge in front. Cursor, Claude Code,
and Continue speak Streamable HTTP natively.

## End-to-end example — curl

Initialize, capture the session id, acknowledge:

```bash
curl -i -X POST http://127.0.0.1:7331/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  --data '{
    "jsonrpc": "2.0", "id": 1, "method": "initialize",
    "params": {
      "protocolVersion": "2025-06-18",
      "capabilities":    {},
      "clientInfo":      { "name": "curl", "version": "0" }
    }
  }'
# → response header: Mcp-Session-Id: <SID>

curl -X POST http://127.0.0.1:7331/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: $SID" \
  --data '{ "jsonrpc": "2.0", "method": "notifications/initialized" }'
```

Spawn a scan against `http://testfire.net/` using the quick-scan
defaults:

```bash
curl -X POST http://127.0.0.1:7331/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: $SID" \
  --data '{
    "jsonrpc": "2.0", "id": 2, "method": "tools/call",
    "params": {
      "name": "spawn_instance",
      "arguments": {
        "options": {
          "url":     "http://testfire.net/",
          "plugins": ["defaults/*"]
        },
        "start": true
      }
    }
  }'
# → result.structuredContent: { instance_id, url }
```

Poll progress, fetching deltas only after the first call:

```bash
curl -X POST http://127.0.0.1:7331/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: $SID" \
  --data '{
    "jsonrpc": "2.0", "id": 3, "method": "tools/call",
    "params": {
      "name": "scan_progress",
      "arguments": {
        "instance_id":         "'$IID'",
        "entries_seen":        [3162940604, 1457298731],
        "errors_since":        12,
        "sitemap_since":       37,
        "without_statistics":  true
      }
    }
  }'
```

Fetch entries and tear down:

```bash
curl -X POST http://127.0.0.1:7331/mcp ... \
  --data '{ "jsonrpc": "2.0", "id": 4, "method": "tools/call",
            "params": { "name": "scan_entries",
                        "arguments": { "instance_id": "'$IID'" } } }'

curl -X POST http://127.0.0.1:7331/mcp ... \
  --data '{ "jsonrpc": "2.0", "id": 5, "method": "tools/call",
            "params": { "name": "kill_instance",
                        "arguments": { "instance_id": "'$IID'" } } }'
```

The same loop expressed as a `quick_scan` prompt expansion is one
`prompts/get` call away.
