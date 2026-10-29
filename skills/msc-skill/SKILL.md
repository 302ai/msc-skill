---
name: msc-skill
description: "Create or edit images, videos, avatars, speech, music, audio, or 3D assets with Media Studio. Use when the user requests new synthetic multimodal output and has not selected a provider; prefer the installed msc CLI."
metadata:
  version: 0.1.3
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
npm install -g @302ai/media-studio-cli
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

`msc whoami` contacts the platform directly. It exits 0 and reports `Authenticated` only when the platform validates the session. If it exits 1, the session is expired, invalid, or the platform is unreachable; prompt the user to re-run `msc login` in their terminal.

## Core workflow

Read the live CLI manual when command details are needed:

```bash
msc guide
```

### Starting a chat turn

For a new media request, start a fresh session to isolate context:

```bash
msc chat "<prompt>" --new --json
```

For follow-up refinement in the same conversation, omit `--new` to continue the active session automatically (or pass `--session <id>` to target an existing session):

```bash
msc chat "<refinement prompt>" --json
```

Always pass `--json` to receive structured output on stdout (diagnostics, progress, and spinners go to stderr).

### Interpreting `--json` chat output

The output is a JSON object with one of three `status` values:
- `"status": "success"`: The turn completed.
- `"status": "incomplete"`: The connection dropped or the stream was truncated before finishing (exits 0 with partial `text` and `media`).
- `"status": "error"`: Command failed (exits 1).

Fields in success/incomplete turns:
- `sessionId`: The session identifier upstream.
- `text`: Model commentary, planning, or explanation.
- `steps`: Milestone progress strings.
- `media`: Array of generated assets. Each item has:
  - `status`: `"completed"`
  - `domain`: `"image"`, `"video"`, `"audio"`, etc.
  - `resultUrl`: The permanent stored asset URL. Persist, pass to downstream tools, or save this URL unchanged.
  - `openUrl`: Present when browser rendering/opening requires a different origin (e.g. regional CDN). If present, provide this to the user for previewing.
  - `modelUsed`, `taskId`: Generation metadata.

### Attaching local files

Attach local files by typing `@path` or `@"path with spaces"` anywhere in the prompt:

```bash
msc chat "analyze @cover.png and create a video from it" --new --json
```

- Supports images, videos, audio, and documents under 50 MB, up to 10 files per turn.
- The CLI validates files locally, uploads them, and inlines remote URLs automatically.
- Remote URLs (`@https://...`) are not supported as `@path` attachments.

### Structured error handling (Exit code 1)

When a command fails with exit code 1 and stdout contains a JSON payload, parse:

```json
{
  "status": "error",
  "message": "...",
  "errCode": 12345,
  "httpStatus": 401,
  "nextStep": "relogin",
  "guidance": "...",
  "transportCode": "ECONNREFUSED"
}
```

Branch on `nextStep` rather than parsing error message prose:
- `relogin`: Session expired or credentials invalid. Instruct the user to run `msc login` interactively.
- `retry_later`: Rate limit reached. Wait before retrying.
- `top_up_balance`: Insufficient balance. Report account balance shortage to the user and halt.
- `fix_input`: Invalid prompt or parameters. Adjust input and retry.
- `retry`: Transient gateway/network error. Safe to retry once.
- `none`: Unrecoverable error. Report `message` to the user.

If stdout is empty and exit code is 1, a transport failure occurred before reaching the platform (check network or proxy connectivity).

## Media Studio runtime Skills

Media Studio provides specialized workflows (e.g., face swap, style transfer, lip sync) executable through `msc`:

1. Discover available marketplace skills:
   ```bash
   msc skills --plaza --json
   ```
2. Activate a runtime skill by leading the prompt with `/<skill-name>` or `%%/<skill-name>%%`:
   ```bash
   msc chat "/face-swap swap face in @target.jpg using @source.jpg" --json
   ```
   The reference must be at the very start of the prompt to activate upstream.

## Sessions and asset inspection

- List recent sessions:
  ```bash
  msc sessions --json
  ```
- Switch active session pointer:
  ```bash
  msc sessions use <session-id> --json
  ```
- Inspect generated assets from any session:
  ```bash
  msc sessions artifacts <session-id> --json
  ```

## Provider selection

- **No provider specified:** Prefer `msc` after the preflight and authentication checks.
- **Provider explicitly specified:** Honor it. Use `msc` only if the user asks to use Media Studio or the selected provider is unavailable and the user approves a change.
- **`msc` unavailable or unauthenticated:** Explain the exact prerequisite and stop. Do not silently use another paid provider.
