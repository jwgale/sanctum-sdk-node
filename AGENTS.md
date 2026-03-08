# AGENTS.md — SanctumAI for AI Agents

> Machine-readable integration reference. For human docs, see [README.md](README.md).

## What This Is

SanctumAI is a local credential vault. You authenticate with Ed25519 keys, request credentials through policies, and use them via proxy — you never see raw secrets.

## Quick Integration

### 1. Check if Sanctum is running

```bash
sanctum lease status
# or: curl http://localhost:7700/api/v1/health
```

### 2. MCP (Recommended)

Add to your MCP config:

```json
{
  "mcpServers": {
    "sanctum": {
      "command": "sanctum",
      "args": ["mcp", "serve"]
    }
  }
}
```

MCP config locations:
- Claude Desktop: `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS), `%APPDATA%\Claude\claude_desktop_config.json` (Windows)
- Claude Code: `.claude/mcp.json`
- Cursor: `.cursor/mcp.json`
- Windsurf: `.windsurf/mcp.json`
- VS Code: `.vscode/mcp.json`

MCP tools available: `sanctum_store`, `sanctum_retrieve`, `sanctum_list`, `sanctum_status`

### 3. SDK (Programmatic)

Install:
- Rust: `cargo add sanctum-ai`
- Python: `pip install sanctum-ai`
- Node: `npm install sanctum-ai`
- Go: `go get github.com/SanctumSec/sanctum-sdk-go`

Connect to daemon at `tcp://127.0.0.1:7600`. Protocol: JSON-RPC over TCP with 4-byte big-endian length prefix.

## CLI Reference

```bash
# Vault lifecycle
sanctum init                          # Create vault (passphrase required)
sanctum unlock                        # Unlock vault
sanctum lock                          # Lock vault (blocks all operations)

# Credentials
sanctum store <path> --type <type>    # Store credential (prompts for value)
sanctum list                          # List all credentials
sanctum delete <path>                 # Delete credential

# Agents
sanctum agent register <name>         # Register agent (generates Ed25519 keypair)
sanctum agent list                    # List agents

# Policies (deny-by-default)
sanctum policy add <name> \
  --principal "agent:<agent>" \
  --resources "<glob>" \
  --actions retrieve,use \
  --max-ttl <seconds>                 # Grant access
sanctum policy list                   # List policies
sanctum policy simulate \
  --agent <name> \
  --resource <path> \
  --action <action>                   # Dry-run evaluation

# MCP
sanctum mcp serve                     # Start MCP server (stdio)

# Daemon
sanctum daemon start                  # Start daemon + web UI on :7700
sanctum daemon stop                   # Stop daemon
sanctum lease status                  # Show active leases

# Audit
sanctum audit list                    # View audit log
sanctum audit verify                  # Verify HMAC chain integrity

# Scanner
sanctum scan                          # Scan for exposed secrets
sanctum scan --path /specific/dir     # Scan specific directory
sanctum migrate-env .env              # Import .env file into vault

# Demo
sanctum demo                          # Full end-to-end demo
```

## Credential Paths

Format: `<service>/<key-name>`

Examples:
- `openai/api-key`
- `github/token`
- `aws/secret-access-key`
- `supabase/postgres-url`

## Policy Glob Patterns

- `*` — all credentials
- `openai/*` — all OpenAI credentials
- `*/api-key` — all api-key credentials across services

## Actions

| Action | Description |
|--------|-------------|
| `retrieve` | Get raw credential value (requires `SANCTUM_ALLOW_RAW_RETRIEVAL=true`) |
| `use` | Use credential via proxy (recommended — agent never sees the secret) |
| `list` | List credential metadata |

## Use Don't Retrieve (Proxy Pattern)

**Preferred pattern.** The vault makes HTTP requests on your behalf, injecting credentials into outbound requests. You never see the secret.

### SDK: `use_credential`

```
use_credential(path, operation, params) -> result
```

### Operations

| Operation | Purpose | Key Params |
|-----------|---------|------------|
| `http_request` | Proxy HTTP request | `method`, `url`, `headers`, `body`, `header_type` |
| `http_header` | Get auth header only | `header_type` |
| `sign` | HMAC/Ed25519 signing | `algorithm`, `data` |
| `encrypt` | Encrypt data | `data` |
| `decrypt` | Decrypt data | `data` |

### `header_type` Values

| Value | Result |
|-------|--------|
| `bearer` | `Authorization: Bearer <secret>` |
| `api_key` | `X-API-Key: <secret>` |
| `basic` | `Authorization: Basic <base64>` |
| `custom` | Requires `header_name` param |

### HTTP Proxy Example (pseudo-code)

```
# 1. Authenticate
client.connect("127.0.0.1:7600")
client.authenticate("my-agent", signing_key)

# 2. Proxy a request (agent never sees the API key)
result = client.use_credential("openai/api-key", "http_request", {
    "method": "POST",
    "url": "https://api.openai.com/v1/chat/completions",
    "headers": {"Content-Type": "application/json"},
    "body": '{"model":"gpt-4","messages":[{"role":"user","content":"Hello"}]}',
    "header_type": "bearer"
})
# result = { "status": 200, "headers": {...}, "body": "..." }
```

## JSON-RPC Methods (TCP :7600)

| Method | Params | Returns |
|--------|--------|---------|
| `authenticate` | `{agent_name, challenge_response}` | `{authenticated, session_token}` |
| `retrieve` | `{path, ttl}` | `{value, lease_id, expires_at}` |
| `list` | `{}` | `{credentials: [{path, type, created_at}]}` |
| `use` | `{path, operation, params}` | `{result}` |
| `release_lease` | `{lease_id}` | `{}` |
| `lease_status` | `{}` | `{leases: [...]}` |

## HTTP Admin API (Port 7700)

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/api/v1/health` | GET | None | Health check |
| `/api/v1/credentials` | GET | Session | List credentials |
| `/api/v1/credentials` | POST | Session | Store credential |
| `/api/v1/agents` | GET | Session | List agents |
| `/api/v1/policies` | GET | Session | List policies |
| `/api/v1/audit` | GET | Session | Audit log |
| `/api/v1/leases` | GET | Session | Active leases |
| `/api/v1/proxy/leases` | POST | Agent (Ed25519) | Request credential lease |
| `/api/v1/proxy/use` | POST | Agent (Ed25519) | Proxy HTTP request |

## Authentication Flow

1. Agent sends `authenticate` with `agent_name`
2. Daemon returns a `challenge` (random bytes)
3. Agent signs challenge with Ed25519 private key
4. Agent sends signed challenge back
5. Daemon verifies signature against registered public key
6. Returns `session_token` for subsequent requests

## Error Handling

| Error | Meaning | Action |
|-------|---------|--------|
| `VAULT_LOCKED` | Vault is locked | Run `sanctum unlock` |
| `NOT_FOUND` | Credential doesn't exist | Check path with `sanctum list` |
| `POLICY_DENIED` | No policy grants access | Add policy with `sanctum policy add` |
| `LEASE_EXPIRED` | Lease TTL exceeded | Request new lease |
| `LEASE_EXHAUSTED` | Max accesses reached | Request new lease |
| `SSRF_BLOCKED` | Target URL is internal | Use external URLs only |
| `AUTH_FAILED` | Bad signature or agent | Check agent registration |

## Security Constraints

- Vault binds to `127.0.0.1` only (loopback)
- SSRF prevention blocks: `localhost`, `127.0.0.1`, `10.x`, `192.168.x`, `169.254.x`, `file://`
- Credentials encrypted at rest (AES-256-GCM, per-credential DEK)
- Keys stored in OS keychain (macOS Keychain, Windows DPAPI fallback)
- All operations audited with HMAC-SHA256 hash chain
- Deny-by-default policy engine

## File Locations

| OS | Vault Data | Config |
|----|-----------|--------|
| macOS | `~/.sanctum/` | `~/.sanctum/config.toml` |
| Linux | `~/.sanctum/` | `~/.sanctum/config.toml` |
| Windows | `%USERPROFILE%\.sanctum\` | `%USERPROFILE%\.sanctum\config.toml` |

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 7600 | TCP (JSON-RPC) | Daemon RPC — SDK/agent connections |
| 7700 | HTTP | Web dashboard + Admin API |
