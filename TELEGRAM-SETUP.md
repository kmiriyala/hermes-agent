# Hermes Agent + Telegram — Local Setup Guide

This guide matches the install performed on this machine from `/Users/aicomputer/Hermes`. Official reference: [website/docs/user-guide/messaging/telegram.md](website/docs/user-guide/messaging/telegram.md).
## Configuration status (this machine)

As of 2026-07-07, Telegram is **configured** on this host:

- `~/.hermes/.env`: `TELEGRAM_BOT_TOKEN` and `TELEGRAM_ALLOWED_USERS` set (mode `600`); `OPENAI_API_KEY` unchanged (FreeLLMAPI).
- `~/.hermes/config.yaml`: `model.provider: custom` with `base_url: https://freelllmapi-production.up.railway.app/v1`.
- Gateway smoke test: log shows `[Telegram] Connected to Telegram (polling mode)` and `✓ telegram connected` in `~/.hermes/logs/gateway.log`.

Start the gateway when you want the bot live: `hermes gateway` or `hermes gateway start`.



## Prerequisites

- macOS (Apple Silicon) with network access
- [Node.js](https://nodejs.org/) 18+ (optional; some tools lazy-install via npm)
- **FreeLLMAPI key** (OpenAI-compatible Bearer token) for the agent LLM
- **Telegram bot token** from [@BotFather](https://t.me/BotFather)
- **Your Telegram user ID** from [@userinfobot](https://t.me/userinfobot)

## Install (from repo clone)

```bash
cd /Users/aicomputer/Hermes
export PATH="$HOME/.local/bin:$PATH"

# Non-interactive: skip optional ripgrep prompt and setup wizard
printf 'n\nn\n' | ./setup-hermes.sh

# If `venv/bin/hermes` is missing, sync explicitly into venv/ (not .venv/):
UV_NO_CONFIG=1 UV_PROJECT_ENVIRONMENT="$PWD/venv" uv sync --extra all --extra messaging --locked

mkdir -p ~/.local/bin
ln -sf "$PWD/venv/bin/hermes" ~/.local/bin/hermes
export PATH="$HOME/.local/bin:$PATH"

hermes --version
hermes doctor
```

**Note:** The `[all]` extra does not include Telegram libraries; use `--extra messaging` (or let the gateway lazy-install on first start).

## LLM — FreeLLMAPI (OpenAI-compatible)

Hermes talks to your hosted router at `https://freelllmapi-production.up.railway.app/v1` using `provider: custom`, not OpenAI direct.

**Split configuration (required):**

| File | Purpose |
|------|---------|
| `~/.hermes/.env` | Secrets (`chmod 600`) |
| `~/.hermes/config.yaml` | Model routing (`provider`, `base_url`, `default`) |

**Why both?** Hermes stores secrets in `.env` but does **not** send `OPENAI_API_KEY` to arbitrary third-party `base_url` hosts (credential-leak protection). For FreeLLMAPI you must set `model.api_key` in `config.yaml` (or use a `custom_providers` entry with `key_env: OPENAI_API_KEY`).

### `~/.hermes/.env`

```bash
# FreeLLMAPI unified key (OpenAI-compatible Bearer token)
OPENAI_API_KEY=freellmapi-<your-key>

# Telegram (configured on this machine)
TELEGRAM_BOT_TOKEN=<from BotFather>
TELEGRAM_ALLOWED_USERS=<your numeric user id>
```

### `~/.hermes/config.yaml`

```yaml
model:
  default: auto
  provider: custom
  base_url: https://freelllmapi-production.up.railway.app/v1
  api_key: freellmapi-<your-key>   # same key as OPENAI_API_KEY in .env
```

`default: auto` lets the FreeLLMAPI router pick the best available model. List models:

```bash
curl -s https://freelllmapi-production.up.railway.app/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY" | head -c 500
```

**Alternative (interactive):** `hermes model` → Custom endpoint → paste base URL, key, and `auto`.

## BotFather — create the bot

1. Open Telegram → chat with [@BotFather](https://t.me/BotFather).
2. Send `/newbot`.
3. Set a **display name** (e.g. `Hermes Agent`).
4. Set a **username** ending in `bot` (e.g. `my_hermes_agent_bot`).
5. Copy the **HTTP API token** BotFather returns (format: `123456789:ABC...`).
6. If the token leaks, revoke with `/revoke` in BotFather and update `~/.hermes/.env`.

Optional BotFather polish: `/setdescription`, `/setabouttext`, `/setuserpic`, `/setcommands`.

## Get your Telegram user ID

1. Open [@userinfobot](https://t.me/userinfobot) in Telegram.
2. Send any message (e.g. `/start`).
3. Copy the numeric **Id** (not `@username`).
4. For multiple people, use comma-separated IDs in `TELEGRAM_ALLOWED_USERS`.

## Required environment variables (Telegram)

Edit `~/.hermes/.env` (permissions should be `600`):

| Variable | Required | Description |
|----------|----------|-------------|
| `OPENAI_API_KEY` | Yes | FreeLLMAPI Bearer token (see LLM section) |
| `TELEGRAM_BOT_TOKEN` | Yes (for gateway) | Token from BotFather |
| `TELEGRAM_ALLOWED_USERS` | Yes (for gateway) | Your numeric user ID(s), comma-separated |

Example when Telegram is configured:

```bash
OPENAI_API_KEY=freellmapi-...
TELEGRAM_BOT_TOKEN=123456789:ABCdef...
TELEGRAM_ALLOWED_USERS=123456789
```

## Start, stop, and logs

```bash
export PATH="$HOME/.local/bin:$PATH"

# Foreground (polling — no public URL needed)
hermes gateway

# Background via launchd (macOS)
hermes gateway install
hermes gateway start
hermes gateway stop
hermes gateway status

# Logs
hermes logs --follow
hermes logs --level INFO --follow gateway
# or: tail -f ~/.hermes/logs/gateway.log
```

**Success signal:** `~/.hermes/logs/gateway.log` contains `[Telegram] Connected to Telegram (polling mode)` and/or `✓ telegram connected`.

## Quick verification (CLI)

```bash
export PATH="$HOME/.local/bin:$PATH"
hermes doctor
hermes status
hermes -z "say hello in one sentence"   # needs FreeLLMAPI config above
```

In Telegram (after gateway is running): DM your bot, send `/help` or `/new`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|----------------|-----|
| `HTTP 401: Invalid API key` on CLI | `model.api_key` missing for custom host | Add `api_key` under `model:` in `config.yaml` (see LLM section) |
| Bot does not reply | Gateway not running | `hermes gateway` or `hermes gateway start` |
| `python-telegram-bot` missing | Messaging extra not installed | `UV_PROJECT_ENVIRONMENT=venv uv sync --extra messaging --locked` |
| `Unauthorized` / invalid token | Wrong or revoked token | Regenerate in BotFather, update `.env`, restart gateway |
| Bot ignores DMs | User ID not allowlisted | Set `TELEGRAM_ALLOWED_USERS` to your @userinfobot Id |
| Bot silent in groups | Privacy mode | In BotFather: `/setprivacy` → Disable; or mention the bot |
| No LLM responses | Missing key or wrong `base_url` | Check `.env` + `config.yaml`; run `hermes doctor` |
| `~/.hermes/.env` ignored | Wrong path or permissions | File must be `~/.hermes/.env`, mode `600` |
| `hermes` not found | PATH / symlink | `export PATH="$HOME/.local/bin:$PATH"` or use `venv/bin/hermes` |

## Security

- Treat `TELEGRAM_BOT_TOKEN` and API keys like passwords.
- Only list trusted user IDs in `TELEGRAM_ALLOWED_USERS`.
- Do not commit `~/.hermes/.env` or live keys in `config.yaml` to git.

## Related docs

- [Telegram user guide](website/docs/user-guide/messaging/telegram.md)
- [Team Telegram assistant](website/docs/guides/team-telegram-assistant.md)
- [Providers / custom endpoints](website/docs/integrations/providers.md)
