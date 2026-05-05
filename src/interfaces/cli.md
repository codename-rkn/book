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
