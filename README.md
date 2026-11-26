# skills

Agent skills for Claude Code and other coding agents. Each skill is a folder under `skills/` with a `SKILL.md` that tells the agent when to use it and what to do.

## Skills

| Skill                                                              | What it does                                                                                                |
|:-------------------------------------------------------------------|:------------------------------------------------------------------------------------------------------------|
| [`addressing-pr-comments`](skills/addressing-pr-comments/SKILL.md) | Triages the open review feedback on your pull request, then fixes and replies to what you approve           |
| [`code-review`](skills/code-review/SKILL.md)                       | Reviews a pull request, or your local changes when you name none, and drafts a verdict with inline comments |
| [`updating-pr-branch`](skills/updating-pr-branch/SKILL.md)         | Merges a pull request's latest base branch into its head, resolves conflicts, and pushes                    |

Each skill runs only when you call it by name: `/<skill>` with skills.sh, or `/nonamer777-skills:<skill>` with the plugin. Each takes an optional pull request number, URL, or `<owner>/<repo>#<number>`. Without one, `code-review` reviews your local changes and the others take the current branch's pull request.

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

| Path                              | Purpose                                                  |
|:----------------------------------|:---------------------------------------------------------|
| `skills/<name>/SKILL.md`          | One skill: its trigger and its instructions              |
| `unreleased/<name>/SKILL.md`      | A skill held back from release, which no install reaches |
| `.claude-plugin/plugin.json`      | Claude Code plugin manifest                              |
| `.claude-plugin/marketplace.json` | Marketplace that lists the plugin for `claude`           |

`plugin.json` lists every skill and carries the plugin's version. Installed copies stay on that version until it changes.

## License

[MIT](LICENSE)
