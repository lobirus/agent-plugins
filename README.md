# lobirus agent plugins

Plugins for AI coding agents, packaged once for every agent. Each plugin here is both an
[Agent Plugins](https://agent-plugins.org) package, the open format that Codex, GitHub Copilot,
VS Code, Cursor, Kiro and others read, and a [Claude Code](https://code.claude.com/docs/en/plugins)
plugin. The repository is a marketplace for all of them: [malac](https://github.com/lobirus/malac),
Claude Code, Codex, GitHub Copilot, VS Code and Cursor.

## Plugins

| Plugin | What it does | Parts |
|---|---|---|
| [server-setup](plugins/server-setup) | `/server-setup:new-server <host>` turns a fresh Ubuntu 22.04+ or Debian 12+ server into a hardened Docker host, without locking you out | skill |
| [release-tools](plugins/release-tools) | Writes the changelog entry for the commits since the last tag, drafts release notes, checks a release before tagging, and blocks `git push --force` | skill with a script, command, agent, hook |
| [ponytail](https://github.com/DietrichGebert/ponytail) | Lazy senior dev mode by Dietrich Gebert: the smallest change that fully solves the task, plus reviews and audits for bugs, risk and bloat | listed from its own repository, so you get its updates |

### server-setup

`/server-setup:new-server [root@]<host-or-ip>` bootstraps a fresh server over SSH from your machine:

1. Installs byobu, ufw, chrony and sudo.
2. Creates an admin user with passwordless sudo, copies your local SSH public key to it, and enables byobu for it.
3. Sets the time zone to UTC and runs chrony for NTP.
4. Sets up ufw to allow only the SSH port. NTP stays closed: time sync works over outgoing traffic.
5. Moves SSH to a random high port, turns off root and password login, and locks root's password.
6. Offers to add a friendly host alias to `~/.ssh/config`.
7. Runs `apt full-upgrade`, reboots, and checks that everything came back up.
8. Formats an unused disk as ext4 and mounts it at `/data`, after asking. Apps go in `/data/apps`.
9. Installs Docker and Compose from Docker's own repository, with Docker's data on `/data` and log rotation on.
10. Generates an ed25519 key on the server for pulling Git repositories, and checks it once you have added it to GitHub or GitLab.

Port 22 stays open until the new port, the new user and sudo all pass a test over a fresh connection, so a failed step does not lock you out. It asks before changing `~/.ssh/config` and before formatting a disk, and every step checks whether its work is already done, so you can run it again after a failure. The agent never starts it on its own (`disable-model-invocation`); you run the command.

### release-tools

- **changelog** (skill): lists the commits since the last tag with its script, groups them the [Keep a Changelog](https://keepachangelog.com) way, and adds the entry to `CHANGELOG.md`.
- **release-notes** (command): `/release-tools:release-notes 2.0.0` drafts release notes from that version's changelog entry.
- **release-reviewer** (agent): checks that the version is bumped everywhere, the changelog has the entry and the tests pass, and changes nothing.
- **no-force-push** (hook): refuses `git push --force` and `-f` before the shell runs it; `--force-with-lease` passes.

## Add the marketplace

**malac**

```sh
malac plugin marketplace add lobirus/agent-plugins
malac plugin install server-setup@lobirus
```

malac asks before enabling a plugin that runs code (release-tools' hook and script, ponytail's hooks). `--project` records the marketplace and plugin in `.malac/plugins.toml` for your team.

**Claude Code**

```text
/plugin marketplace add lobirus/agent-plugins
/plugin install server-setup@lobirus
```

From a shell: `claude plugin marketplace add lobirus/agent-plugins`, then `claude plugin install server-setup@lobirus`.

**GitHub Copilot CLI**

```sh
copilot plugin marketplace add lobirus/agent-plugins
copilot plugin install server-setup@lobirus
```

**VS Code** (agent plugins, experimental): add the repository to the `chat.plugins.marketplaces` setting, or run `/plugin marketplace add lobirus/agent-plugins` in a chat session.

```json
"chat.plugins.marketplaces": ["lobirus/agent-plugins"]
```

**Codex**

```sh
codex plugin marketplace add lobirus/agent-plugins
codex plugin add server-setup@lobirus
```

Codex reads `.agents/plugins/marketplace.json`. In the ChatGPT desktop app the marketplace then shows up in the Plugins Directory.

**Cursor**: in the Cursor dashboard, **Plugins & MCPs** → **Team Marketplaces** → **Add Marketplace** → **Import from Repo**, and paste `https://github.com/lobirus/agent-plugins`. Cursor reads `.cursor-plugin/marketplace.json`, which lists the plugins in this repository; ponytail ships its own Cursor setup, described in [its install guide](https://github.com/DietrichGebert/ponytail/blob/main/INSTALL.md).

**Other Agent Plugins clients** load a plugin from its folder: clone this repository and point the client at `plugins/<name>`.

## What each agent gets

| Part | Where it lives | Read by |
|---|---|---|
| Skills | `skills/<name>/SKILL.md` with `scripts/` and `references/` | every client: Agent Plugins and Claude Code |
| MCP servers | `mcp.json` (Agent Plugins) and `.mcp.json` (Claude Code) | every client that supports MCP (no plugin here has one yet) |
| Commands | `commands/*.md` | Claude Code, malac |
| Agents | `agents/*.md` | Claude Code, malac |
| Hooks | `hooks/hooks.json` | Claude Code, malac |

Skills and MCP servers are the portable parts of the Agent Plugins format. Commands, agents and hooks are not part of it yet, so they use Claude Code's layout, which malac also reads; other clients ignore them. A client-only part that no other agent understands goes in a top-level folder named after the client's reverse-domain namespace, such as `io.malac/` for malac, as the [specification](https://agent-plugins.org/specification) describes for client extensions. None of the plugins needs one today.

Two fields in server-setup's `SKILL.md`, `argument-hint` and `disable-model-invocation`, are Claude Code extensions to Agent Skills that malac and VS Code read too. They keep the agent from starting a server bootstrap on its own, so they stay; strict Agent Skills validators report them, and `scripts/validate` lists them as warnings.

## Layout

```text
.
├── .claude-plugin/marketplace.json   catalog for Claude Code, Copilot, VS Code and malac
├── .agents/plugins/marketplace.json  catalog for Codex
├── .cursor-plugin/marketplace.json   catalog for Cursor
├── plugins/<name>/
│   ├── plugin.json                   Agent Plugins manifest
│   ├── .claude-plugin/plugin.json    Claude Code manifest, same metadata
│   ├── skills/<skill>/SKILL.md       with scripts/ and references/
│   ├── mcp.json, .mcp.json           MCP servers, when the plugin has any
│   ├── commands/, agents/, hooks/    Claude Code layout
│   └── LICENSE                       when it differs from the repository's
├── schemas/agent-plugins/            the Agent Plugins JSON schemas (Apache-2.0)
└── scripts/validate
```

The Agent Plugins specification defines the package, not a catalog format or location, so this repository ships the catalog each client looks for. All three list the same plugins.

## Contribute a plugin

1. Create `plugins/<name>/` with a `name` of lowercase letters, digits, `-` and `.`.
2. Write `plugin.json` with `"$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json"`, `name`, `version`, `description`, `author` and `license`, and `.claude-plugin/plugin.json` with the same `name`, `version`, `description` and `license`.
3. Put each skill in `skills/<skill>/SKILL.md` with `name` (the folder's name) and `description` (what it does and when to use it) frontmatter, following [Agent Skills](https://agentskills.io/specification). Refer to its scripts and references by paths relative to the skill's folder, such as `scripts/check.sh`, so every client can find them; make scripts executable.
4. MCP servers go in `mcp.json` with the Agent Plugins MCP `$schema` (`./` commands, `${PLUGIN_ROOT}` in arguments) and, for Claude Code, in `.mcp.json` with `${CLAUDE_PLUGIN_ROOT}`. Do not put secrets in either.
5. Commands, agents and hooks use Claude Code's layout. Hooks run code on users' machines: keep them small and readable, and say in the description what they do.
6. Add the plugin to all three catalogs: `.claude-plugin/marketplace.json` and `.cursor-plugin/marketplace.json` with `"source": "./plugins/<name>"`, and `.agents/plugins/marketplace.json` with `"source": {"source": "local", "path": "./plugins/<name>"}` and a `policy`.
7. Run `scripts/validate`, and add the plugin to the table in this README.
8. A plugin that lives in its own repository is listed, not copied: add a `github` source (`{"source": "github", "repo": "owner/repo"}`) to the Claude Code catalog and a `url` source to the Codex one, as ponytail is.

Plugins are under the repository's MIT license unless their folder has a LICENSE of its own.

## Validation

```sh
scripts/validate            # warnings are reported, errors fail
scripts/validate --strict   # warnings fail too
```

It needs Node.js 20 or newer and nothing else, and checks:

- every `plugin.json` and `mcp.json` against the Agent Plugins schema its `$schema` names (vendored in `schemas/agent-plugins/`, so nothing is fetched), plus the rules the schema cannot express: `./` commands and working directories inside the plugin, HTTPS for remote servers, the MCP version matching the manifest's
- every `.claude-plugin/plugin.json`: kebab-case name, the same name, version, description and license as `plugin.json`, component paths that start with `./` and stay inside the plugin
- every skill against Agent Skills (name, folder, description, compatibility, metadata, allowed fields) and that its scripts are executable
- commands' and agents' frontmatter, and `hooks/hooks.json` events and the files its commands refer to
- that no symlink leaves its plugin
- the three catalogs: names, sources that exist, entry names matching the manifests, every plugin listed, and the same plugins in each

GitHub Actions runs it on every push and pull request ([workflow](.github/workflows/validate.yml)). `claude plugin validate .` and `claude plugin validate plugins/<name>` check the Claude Code side as Claude Code sees it; `malac plugin marketplace add ./` followed by `malac plugin show <name>` shows what malac makes of each plugin.

## License

[MIT](LICENSE), © Bálint Horváth. The Agent Plugins schemas in `schemas/agent-plugins/` are under the Apache License 2.0 ([details](schemas/agent-plugins/README.md)). ponytail is not part of this repository: it comes from [its own repository](https://github.com/DietrichGebert/ponytail) under its own MIT license.
