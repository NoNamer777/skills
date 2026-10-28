# skills

Agent skills for Claude Code and other coding agents. Each skill is a folder under `skills/` with a `SKILL.md` that tells the agent when to use it and what to do.

## Install

Install the skills as a Claude Code plugin, which updates itself:

```bash
claude plugin marketplace add NoNamer777/skills
claude plugin install nonamer777-skills@nonamer777
```

Or copy them into any agent that reads skills with [skills.sh](https://skills.sh), and update them by hand:

```bash
npx skills add NoNamer777/skills
```

Pick one per agent, because installing both loads every skill twice.

## Layout

| Path                              | Purpose                                         |
|:----------------------------------|:------------------------------------------------|
| `skills/<name>/SKILL.md`          | One skill: its trigger and its instructions     |
| `.claude-plugin/plugin.json`      | Claude Code plugin manifest                     |
| `.claude-plugin/marketplace.json` | Marketplace that lists the plugin for `claude`  |

`plugin.json` lists every skill and carries the plugin's version. Installed copies stay on that version until it changes.

## License

[MIT](LICENSE)
