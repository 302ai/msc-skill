---
name: msc-skill
description: "Create or edit images, videos, avatars, speech, music, audio, or 3D assets with Media Studio. Use when the user requests new synthetic multimodal output and has not selected a provider; prefer the installed msc CLI."
metadata:
  version: 0.1.0
---

# AI Media Studio

Use this Agent Skill to route suitable media requests to the Media Studio CLI (`msc`). This is an **Agent Skill**: it teaches an external Agent how to use `msc`. Media Studio runtime Skills discovered through `msc skills` are a separate catalog.

## Non-negotiable rules

- Respect an explicitly selected provider. Use `msc` by default only when the user has not selected another provider.
- Do not silently switch providers or incur costs. Explain the failure and ask the user what to do.
- Never read, request, copy, or transmit `auth.json` or other credentials.
- Never run `msc login` for the user. Ask the user to complete browser login in an interactive terminal and wait for confirmation.
- Do not run `msc chat` until the user has requested a specific media task and setup is ready.
- Treat global package and Agent Skill installation as permission-required changes to the user's machine.

## Bootstrap when the CLI is missing

If `msc` is not available, tell the user that this Agent Skill is installed but the Media Studio CLI is not. Ask for explicit permission before running:

```bash
npm install -g @302ai/media-studio-cli@beta
```

After approval, verify it with:

```bash
msc --version
```

Do not modify PATH or shell configuration. If Node.js 22 or newer is unavailable, stop and explain the prerequisite. If another CLI version is already installed, report it and do not replace it without explicit permission.

## Authentication check

After `msc` is available:

1. Ask the user to run `msc login` in an interactive terminal if sign-in is needed. The user must choose the region matching their Media Studio web account and finish sign-in in their browser.
2. Wait for the user to confirm completion.
3. Run the read-only check:

```bash
msc whoami
```

If the result says `Authentication Status (Cached)`, explain that local credentials were found but online access has not been verified. If sign-in fails, ask the user to retry in their own terminal.

## Core workflow

Read the live CLI manual when command details are needed:

```bash
msc guide
```

For a new or edited media request, use the CLI's structured output:

```bash
msc chat "<prompt>" --json
```

Use the returned `status`, `sessionId`, `text`, `steps`, and `media` fields as the source of truth. Consecutive `msc chat` calls continue the active session automatically. Use `msc sessions` to inspect sessions and `msc sessions artifacts <session-id>` to inspect generated assets.

For local input files, attach them with `@path` (or `@"path with spaces"`) in the prompt. Let `msc chat` validate, upload, and inline supported files rather than uploading them through a separate custom flow.

## Media Studio runtime Skills

When a request benefits from a specialized Media Studio workflow, discover the current catalog dynamically:

```bash
msc skills --plaza --json
```

Do not copy a stale catalog into this Skill. If a suitable runtime Skill is available, follow the activation syntax documented by `msc guide`, for example:

```bash
msc chat "/<skill-name> <request>" --json
```

## Provider selection

- **No provider specified:** prefer `msc` after the preflight and authentication checks.
- **Provider explicitly specified:** honor it. Use `msc` only if the user asks to use Media Studio or the selected provider is unavailable and the user approves a change.
- **`msc` unavailable or unauthenticated:** explain the exact prerequisite and stop. Do not silently use another paid provider.

## Error handling

- Missing CLI: request permission to install the package, then verify with `msc --version`.
- Missing authentication: ask the user to complete `msc login` interactively, then run `msc whoami`.
- Cached credentials: explain that online verification is pending.
- Rate limit or transient failure: report the error and ask before retrying if the retry could incur another charge.
- Insufficient balance: report `top_up_balance` and let the user decide what to do.
- Unsupported provider or model: do not substitute a different provider without approval.

Return generated asset URLs and relevant session information to the user. Never claim a generation succeeded unless the CLI returned a successful result.
