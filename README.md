# AI Media Studio Agent Skill

Public Agent Skill for teaching external Agents when and how to use the Media Studio CLI (`msc`) for multimodal generation and editing.

The Skill is intentionally separate from Media Studio runtime Skills. Runtime Skills are discovered and activated through `msc skills`; this repository teaches an Agent how to call the CLI.

The public repository `302ai/msc-skill` distributes both channels: `main` is stable and `beta` is opt-in. Stable Skill `main` and both production websites are released together from the same application commit: Deploy synchronizes the public Skill before rollout and verifies both `/SKILL.md` responses afterward. Beta CLI Release continues to sync only the public `beta` branch; it never updates the stable website or public `main`. Pushing source `main` alone does not sync either distribution. A manual stable sync is a no-drift verification and repair operation, not a way to publish ahead of the websites.

Each published Skill declares its own `metadata.version`, independently of the CLI package version, and its distribution `metadata.channel`; the latter does not select the CLI's site. Changing stable Skill content requires a new Skill version and an application release so all stable surfaces advance together. The two CLI packages share the `msc` executable, and Skill variants share the `msc-skill` name. The test site has only the global `@beta` package; there is no CN beta. Installing or switching either requires explicit user permission; preserve user work, but sign in again after installing or upgrading because each package and channel keeps a separate config directory and legacy credentials are not read.

## Install the Agent Skill

Install the stable Skill globally for the Agents supported by the Skills ecosystem:

```bash
npx -y skills add 302ai/msc-skill --skill msc-skill -g -y
```

For testing on the staging site, use the beta Skill:

```bash
npx -y skills add https://github.com/302ai/msc-skill/tree/beta --skill msc-skill -g -y
```

Start a new Agent session if the current session has already loaded its Skill catalog.

## Install the CLI

Use the install command provided by the `/agent-cli` page on the Media Studio site you intend to use:

| Site | Command |
| --- | --- |
| `https://media.302.ai` | `npm install -g @302ai/media-studio-cli@latest` |
| `https://media.302ai.cn` | `npm install -g @302ai/media-studio-cli-cn@latest` |
| `https://media.tool302.com` (testing only) | `npm install -g @302ai/media-studio-cli@beta` |

Node.js 22 or newer is required. Published packages do not target localhost; local development requires a matching local build.

## Bootstrap from either side

### Skill-only

When the Skill is active but `msc` is missing, use the site's copied setup instruction and ask for permission before installing its CLI. Reinstalls and package or channel switches also require permission. Obtaining a login link does not authenticate or generate media; authentication requires the user's one-time code. Do not generate media without a specific request.

### CLI-only

After installing the CLI, read the Agent-facing manual:

```bash
msc guide
```

The guide identifies the installed package, channel and site and provides its recommended Skill command.

### Full setup

The `/agent-cli` page copies concise numbered setup steps: install or update the CLI, then `msc-skill`, verify with `msc guide`, authenticate, and verify with `msc whoami`. Its CN text is Chinese; other sites use English. Both production origins serve the same canonical `/SKILL.md`, unchanged. It lists both production install commands; the Agent asks the user which site to use before running either.

For authentication, `msc login --json` exits 2 with `status: "authorization_required"`, `nextStep: "open_authorization_url"` and one `sites[0].authorizationUrl`. An Agent with an interactive browser tool opens that link in a new tab; otherwise the user opens it. The user signs in if needed and shares the displayed one-time code. Pass it to `msc login --with-code <code> --json` and check `msc whoami` (which has no `--json` flag). Do not use the host clipboard: sandboxed Agents cannot read it.

Codes expire in five minutes and work once. Generating another code does not invalidate earlier unused codes. Chat or Agent tools may retain a pasted code. Never copy `auth.json`.

## Required and optional updates

Agents finish the current task before reporting an optional update and never silently install it. Use the site-provided command and `msc guide` for any required or optional upgrade, requesting permission before installing or switching the CLI or Skill. Never follow a generic server message to a different site.

Online requests carry `X-Agent-Cli-Version` and `X-Agent-Cli-Package`. A `426` / `10010` upgrade refusal rejects that request before upstream work or streaming, but earlier stages may have side effects. An old headerless CLI may receive a prewarm SSE error with `errCode: 10010` and no HTTP status. A `503` is a service failure, not an upgrade refusal. Retry generation only when definitely unaccepted. Reuse any error `sessionId` with `--session`, never replay `--new`; inspect the existing session and artifacts if acceptance is uncertain.

## Safety

Installation and authentication are deliberate user actions. With explicit sign-in permission the Agent may execute one-time-code login with `--with-code <code>`. It must not read or copy `auth.json`, expose the code, run `msc chat` without a media request, or silently choose another paid provider.
