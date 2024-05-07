# CLI

Command-line interface executables can be found under the `bin/` directory and
at the time of writing are:

1. `rkn` -- Direct scanning utility.
2. `rkn_shell` -- Starts a Bash shell under the package environment.
3. `rkn_system_info` -- Presents system information about the host.

## Scanning and reporting

To start a scan and save the report as JSON:

    bin/rkn https://ginandjuice.shop/ --report-save-path=report.json
