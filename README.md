> **AI Agents:** See [AGENTS.md](AGENTS.md) for machine-readable integration docs.

# SanctumAI Node.js SDK

[![npm version](https://img.shields.io/npm/v/sanctum-ai.svg)](https://www.npmjs.com/package/sanctum-ai)
[![Node.js](https://img.shields.io/badge/node-18%2B-brightgreen.svg)](https://nodejs.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Node.js/TypeScript SDK for [SanctumAI](https://github.com/SanctumSec/sanctum) — a local-first credential vault for AI agents.

Your agent connects to the SanctumAI daemon over JSON-RPC, authenticates with Ed25519 keys, and accesses credentials without ever holding secrets in memory.

## Installation

```bash
npm install sanctum-ai
```

Requires **Node.js 18+**.

## Quick Start

```typescript
import { SanctumClient } from "sanctum-ai";

const client = new SanctumClient("my-agent");
await client.connect(); // connects to daemon and authenticates

// Use a credential without ever seeing it
const response = await client.use("openai/api-key", "http_request", {
  method: "POST",
  url: "https://api.openai.com/v1/chat/completions",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "gpt-4",
    messages: [{ role: "user", content: "Hello" }],
  }),
  header_type: "bearer",
});

console.log(response.status); // 200
console.log(response.body);   // completion result

await client.close();
```

The agent never sees the API key. The vault injects credentials and proxies the request.

## Use Don't Retrieve

This is the core pattern. Instead of retrieving a secret and using it yourself, you tell the vault *what you want to do* and it handles the credential for you.

### Proxy an HTTP Request

The vault injects the credential into the request, makes the call, and returns the response. Your agent never touches the secret.

```typescript
const result = await client.use("openai/api-key", "http_request", {
  method: "POST",
  url: "https://api.openai.com/v1/chat/completions",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "gpt-4",
    messages: [{ role: "user", content: "Hello" }],
  }),
  header_type: "bearer", // bearer | api_key | basic | custom
});
// result => { status: 200, headers: {...}, body: "..." }
```

### Get an HTTP Header

If you need to make the request yourself but want the vault to construct the auth header:

```typescript
const result = await client.use("github/token", "http_header", {
  header_type: "bearer",
});
// result => { header_name: "Authorization", header_value: "Bearer ghp_..." }
```

### Sign Data (HMAC)

```typescript
const result = await client.use("webhook/secret", "sign", {
  algorithm: "hmac-sha256",
  data: "payload-to-sign",
});
// result => { signature: "base64-encoded-signature" }
```

### Encrypt / Decrypt

```typescript
const encrypted = await client.use("encryption/key", "encrypt", {
  plaintext: "sensitive data",
});
// encrypted => { ciphertext: "..." }

const decrypted = await client.use("encryption/key", "decrypt", {
  ciphertext: encrypted.ciphertext,
});
// decrypted => { plaintext: "sensitive data" }
```

## Connecting

```typescript
// Default: Unix socket at ~/.sanctum/vault.sock
const client = new SanctumClient("my-agent");
await client.connect();

// Custom socket path
await client.connect("/tmp/sanctum.sock");

// TCP (e.g. default daemon port 7600)
await client.connect({ host: "127.0.0.1", port: 7600 });

// Or via constructor
const client = new SanctumClient("my-agent", {
  host: "127.0.0.1",
  port: 7600,
});
await client.connect();
```

Authentication happens automatically during `connect()`. The SDK reads your agent's Ed25519 key from `~/.sanctum/keys/{agent_name}.key` and performs a challenge-response handshake with the daemon.

## Retrieving Credentials

When you need the raw credential value (e.g. for SDKs that require it), use `retrieve()`. The vault tracks access with a lease.

```typescript
const apiKey: string = await client.retrieve("anthropic/api-key");
// use apiKey with the Anthropic SDK...

// Retrieve with a TTL (seconds) — lease auto-expires
const shortLived = await client.retrieve("anthropic/api-key", 300);
```

Prefer [`use()`](#use-dont-retrieve) when possible. It keeps secrets out of agent memory entirely.

## Listing Credentials

```typescript
const credentials = await client.list();
for (const cred of credentials) {
  console.log(cred.path);
}
```

## Releasing Leases

Leases are released automatically when you call `close()`. To release early:

```typescript
// client.retrieve() returns the value; lease is tracked internally
const value = await client.retrieve("openai/api-key");

// Release all leases and disconnect
await client.close();
```

## API Reference

### `new SanctumClient(agentName, opts?)`

| Option | Type | Description |
|---|---|---|
| `socketPath` | `string` | Unix socket path (default: `~/.sanctum/vault.sock`) |
| `host` | `string` | TCP host (alternative to socket) |
| `port` | `number` | TCP port (default daemon port: `7600`) |
| `keyPath` | `string` | Path to Ed25519 key file (default: `~/.sanctum/keys/{agentName}.key`) |

### Methods

| Method | Returns | Description |
|---|---|---|
| `connect(target?)` | `Promise<this>` | Connect to daemon and authenticate |
| `use(path, operation, params?)` | `Promise<UseResult>` | **Use a credential without retrieving it** |
| `retrieve(path, ttl?)` | `Promise<string>` | Retrieve credential value (lease auto-tracked) |
| `list()` | `Promise<CredentialEntry[]>` | List accessible credentials |
| `releaseLease(leaseId)` | `Promise<void>` | Release a credential lease early |
| `close()` | `Promise<void>` | Release all leases and disconnect |

### `use()` Operations

| Operation | Description | Key Params |
|---|---|---|
| `http_request` | Proxy an HTTP request through the vault | `method`, `url`, `headers`, `body`, `header_type` |
| `http_header` | Get an auth header without making the request | `header_type` |
| `sign` | Sign data with HMAC | `algorithm`, `data` |
| `encrypt` | Encrypt data | `plaintext` |
| `decrypt` | Decrypt data | `ciphertext` |

## Error Handling

All errors extend `VaultError` with structured context:

```typescript
import { SanctumClient } from "sanctum-ai";
import {
  VaultError,
  AuthError,
  AccessDenied,
  CredentialNotFound,
  VaultLocked,
  LeaseExpired,
} from "sanctum-ai/errors";

try {
  const result = await client.use("openai/api-key", "http_request", {
    method: "GET",
    url: "https://api.openai.com/v1/models",
    header_type: "bearer",
  });
} catch (e) {
  if (e instanceof AccessDenied) {
    console.error(`No access: ${e.detail}`);
    console.error(`Suggestion: ${e.suggestion}`);
  } else if (e instanceof CredentialNotFound) {
    console.error(`Path not found: ${e.detail}`);
  } else if (e instanceof AuthError) {
    console.error("Authentication failed — check your Ed25519 key");
  } else if (e instanceof VaultLocked) {
    console.error("Vault is locked — unlock it first");
  } else if (e instanceof VaultError) {
    console.error(`[${e.code}] ${e.detail}`);
  }
}
```

### Error Types

| Class | Code | When |
|---|---|---|
| `VaultError` | — | Base error for all vault errors |
| `AuthError` | `AUTH_FAILED` | Ed25519 authentication failed |
| `AccessDenied` | `ACCESS_DENIED` | Agent lacks permission for this credential |
| `CredentialNotFound` | `CREDENTIAL_NOT_FOUND` | Credential path doesn't exist |
| `VaultLocked` | `VAULT_LOCKED` | Vault is sealed/locked |
| `LeaseExpired` | `LEASE_EXPIRED` | Credential lease timed out |
| `RateLimited` | `RATE_LIMITED` | Too many requests |
| `SessionExpired` | `SESSION_EXPIRED` | Session expired, re-authenticate |

All errors carry `.code`, `.detail`, `.suggestion`, `.docsUrl`, and `.context`.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Write tests for new functionality
4. Ensure all tests pass (`npm test`)
5. Submit a pull request

## License

MIT — see [LICENSE](LICENSE).
