---
name: msc-skill
description: "Generate or edit AI images, videos, avatars, speech, music, audio, or 3D assets with Media Studio's msc CLI. Use for new media requests, or to set up, sign in to, or upgrade msc or this Skill. Honors any provider the user names."
metadata:
  version: 0.1.9
  channel: stable
---

# AI Media Studio

You drive Media Studio through the `msc` CLI. This Skill covers routing, setup decisions, and failure handling. Command details live in `msc guide`, which matches the installed CLI. Runtime Skills (`msc skills`) are a separate catalog.

## Non-negotiable rules

- Honor an explicitly named provider. Use `msc` by default only when none is named, after setup and authentication checks.
- Use `msc` for a named provider only if the user asks for Media Studio, or the named provider is unavailable and the user approves the switch.
- Never switch providers or incur cost silently. If `msc` is unavailable or unauthenticated, state the exact missing prerequisite and stop.
- Run `msc chat` only for a specific media request, after setup is ready.
- Installing or upgrading the global CLI or this Skill changes the user's machine. Ask first, every time.
- Use a login code only for the user's authorized sign-in. Pass it via `--with-code`, never repeat it, never read or copy `auth.json`.

## Setup & Maintenance

Check your installation at any time with `msc guide`.

- **Prerequisites:** Node.js 22+ and `npm` are required.
  - **Diagnosis:** If `node` or `npm` is missing or outdated, actively diagnose the environment. Provide the user with the exact command to install Node.js 22+ for their OS (e.g., `nvm install 22` or a system package manager command).
  - **Connectivity:** If npm registry access is blocked, check for proxy configurations. If unreachable, offer to guide the user through downloading the CLI via alternate registries or direct sources.
- **Installation:** If `msc` is not installed, ask the user which production site they use, then after permission run only that command:
  - Global — https://media.302.ai: `npm install -g @302ai/media-studio-cli@latest`
  - Mainland China — https://media.302ai.cn: `npm install -g @302ai/media-studio-cli-cn@latest`
  Once installed, `msc guide` prints the exact install or update commands for the detected site.
- **Permissions:** You MUST ask for explicit permission before running any install or update command. If installation fails, diagnose the error (e.g., permission, registry) and propose a specific fix to the user.
- **Verification:** After any change, run `msc guide` again to confirm `Package:`, `Channel:` and `Site:` match the intended site. Start a new session if the updated Skill is not loaded.

## Authentication

Skip if `msc whoami` already succeeds.

1. Run `msc login --json`. Exit 2 with `status: "authorization_required"` is normal: it means waiting for the user. Open the single `sites[0].authorizationUrl` in a browser tool, or give it to the user.
2. Ask the user for the displayed one-time code. It expires in five minutes and works once. Run `msc login --with-code <code> --json`.
3. Run `msc whoami` to verify. It has no `--json` flag.

If the exchange fails or may have consumed the code, ask for a new code. Do not retry the old one.

## Creating media

- New request: `msc chat "<prompt>" --new --json`.
- Refinement of the same conversation: `msc chat "<refinement>" --json`. It continues the active session. Use `--session <id>` to target another.
- Always pass `--json`. stdout is data; stderr is diagnostics and progress.
- Attach local files inline with `@path` or `@"path with spaces"`. Remote URLs are not attachments.
- The CLI skips the website's per-turn price-confirmation setting. Clarification questions and insufficient balance still apply.

## Reading results

Exit 0 means `status` is `success` or `incomplete`. Exit 1 means `status` is `error`.

- **`incomplete`**: the stream ended early. `text` and `media` are partial, and media already received is downloaded. Inspect the session with `msc sessions artifacts <id> --json` before deciding anything else. Do not resubmit.
- **`AskUserQuestion` in `text`**: show the numbered options and wait. Never pick the recommended option for the user, and never treat stream completion as approval.
- **`steps`**: match `[step]` and `[result]` by full `toolCallId`, not by array position. Milestone wording follows the site's language; rely only on the prefix and the id.
- **`media` items** (`status: "completed"`):
  - `localPath`: the file is already downloaded. Show it through the host's attachment or local-file capability. Do not download it again. Use `fileUrl` only when the host cannot attach local files.
  - `resultUrl`: the permanent asset URL. Pass it on unchanged.
  - `downloadError`: explain it and offer `openUrl` or `resultUrl`. The asset still exists remotely.
  - Assets on other hosts have neither `localPath` nor `downloadError`. Use their remote URLs.
  - A dry-run price quote is not an asset and never appears in `media`.
- **`updateAvailable`**: finish the task, then tell the user. Never install it silently.
- Downloads go to `~/Downloads/media-studio-cli/` by default. Change it with `msc config set output-dir <path> --json`.

## Errors

On exit 1, parse the JSON on stdout and branch on `nextStep`. Do not parse `message` prose.

| `nextStep` | Action |
| --- | --- |
| `relogin` | Run the login flow again. Do not copy credentials from an earlier version. |
| `retry_later` | Wait, then retry. |
| `top_up_balance` | Report the balance shortfall and stop. |
| `fix_input` | Correct the prompt or parameters, then retry. |
| `retry` | Retry only if the request was definitely not accepted. Otherwise inspect the session first. |
| `upgrade_cli` | Follow "Required upgrade" below. |
| `none` | Report `message` to the user. |

If stdout is empty and exit is 1, read stderr. For `fetch failed`, check proxy, DNS and TLS diagnostics. Never disable TLS verification.

### Replay rules

- Retry generation only when it was known not to be accepted.
- If the error includes `sessionId`, reuse it with `--session <id>`. Never replay `--new`.
- If acceptance is uncertain, or a stream was incomplete, inspect the existing session and artifacts before submitting again. Never replay a prompt blindly; that duplicates charges.

## Required upgrade

`426`, `errCode: 10010` or `upgrade_cli` means the request was refused before any upstream work or stream began. Earlier stages of the same command may already have had effects. `503` is a service failure, not an upgrade refusal.

1. Ask permission. Name the site from the error or from `msc guide`. Never follow a generic server message that names a different site.
2. After approval, install per Setup. `msc guide` prints the Skill update command and the `Package:`, `Channel:` and `Site:` lines to verify.
3. If an install step fails, stop and report what completed. Do not claim success.

## Runtime Skills and sessions

- Discover runtime Skills with `msc skills --plaza --json`. Activate one by starting the prompt with `/<name>` or `%%/<name>%%`. The reference must be at the very start.
- Sessions: `msc sessions --json`, `msc sessions use <id> --json`, `msc sessions artifacts <id> --json`.
