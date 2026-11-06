# AI Media Studio Agent Skill

Public Agent Skill for teaching external Agents when and how to use the Media Studio CLI (`msc`) for multimodal generation and editing.

The Skill is intentionally separate from Media Studio runtime Skills. Runtime Skills are discovered and activated through `msc skills`; this repository teaches an Agent how to call the CLI.

The public repository `302ai/msc-skill` distributes both channels: `main` is stable and `beta` is opt-in. A verified CLI release syncs the Skill from the release's source commit to the branch matching its npm dist-tag (`latest` to `main`, `beta` to `beta`); pushing source `main` alone does not sync it. A manual sync supports Skill-only updates, promotion, and recovery when the source CLI version matches the selected live npm dist-tag.

Each published Skill declares its channel in `metadata.channel`. Both variants use the same `msc-skill` name, and both CLI variants use the same global package, so installing another channel replaces the existing Skill or CLI respectively. Channel changes require explicit user permission; do not infer the channel from the CLI version, since npm `beta` may point to a stable version.

## Install the Agent Skill

Install the stable Skill globally for the Agents supported by the Skills ecosystem:

```bash
npx -y skills add 302ai/msc-skill --skill msc-skill -g -y
```

For the opt-in beta Skill, use:

```bash
npx -y skills add https://github.com/302ai/msc-skill/tree/beta --skill msc-skill -g -y
```

Start a new Agent session if the current session has already loaded its Skill catalog.

## Install the CLI

The Agent Skill can guide an Agent through setup. To install the stable CLI directly:

```bash
npm install -g @302ai/media-studio-cli@latest
```

For the opt-in beta CLI:

```bash
npm install -g @302ai/media-studio-cli@beta
```

Node.js 22 or newer is required.

## Bootstrap from either side

### Skill-only

When the Skill is active but `msc` is missing, the Agent should select the CLI command matching the Skill's `metadata.channel` and ask for permission before running it. Reinstalls and channel switches also require permission. It must not run `msc login` or generate media automatically.

### CLI-only

After installing the CLI, read the Agent-facing manual:

```bash
msc guide
```

The guide includes commands for both public Agent Skill channels.

### Full setup

The `/agent-cli` page displays `Set up <origin>/SKILL.md` but copies expanded setup, account, conditional trial-credit, and authentication guidance. If copying fails, a dialog provides the full text for manual selection. The app serves [the canonical Skill](./skills/msc-skill/SKILL.md) at that same-origin URL as part of its build, independently of public repository sync; this source defaults to stable. The Skill checks Node.js, `msc`, and Agent Skill availability, asks before global installation, checks authentication with `msc whoami`, and waits for a specific media request. The user completes any browser login in their own terminal. A newly installed Skill may require a new Agent session; the current session can follow the Skill and `msc guide`.

## Required and optional updates

Agents must finish the current task before reporting an optional update, and never silently install it. Chat success/incomplete JSON may include a stable `updateAvailable: { current, latest, upgradeCommand, skillUpgradeCommand }`; successful whoami may append a stable notice. Stable notices share per-target-version local emission deduplication across both commands, saved after output rather than acknowledgment. Interactive terminal commands may also show beta notices for prerelease builds; beta notices are not deduplicated. A stable version installed via `@beta` cannot be identified as beta and receives no beta notice. Skills/sessions JSON arrays remain unchanged. npm discovery uses a best-effort 24-hour cache and short timeout; failure leaves the task result and exit status intact.

Local notification deduplication is best-effort, not exactly-once: simultaneous processes or an unwritable cache can repeat notices.

Online requests carry `X-Agent-Cli-Version`. CMS `CliVersionPolicy` covers whoami/chat/sessions/skills/attachments: `enabled` defaults true and false is the emergency disable. Blank `minSupportedVersion` uses the CLI package version bundled in the BFF build (0.1.4 for this release); a custom stable SemVer can raise or lower it. Clearing the minimum restores the baseline, not disabled enforcement. Headerless Bearer requests are legacy CLI and normally receive `426` (prewarm exception below); headerless browser-cookie/anonymous requests are unaffected (this is not an authentication boundary). Cached policy reads use save-triggered invalidation and periodic revalidation, not simultaneous cross-instance activation. CMS failure uses a valid available cache or `503`, never an upgrade refusal. Local help/version/guide/logout and browser login remain available. A whoami exit 1 is not always a session failure: distinguish upgrade refusal, service/network failure and actual authentication failure before asking for re-login.

For `CLI_UPGRADE_REQUIRED` (`10010`, HTTP `426`, `upgrade_cli`), explain that recovery installs the stable CLI and Skill, including a switch from beta if applicable. Ask permission for **both** global installations, then run in order:

Old clients may not decode `upgrade_cli`; the server's actionable message still supplies both commands and requires permission first. CLI publication does not deploy the BFF: publish 0.1.4 and sync the matching public Skill before deploying/activating the BFF build containing the updated CLI manifest.

The sole legacy transport exception is an upgrade refusal for **headerless Bearer `POST /api/sessions/prewarm`**: HTTP `200` SSE `event:error` carries flat `{ err_code: 10010, message: <full permission-first commands> }`, with no `session_ready` or upstream work. The old CLI exits 1 and exposes `errCode: 10010` and the message, but no `httpStatus`. Stamped clients and all other interfaces retain `426`; CMS failure stays `503`, never this SSE conversion.

```bash
npm install -g @302ai/media-studio-cli@latest
npx -y skills add 302ai/msc-skill --skill msc-skill -g -y
msc --version
msc guide
```

If either install fails, stop automatic recovery and report which steps completed; do not claim success or automatically roll back the other install. Installing the Skill may not reload a running Agent's cached context; the guide refreshes immediate command knowledge.

A `426` rejects only that request before upstream work or streaming; earlier stages may have side effects. Retry generation only when definitely unaccepted. Reuse any error `sessionId` with `--session`, never replay `--new`; inspect the existing session/artifacts if acceptance is uncertain. Accepted work and established SSE streams must not be terminated by a policy change.

## Safety

Installation and authentication are deliberate user actions. The Agent must not read or copy `auth.json`, run `msc login` on the user's behalf, run `msc chat` without a media request, or silently choose another paid provider.
