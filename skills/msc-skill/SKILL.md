---
name: msc-skill
description: "Create or edit images, videos, avatars, speech, music, audio, or 3D assets with Media Studio. Use when the user requests new synthetic multimodal output and has not selected a provider; prefer the installed msc CLI."
metadata:
  version: 0.1.8
  channel: beta
---

# AI Media Studio

Use this Agent Skill to route suitable media requests to the Media Studio CLI (`msc`). This is an **Agent Skill**: it teaches an external Agent how to use `msc`. Media Studio runtime Skills discovered through `msc skills` are a separate catalog.

## Non-negotiable rules

- Respect an explicitly selected provider. Use `msc` by default only when the user has not selected another provider.
- Do not silently switch providers or incur costs. Explain the failure and ask the user what to do.
- Handle only the one-time website login code needed for the user's authorized sign-in. Pass it through `--with-code <code>` without repeating it.
- Do not run `msc chat` until the user has requested a specific media task and setup is ready.
- Treat global package and Agent Skill installation as permission-required changes to the user's machine.

## Setup

Use the site-matching commands below. Node.js 22+ is required. After installing or updating the CLI, install or update `msc-skill` as well; they are independent components.

<!-- CLI_INSTALLATION_START -->
### Install the CLI

Choose exactly one production site and use only its command:

**Global — https://media.302.ai**

```bash
npm install -g @302ai/media-studio-cli@latest
```

**Mainland China — https://media.302ai.cn**

```bash
npm install -g @302ai/media-studio-cli-cn@latest
```

<!-- CLI_INSTALLATION_END -->

1. Install or update the CLI, then the Skill. Run `msc guide` and confirm `Package:`, `Channel:` and `Site:` match the intended site. For a local site, use a matching local build.
2. Run `msc whoami`. If a fresh login is needed, follow the three steps below. Use `msc doctor` for installation/PATH problems.
3. Start a new Agent session if the updated Skill is not loaded.

## Authentication

1. Run `msc login --json` and open the single `sites[0].authorizationUrl` in a browser tool, or give it to the user.
2. Ask for the new one-time code and run `msc login --with-code <code> --json`. Codes expire in five minutes and work once; request a new one after a failed or uncertain exchange.
3. Run `msc whoami` to verify. `whoami` has no `--json` flag.

For `upgrade_cli` or `503`, follow the recovery guidance below instead of repeating login.

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

The CLI skips the website's price-confirmation setting for each chat turn, including when continuing a web-created session. Other clarification questions still apply, and insufficient balance still prevents generation.

### Interpreting `--json` chat output

The output is a JSON object with one of three `status` values:
- `"status": "success"`: The turn completed.
- `"status": "incomplete"`: The connection dropped or the stream was truncated before finishing (exits 0 with partial `text` and `media`).
- `"status": "error"`: Command failed (exits 1).

Fields in success/incomplete turns:
- `sessionId`: The session identifier upstream.
- `text`: Model commentary, planning, explanation, or a pending `AskUserQuestion` with numbered options. When questions are present, show them to the user and send the user's explicit answer in the next `msc chat` turn; never select the recommended option automatically or treat the completed stream as approval.
- `steps`: Ordered tool-call and result strings. `[step] [toolCallId] ...` identifies each call; `[result] [toolCallId] ...` carries its result when one arrives. Match calls and results by the full `toolCallId`, not array position, because parallel calls can finish out of order. `AskUserQuestion` has a step but no result; read its pending question from `text`. Milestone bodies follow the installed site's language; only the `[step]`/`[result]` prefix and toolCallId are stable.
- `media`: Array of generated assets; a dry-run price quote is not an asset and never appears here. Each item has:
  - `status`: `"completed"`
  - `domain`: `"image"`, `"video"`, `"audio"`, etc.
  - `resultUrl`: The permanent stored asset URL. Persist, pass to downstream tools, or save this URL unchanged.
  - `openUrl`: The remote preview URL whenever `resultUrl` exists; it may be identical. Use it only as a fallback when local download fails.
  - `localPath`: The absolute local path after the CLI downloads the artifact. Prefer displaying it through the host's local-file/media attachment capability; do not download the artifact again. Only results on Media Studio storage are downloaded; an item on any other host has neither `localPath` nor `downloadError`, so use its remote URLs.
  - `fileUrl`: The `file://` URL corresponding to `localPath`. Provide it when the host cannot attach local files; do not treat it as a general web-page URL.
  - `downloadError`: Present when local saving failed. Explain the failure and offer the remote `openUrl`/`resultUrl`; the generated asset is still available remotely.
  - `modelUsed`, `taskId`: Generation metadata.

The default output directory is `~/Downloads/media-studio-cli/` (the native equivalent on Windows). Use `msc config get output-dir --json` to read the effective directory, `msc config set output-dir <path> --json` to set it, and `msc config unset output-dir --json` to restore the default. All three print `{ "outputDir": "/absolute/path", "source": "default" | "configured" }`. `get` is read-only; `chat` creates the directory only when downloading. The CLI also downloads media received before an `incomplete` stream ends.
- Optional `updateAvailable: { current, latest, upgradeCommand, skillUpgradeCommand }`: Finish the current task, then tell the user about the update. Compare the commands with the current `msc guide` and site instruction; never silently install or switch package or channel. Obtain explicit permission before each global installation.

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
- `relogin`: Follow the website authorization flow for an expired, invalid or legacy session. This version does not read credentials from an earlier version; sign in again instead of migrating or copying them. Each package and channel keeps its own configuration directory.
- `retry_later`: Rate limit reached. Wait before retrying.
- `top_up_balance`: Insufficient balance. Report account balance shortage to the user and halt.
- `fix_input`: Invalid prompt or parameters. Adjust input and retry.
- `retry`: Transient gateway/network error. Retry generation only when definitely not accepted; otherwise inspect existing work first.
- `none`: Unrecoverable error. Report `message` to the user.
- `upgrade_cli`: `CLI_UPGRADE_REQUIRED` (`10010`, HTTP `426`); follow the permission-required recovery below.

If stdout is empty and exit code is 1, inspect stderr: parser errors and transport failures in other commands may have no JSON payload. `msc login --json` emits `{status:"authorization_required",nextStep:"open_authorization_url",guidance:"...",sites:[{authorizationUrl}]}` and exits 2 while waiting for user authorization. `msc login --with-code <code> --json` emits one success `{status:"success",apiUrl,uid,username?}` after save, or the shared structured error on failure. Login JSON never includes a code/token. `whoami` and `logout` do not support `--json`.

### Required CLI upgrade and safe recovery

An upgrade refusal (`426`, `10010`, `upgrade_cli`) rejects that request before its upstream work or stream begins, but earlier command stages may have effects. A policy failure (`503`) is not an upgrade refusal.

For an upgrade refusal, use the site-provided install command and the current `msc guide`, not a generic server message that might name a different site. Ask permission before upgrading or replacing the CLI and before installing or switching the Skill. Follow the setup procedure above, verify `Package:`, `Channel:` and `Site:` after installation, and preserve user work and credentials. If installation fails, stop and report which steps completed; do not claim success or automatically roll back. Skill installation may not reload the running Agent's cached context; read `msc guide` before resuming.

Retry only generation known not to have been accepted. If the error includes `sessionId`, reuse it with `--session <id>`, never replay `--new`. If acceptance is uncertain or work has begun (including an incomplete stream), inspect the existing session and artifacts before deciding on another submission; never blindly replay a prompt or incur duplicate charges.

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
