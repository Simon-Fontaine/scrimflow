# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Scrimflow is an Overwatch 2 team management platform. The repository is structured as a monorepo with the Next.js app under `next-app/` and Docker infrastructure configs at the root.

## Commands

All app commands run from `next-app/`. Root-level `package.json` scripts `cd next-app` automatically.

**Infrastructure (run first):**

```bash
docker compose -f docker-compose.dev.yml up db cache storage storage-init mail redis-commander -d
```

**App development:**

```bash
cd next-app
pnpm install
pnpm dev          # Next.js dev server (Turbopack)
pnpm build
```

**Linting/formatting (from repo root):**

```bash
pnpm check        # Biome check (lint + format)
pnpm check:fix    # Auto-fix
pnpm lint         # Lint only
pnpm format:fix   # Format only
```

**Database (from `next-app/`):**

```bash
pnpm db:generate  # Generate migration from schema changes
pnpm db:migrate   # Apply migrations
pnpm db:push      # Push schema directly (dev only)
pnpm db:studio    # Open Drizzle Studio
```

Biome runs on staged files as a pre-commit hook via Lefthook. No separate test framework is configured.

## Architecture

### Directory layout (`next-app/`)

- `app/` — Next.js App Router pages and Server Actions
  - `(auth)/auth/` — Single auth page with step-based UI; all auth Server Actions live here
  - `(home)/` — Marketing/landing pages
  - `dashboard/` — Authenticated user area (settings, etc.)
- `components/` — React components grouped by section (`auth/`, `home/`, `settings/`, `shared/`, `ui/`)
- `lib/` — Server-side utilities: `auth/` (session, 2FA, WebAuthn, password, device, email), `config/`, `rate-limit.ts`, `mailer.ts`, `logger.ts`, `crypto.ts`, `encryption.ts`
- `db/` — Drizzle ORM: `index.ts` (pg connection), `schema/` (split by domain: `auth.ts`, `core.ts`, `audit.ts`, `enums.ts`, `relations.ts`)
- `stores/` — Zustand client state (`auth-flow.ts`, `security-status.tsx`)
- `hooks/` — Custom React hooks (`use-auth-action.ts` is central to the auth flow)
- `emails/` — React Email templates (password reset, verification, security alerts)

Path alias: `@/` maps to the `next-app/` root.

### Auth flow pattern

The entire auth UI is a single page (`app/(auth)/auth/page.tsx`) rendered as a step-router. The current step is managed by Zustand (`stores/auth-flow.ts`). Steps: `login`, `register`, `forgot-password`, `forgot-password-sent`, `verify-email`, `new-device-verification`, `two-factor`, `recovery-code`, `reset-password`.

Server Actions in `app/(auth)/auth/actions/` return `ActionResult` (`types.ts`). When a result includes `nextStep`, the `useAuthAction` hook (wrapping `useActionState`) calls `transitionTo()` on the store to advance the UI. Errors are surfaced via Sonner toasts.

### Session & security

- Sessions: stored as an HttpOnly cookie `session_token` containing a random token; the SHA-256 hash is the DB key. 30-day rolling expiry, soft-revoked (never deleted).
- 2FA methods: TOTP (`@oslojs/otp`), Passkeys and Security Keys (WebAuthn via `@oslojs/webauthn`), Recovery codes.
- TOTP secrets and recovery codes are AES-encrypted at rest (`lib/encryption.ts`, key from `ENCRYPTION_KEY` env var).
- Rate limiting: Redis-backed (`ioredis`) with transparent in-memory fallback (`lib/rate-limit.ts`).
- Device fingerprinting: SHA-256 of UA triggers "new device" security email on first login from unknown device.

### Key conventions

- **Validation**: Valibot (`v` import from `valibot`) — not Zod. Schemas live in `lib/validations/auth.ts`.
- **Linting/formatting**: Biome — not ESLint or Prettier. Never configure ESLint.
- **Icons**: `@hugeicons/react` — not lucide-react or heroicons.
- **UI components**: shadcn/ui (`components/ui/`). Add new components with `pnpm dlx shadcn add <component>` from `next-app/`.
- **Package manager**: pnpm only. Node >=20 required.
- **Output**: `next.config.ts` sets `output: "standalone"` for Docker deployment.

### Environment variables

Copy `next-app/.env.example` to `next-app/.env`. Key vars:

- `DATABASE_URL` — PostgreSQL connection string
- `ENCRYPTION_KEY` — AES key for TOTP secrets/recovery codes (generate: `openssl rand -base64 16`)
- `REDIS_URL` — optional; falls back to in-memory rate limiting if unset
- `WEBAUTHN_RP_ID` / `WEBAUTHN_ORIGIN` — must match the app domain for passkeys to work
- `SMTP_*` — dev uses Mailpit on port 1025 (included in Docker Compose)
