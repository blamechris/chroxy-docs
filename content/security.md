---
title: Security & Encryption
---

Chroxy is **self-hosted**: the server daemon runs on the user's own development machine, and remote clients connect to it through a Cloudflare tunnel. This is fundamentally different from a multi-tenant relay. The encryption and auth layers exist to protect data in transit and to gate who can drive the server — not to defend the user from their own machine.

## Trust boundaries

```
                Trusted                 Untrusted              Trusted
            +--------------+      +------------------+     +-----------+
            |  User's Mac  |      | Cloudflare Tunnel|     | User's    |
            |  (Server)    |<---->|  (Transport)     |<--->| Phone     |
            |              |      |                  |     | (App)     |
            +--------------+      +------------------+     +-----------+
                Plaintext           Encrypted only           Plaintext
                (by design)                                  (by design)
```

The **server** is fully trusted — it must decrypt messages to route input to the AI CLI, parse output, and manage sessions. The **Cloudflare tunnel** is untrusted transport; encryption prevents the tunnel operator or any network intermediary from reading message content. The **client device** is a trusted endpoint that generates ephemeral keys and encrypts/decrypts locally.

## End-to-end encryption

Chroxy uses TweetNaCl on both the server and clients:

1. **Auth** — the client authenticates with a pre-shared API token. The server replies `auth_ok` with `encryption: 'required'` (or `'disabled'` if `--no-encrypt` was passed).
2. **Key exchange** — both sides generate ephemeral X25519 keypairs (`nacl.box.keyPair()`). The client sends its public key via `key_exchange`; the server replies with `key_exchange_ok` carrying its own.
3. **Shared key** — both call `nacl.box.before(...)`, performing Curve25519 Diffie-Hellman + HSalsa20 to derive a 32-byte symmetric key.
4. **Encryption** — subsequent messages are sealed with XSalsa20-Poly1305 (TweetNaCl secretbox).

Keys are **ephemeral** (new keypair per connection), provide **forward secrecy** (compromising one session reveals nothing about others), and are **never persisted** to disk. During the key-exchange window the server queues outbound messages and never downgrades to plaintext — if encryption is enabled, any non-`key_exchange` message during the pending phase disconnects the client.

## Token classes

The control surface accepts three distinct token classes, each with a different authority scope:

| Token | Issued by | Scope |
|-------|-----------|-------|
| **Primary API token** | `chroxy init` / `TokenManager` rotation | Full session authority — equivalent to local access on the host |
| **Pairing-bound session token** | `PairingManager` on consuming a one-shot pairing ID | Bound to exactly one session; rejected by session-scoped handlers if the target differs |
| **Per-session hook secret** | `CliSession` constructor (32 random bytes) | One session and the CLI permission-hook callback endpoint (`POST /permission`) only |

The **primary token represents the operator of the host machine** — any holder can read every session's history, send input, switch models, change permission modes, and modify settings. That breadth is by design: the token is deliberately equivalent to local access. Protect it, and rotate it (`npx chroxy init` regenerates) if it leaks. The token is stored in the OS keychain when available, falling back to `~/.chroxy/config.json`; the mobile app uses `expo-secure-store`.

## Pairing flow

Pairing turns a short-lived pairing ID (shown in a QR code, 60s single-use TTL) into a longer-lived session token without ever exposing the primary token on the wire. The client scans the QR, sends `{ type: 'pair', pairingId }`, and the server issues a fresh 32-byte session token (24h TTL, in-memory) that the client stores and uses on subsequent connections.

## Permission gating

For Claude providers, tool use is gated through a permission system. The SDK provider gates **in-process** via `canUseTool`; the CLI and TUI providers route permission requests back to the host server through an HTTP hook authenticated by the per-session hook secret. A per-session **permission rule engine** can auto-allow or auto-deny eligible read-style tools (`Read`, `Write`, `Edit`, `NotebookEdit`, `Glob`, `Grep`); never-auto-allow tools (`Bash`, `Task`, `WebFetch`, `WebSearch`) are always prompted. Permission resolution is broadcast (`permission_resolved`) so a prompt answered on one client dismisses on the others.

Gemini and Codex report `permissions: false` — they run tools under whatever policy each CLI itself enforces. See [[providers|Providers]] and [[container-isolation|Container Isolation]] for how to constrain non-Claude providers.

## See also

- [Bearer token authority threat model (main repo)](https://github.com/blamechris/chroxy/blob/main/docs/security/bearer-token-authority.md)
- [Encryption threat model (main repo)](https://github.com/blamechris/chroxy/blob/main/docs/security/encryption-threat-model.md)
- [[configuration|Configuration]] — the `--no-auth` dev-only trust model and skip-permissions flag
