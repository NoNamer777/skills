# Agent instructions

This repository publishes agent skills, both as a Claude Code plugin and through skills.sh. [README.md](README.md) describes the layout.

- Put each skill in its own folder, `skills/<name>/`, with a `SKILL.md` at its root. Name the folder after the `name` in the frontmatter of `SKILL.md`, in kebab-case.
- Follow the `writing-for-agents` skill when you create or edit a skill, or this file.
- List each skill folder in the `skills` array of `.claude-plugin/plugin.json`, as `./skills/<name>`, in the same change that adds, renames, or removes it.
- Bump `version` in `.claude-plugin/plugin.json` in every change that ships to users, since installed copies update only when it changes. Follow semver: a major bump for a removed or renamed skill, a minor bump for a new skill, and a patch bump for a fix to an existing one. Add the version's section to `CHANGELOG.md` in the same change, as [CONTRIBUTING.md](CONTRIBUTING.md) describes.
