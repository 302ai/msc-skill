# AI Media Studio Agent Skill

Public Agent Skill for teaching external Agents when and how to use the Media Studio CLI (`msc`) for multimodal generation and editing.

The Skill is intentionally separate from Media Studio runtime Skills. Runtime Skills are discovered and activated through `msc skills`; this repository teaches an Agent how to call the CLI.

## Install the Agent Skill

Install it globally for the Agents supported by the Skills ecosystem:

```bash
npx -y skills add 302ai/msc-skill \
  --skill msc-skill -g -y
```

Start a new Agent session if the current session has already loaded its Skill catalog.

## Install the CLI

The Agent Skill can guide an Agent through this setup, but the CLI can also be installed directly:

```bash
npm install -g @302ai/media-studio-cli@beta
```

Node.js 22 or newer is required.

## Bootstrap from either side

### Skill-only

When the Skill is active but `msc` is missing, the Agent should ask for permission before running the CLI install command. It must not run `msc login` or generate media automatically.

### CLI-only

After installing the CLI, read the Agent-facing manual:

```bash
msc guide
```

The guide includes the command for installing this public Agent Skill.

### Full setup

The `/agent-cli` page provides one setup prompt that checks Node.js, `msc`, and this Skill, asks before global installation, checks authentication with `msc whoami`, and waits for a specific media request.

## Safety

Installation and authentication are deliberate user actions. The Agent must not read or copy `auth.json`, run `msc login` on the user's behalf, run `msc chat` without a media request, or silently choose another paid provider.
