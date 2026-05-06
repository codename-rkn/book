# CLI

Command-line interface executables can be found under the `bin/` directory and
at the time of writing are:

1. `apex` -- Direct scanning utility.
2. `apex_mcp_server` -- Starts an [MCP](./mcp/index.md) server daemon.
3. `apex_shell` -- Starts a Bash shell under the package environment.
4. `apex_system_info` -- Presents system information about the host.

## Scanning and reporting

To start a scan and save the report as JSON:

    bin/apex https://ginandjuice.shop/ --report-save-path=report.json

## Sink filter

Apex traces five kinds of [sink](./mcp/index.md#resources) — `active`,
`body`, `header_name`, `header_value`, and `blind`. The CLI logs the
first four by default and skips `blind` (input reaches a sink with
no observable response signal — useful for targeted recon, noisy as
a default). Hits at filtered-out sinks are skipped at trace time
and never enter the report.

Override with `--sink-filter NAME` (repeatable, OR-ed):

    # Only exec-context findings
    bin/apex https://example.com/ --sink-filter active

    # Default four PLUS blind
    bin/apex https://example.com/ \
        --sink-filter active --sink-filter body \
        --sink-filter header_name --sink-filter header_value \
        --sink-filter blind

The first explicit `--sink-filter` clears the default; subsequent
ones accumulate. The CLI prints which sinks are loaded at scan start:

```
 [~] Recording sinks (default — pass `--sink-filter blind` to include blind hits): active, body, header_name, header_value
```

To run a crawl-only pass (sitemap, no entries), there is no CLI
flag yet — use the [profile UI](./web/index.md) or the MCP
`sinks_filter: []` shape.
