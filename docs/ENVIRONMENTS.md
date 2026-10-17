# Environment preparation

Status: OWNER ACTION REQUIRED for three Telegram bots and credentials. No bots or webhooks were created.

| Environment | Bot | Secrets and webhook | MARKET_FORCE_OPEN |
|---|---|---|---|
| Development | Separate dev bot via BotFather | Private local values; dev HTTPS tunnel later | false by default; optional local true |
| Staging | Separate staging bot via BotFather | Separate secret store and staging HTTPS URL | Keep false |
| Production | Separate production bot via BotFather | Production secret store and production HTTPS URL | Must be absent |

1. In Telegram, open the official @BotFather, run `/newbot` three times and choose distinct owner-approved names/usernames. Record bot identity and environment without tokens in project notes.
2. Store each token privately as TELEGRAM_BOT_TOKEN; generate a separate TELEGRAM_WEBHOOK_SECRET per environment. Never paste tokens in issues, commits, logs or this documentation. Never reuse a bot across environments: setWebhook replaces the previous webhook.
3. Fill JWT_SECRET and environment-specific DB credentials privately. DATABASE_URL_TEST must use independent `monsaham_test_claude` and `monsaham_test_codex` databases for later worktrees; do not create databases or run migrations in this phase.
4. Populate MARKET_API_URL/MARKET_API_KEY only after provider approval. AI_PROVIDER_URL/AI_PROVIDER_API_KEY/AI_MODEL_CONTEXT require provider selection. Payment variable names await a gateway contract.
5. Set VITE_API_URL to the public API URL for that environment. It is the only client setting in the template; keep all secrets server-side. NODE_ENV must be production in production.
6. After the later server phase supplies a working endpoint, the owner registers that bot's webhook to its environment's HTTPS `/webhook/telegram`, supplies the matching secret token, checks getWebhookInfo and smoke-tests `/status`. Do not register against nonexistent endpoints now; avoid exposing token-bearing request URLs in shell history or screenshots.

The root `.env.example` inventories existing names from AGENTS §7, STYLE §6–7 and WORKFLOW Phases 4/7/8. DATABASE_URL is explicitly a proposed config name pending Phase 2 owner confirmation. No credentials are included. The existing empty `.env` and legacy `env-example` are preserved; use `.env.example` as the template. Environment loading, database setup, web server and cron do not exist yet.
