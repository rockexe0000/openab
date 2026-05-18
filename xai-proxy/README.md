# xai-proxy

Lightweight Rust sidecar that authenticates with xAI via OAuth PKCE (SuperGrok subscription) and proxies OpenAI-compatible requests to `api.x.ai/v1`.

## Why

Use your SuperGrok subscription quota (instead of API credits) with any OpenAI-compatible coding agent — Claude Code, OpenCode, Codex CLI, etc.

## How it works

```
Your coding agent (CC, OpenCode, Codex, etc.)
        ↓  OpenAI-compatible request
  http://127.0.0.1:9090/v1
        ↓
  [xai-proxy]
  - Injects Authorization: Bearer <oauth_token>
  - Auto-refreshes token before expiry
        ↓
  https://api.x.ai/v1
```

## Build

```bash
cargo build --release
```

## Usage

### 1. Login (one-time)

```bash
./target/release/xai-proxy login
```

Opens your browser to `accounts.x.ai`. Sign in and authorize. Token is saved to `~/.xai-proxy/tokens.json`.

### 2. Start proxy

```bash
./target/release/xai-proxy serve --port 9090
```

### 3. Point your client

```bash
# Claude Code
export OPENAI_BASE_URL=http://127.0.0.1:9090/v1
export OPENAI_API_KEY=dummy

# OpenCode
export OPENAI_BASE_URL=http://127.0.0.1:9090/v1

# Codex CLI
export OPENAI_BASE_URL=http://127.0.0.1:9090/v1
export OPENAI_API_KEY=dummy
```

## OAuth details

| Item | Value |
|------|-------|
| Auth server | `https://auth.x.ai` |
| Client ID | Grok CLI public client |
| Flow | OAuth 2.0 PKCE (loopback callback) |
| Scope | `openid profile email offline_access grok-cli:access api:access` |
| Token storage | `~/.xai-proxy/tokens.json` (chmod 600) |
| Auto-refresh | Yes, 120s before expiry |

## Requirements

- Active SuperGrok subscription (any tier)
- Browser access for initial login (or SSH port-forward for headless)

## Headless / SSH login

```bash
# On your local machine:
ssh -N -L 56121:127.0.0.1:56121 user@remote-host

# On the remote host:
xai-proxy login
# Copy the URL and open in your local browser
```

## License

MIT
