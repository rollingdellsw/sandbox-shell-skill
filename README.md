# Sandbox Shell Skill

A Koi™ Assistant skill providing sandboxed shell execution environment.

## What It Is

The Sandbox Shell Skill enables LLM sessions to execute terminal commands, modify code, and inspect repositories directly on user's local machine, but inside an isolated environment.
It combines host shell execution with a LSP (Language Server Protocol) code-intelligence engine for symbol navigation, AST searching, and diagnostics without risking unwanted modifications to the user's host environment.

## Implementation

* **OverlayFS / Bubblewrap Isolation**: Uses `bwrap` + OverlayFS on Linux (or APFS copy-on-write clones on macOS). The host filesystem remains strictly read-only,
    with all session writes directed to temporary overlay directories (`~/.koi/sandbox/...`). The overlay filesystem size is also capped.
* **Gateway & MCP Architecture**: Runs as a Model Context Protocol (MCP) server (`sandbox-shell-mcp.mjs`) behind `koi-gateway.js` on user's local machine, connecting to Koi Assistant via a WebSocket-to-stdio bridge.
* **Embedded Code Intelligence (`lsp_search`)**: Spawns an embedded language server manager directly inside the MCP server, re-exporting tools across three tiers:
  * **Semantic**: Language Server Protocol (LSP) for signatures, definition lookup, implementations, and diagnostics.
  * **Structural**: `ast-grep` (tree-sitter) pattern matching for syntax-aware code searches and declaration reading.
  * **Text**: `ripgrep` (`rg`) fast string matching.

## Security Model

* **Configurable Credential Masking**: Masks sensitive paths (such as `~/.ssh` or `~/.aws`) using tmpfs mounts or `/dev/null` binds, driven by `sandbox-exclude.default` or `--exclude` flags.
* **Patch Outbox Workflow**: Agents cannot directly modify the host repository or push to remote repositories. Work is exported as git patch series (`git format-patch`) to an outbox directory for human review and apply.
* **Network Isolation**: Supports running under `--net host` for local dev-server testing or `--net loopback` for fully offline sandboxes. Direct network push is prohibited.
