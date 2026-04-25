# Trace360

Trace360 is an evidence collection engine for cloud security audits. The current
repo is centered on the `trace360` Python package, with MCP as one local
interaction interface for OpenCode and CLI/web interfaces planned alongside it.

## Current Demo

Run the local stdio MCP server:

```powershell
venv\Scripts\python.exe -m trace360.interfaces.mcp_server
```

OpenCode is configured through `opencode.json` to start that server and expose
the `get_bucket_encryption` tool.

## Controls

Audit controls live in `controls/`. The first bridge control is:

```text
controls/gcp/bucket_encryption.yaml
```

Control sets live in:

```text
controls/control_sets/
```

## GCP Access

Local development uses Google Application Default Credentials or
`GOOGLE_APPLICATION_CREDENTIALS`. Cloud Run Jobs should use the runtime service
account identity instead of key files.
