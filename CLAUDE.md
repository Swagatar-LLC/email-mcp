# email-mcp

Email MCP server (IMAP + SMTP) with 47+ tools, IMAP IDLE push, multi-account, AI triage.
Forked from `codefuturist/email-mcp` into `Swagatar-LLC/email-mcp` for security hardening.

## Quick Reference

```bash
pnpm test          # Unit tests (vitest, 169 tests)
pnpm typecheck     # tsc --noEmit
pnpm check         # Biome + ESLint (must both pass)
pnpm format        # Biome auto-format
pnpm lint:fix      # ESLint auto-fix
pnpm build         # tsc → dist/
```

All three gates (`typecheck`, `check`, `test`) must pass before commit.
Lefthook enforces this via pre-commit, commit-msg, and pre-push hooks.

## Tech Stack

- **TypeScript** — strict mode, ES2022 target, ESM (`"type": "module"`)
- **Node.js 24+** (works on 22)
- **pnpm 9.15.0** — lockfile must stay frozen (`--frozen-lockfile`)
- **Vitest** — unit tests in `src/**/*.test.ts`, integration in `src/__integration__/`
- **Biome** — formatting (2-space indent, single quotes, trailing commas, semicolons, 100 char line width)
- **ESLint** — airbnb-extended + TypeScript strict. Biome handles formatting; ESLint handles logic rules
- **Lefthook** — git hooks (pre-commit: biome + eslint + typecheck + actionlint; commit-msg: cocogitto; pre-push: test + check)
- **Cocogitto (cog)** — conventional commit enforcement
- **Docker** — multi-stage build, non-root `node` user

## Architecture

```
src/
├── main.ts              # CLI entry point + MCP server bootstrap
├── server.ts            # McpServer factory (name, version, capabilities)
├── logging.ts           # MCP protocol logging bridge
├── cli/                 # CLI subcommands (account, setup, test, config, install, scheduler, notify)
├── config/              # TOML config loading, Zod schemas, XDG path resolution
├── connections/         # Lazy-persistent IMAP (ImapFlow) + SMTP (nodemailer) connection pool
├── services/            # Business logic — one service per domain:
│   ├── imap.service.ts          # All IMAP read operations
│   ├── smtp.service.ts          # Send, reply, forward (with rate limit + send policy)
│   ├── oauth.service.ts         # OAuth2 token management (auth code + device code flows)
│   ├── credential.service.ts    # Hybrid credential resolution (keychain / env / plaintext)
│   ├── hooks.service.ts         # AI triage via MCP sampling + static rule matching
│   ├── watcher.service.ts       # IMAP IDLE real-time monitoring
│   ├── notifier.service.ts      # Desktop notifications + webhook dispatch
│   ├── scheduler.service.ts     # Scheduled email queue (HMAC-signed JSON files)
│   ├── calendar.service.ts      # ICS parsing from email attachments
│   ├── local-calendar.service.ts # macOS Calendar.app / Linux xdg-open integration
│   ├── template.service.ts      # User-defined TOML email templates
│   └── reminders.service.ts     # macOS Reminders integration
├── tools/               # MCP tool definitions (one file per tool group)
│   └── register.ts      # Central registration — read-only vs write gating
├── resources/           # MCP resource definitions (accounts, mailboxes, unread, stats, templates, scheduled)
├── prompts/             # MCP prompt definitions (compose, triage, cleanup, etc.)
├── safety/              # Security layer:
│   ├── audit.ts         # Append-only JSON Lines audit log (sensitive fields redacted)
│   ├── rate-limiter.ts  # Token-bucket rate limiter (per account)
│   └── validation.ts    # Input sanitization (mailbox names, search queries, webhook URLs, recipient domains, template vars)
├── types/               # Shared TypeScript interfaces
└── utils/               # Helpers (calendar notes, conference details, meeting URLs)
```

## Code Conventions

### Formatting (Biome)
- 2-space indentation
- Single quotes
- Trailing commas on all multi-line constructs
- Semicolons always
- 100-character line width
- Biome organizes imports — do not manually sort

### Linting (ESLint airbnb-extended + TypeScript strict)
- `no-console` is OFF in `src/cli/` and `src/main.ts` only
- `no-await-in-loop` — disable with `/* eslint-disable no-await-in-loop */` block comment when sequential awaits are required (polling loops, file processing)
- `no-continue` — avoid; restructure with `if/else` instead
- `no-underscore-dangle` — avoid `_prefixed` properties; use descriptive names
- `@typescript-eslint/no-use-before-define` — define helper functions ABOVE their callers
- `@typescript-eslint/promise-function-async` — functions returning promises must be `async`
- Test files (`*.test.ts`) have relaxed rules — see `eslint.config.mjs`

### TypeScript
- Strict mode with all `noUnused*` and `noImplicit*` checks enabled
- All imports use `.js` extension (ESM resolution): `import foo from './foo.js'`
- Prefer `interface` over `type` for object shapes
- Use Zod schemas for all external input validation (config, tool params)
- Service classes are plain TypeScript — no MCP dependency (unit-testable)
- Tool files depend on McpServer and delegate to services

### Commits
- **Conventional Commits** enforced by cocogitto: `feat(scope): description`
- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`
- Scope is optional but encouraged: `feat(oauth): add device code flow`
- Co-author trailers:
  ```
  Co-Authored-By: Craft Agent <agents-noreply@craft.do>
  Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
  ```

### Testing
- Unit tests live next to source: `foo.service.ts` → `foo.service.test.ts`
- Use `vi.mock()` for external dependencies (imapflow, nodemailer, child_process)
- Integration tests in `src/__integration__/` use testcontainers (Docker-based mail server)
- Test config objects must include all required fields (especially `sendPolicy` in `AppConfig`)

### Adding a New MCP Tool
1. Create `src/tools/my-feature.tool.ts` — export default function taking `McpServer` + services
2. Use `server.tool()` with Zod schema for input validation
3. Set tool annotations: `readOnlyHint`, `destructiveHint`, `openWorldHint`
4. Audit log all write operations via `audit.log()`
5. Register in `src/tools/register.ts` — read tools always, write tools gated behind `readOnly`

### Adding a New Service
1. Create `src/services/my-feature.service.ts` — plain class, no MCP imports
2. Accept dependencies via constructor injection (connections, other services)
3. Instantiate in `src/main.ts` `runServer()` and pass to tool registration
4. Add corresponding test file

## Security Patterns (Swagatar-LLC hardening)

These patterns were established during our security audit and must be maintained:

- **Send policy** — all outbound email paths (send, reply, forward, draft) call `SmtpService.checkSendPolicy()` which validates recipients against `settings.send_policy.allowed_domains` / `blocked_domains`
- **Credential service** — `ConnectionManager` and `WatcherService` resolve passwords via `resolveCredential()` which supports `"keychain"`, `"env:VAR_NAME"`, or `"plaintext"` (with runtime warning)
- **Config file permissions** — `saveConfig()` writes with mode `0o600`, directory with `0o700`
- **HMAC scheduler integrity** — `SchedulerService.writeScheduledFile()` signs queue files; `checkAndSend()` verifies before sending
- **Webhook URL validation** — `validateWebhookUrl()` blocks loopback, RFC1918, cloud metadata (169.254.x.x), CGNAT, IPv6 ULA/link-local
- **AppleScript escaping** — `escapeAS()` strips control chars and truncates to 1000 chars
- **Audit logging** — all write tools call `audit.log()` with sensitive field redaction
- **Rate limiting** — token-bucket per account, default 10/min
- **Read-only mode** — write tools are not registered when `settings.read_only = true`

## CI/CD

- **`.github/workflows/test.yml`** — runs on all branches: typecheck → lint → test → sanity check (build, CLI smoke test, Docker build)
- **`.github/workflows/ci.yml`** — upstream shared workflow (pinned to commit SHA `b2d5cf9`)
- **`.github/workflows/release.yml`** — npm publish + MCP registry + Docker (SHA-pinned workflows, checksummed mcp-publisher)
- **Dependency audit** — `pnpm audit --audit-level=high` runs in CI

## Config

Config lives at `~/.config/email-mcp/config.toml` (XDG). Key sections:

- `[settings]` — `rate_limit`, `read_only`, `send_policy`, `watcher`, `hooks`
- `[[accounts]]` — `name`, `email`, `password`/`credential_source`/`oauth2`, `imap`, `smtp`
- OAuth2 supports `flow = "device_code"` for M365 corporate accounts (no admin consent needed)
- Environment variables override config file (prefix: `MCP_EMAIL_`)
