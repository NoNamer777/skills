# Contributing

This file covers adding and changing skills, the changelog, and releases.

## Adding or changing a skill

- Put each skill in its own folder, `skills/<name>/`, with a `SKILL.md` at its root. Name the folder after the `name` in the frontmatter of `SKILL.md`, in kebab-case.
- List each skill folder in the `skills` array of `.claude-plugin/plugin.json`, as `./skills/<name>`, in the same change that adds, renames, or removes it, except for a skill you hold back as [unreleased](#unreleased-skills).
- Bump `version` in `.claude-plugin/plugin.json` in every change that ships to users, since installed copies update only when it changes. Follow semver: a major bump for a removed or renamed skill, a minor bump for a new skill, and a patch bump for a fix to an existing one. Add the version's section to `CHANGELOG.md` in the same change, as [Changelog](#changelog) describes.

## Unreleased skills

Claude Code detects a plugin update by its `version`. While `version` stays the same, `claude plugin update` and auto-update leave an installed copy alone, even when `plugin.json` or a skill changed on `main`. A fresh install copies the current files, so it picks up a listed skill without any bump.

A listing without a bump therefore reaches new users and never existing ones. To keep a skill unreleased, leave it out of `skills` in `plugin.json` and leave `version` and the changelog alone. Add the listing, the bump, and the changelog entry together in the change that releases it.

`npx skills add` reads the `skills/` folder and ignores `plugin.json`, so a skill merged to `main` is installable that way, listed or not. Keep a skill that must stay hidden on its branch until its release.

## Changelog

[CHANGELOG.md](CHANGELOG.md) follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Its first section is always an empty `## [Unreleased]`, which stays in place through every release. Every change that bumps `version` in `.claude-plugin/plugin.json` adds a section for that version right below it, in the same pull request:

- Head the section `## [<version>] - <date>`, with the date the pull request merges in `YYYY-MM-DD` form.
- Group the entries under `### Added`, `### Changed`, `### Deprecated`, `### Removed`, `### Fixed`, or `### Security`. Write each entry for someone who uses the skills.
- Add a link reference for the version at the bottom of the file, below `[unreleased]` and above the older ones: `[<version>]: https://github.com/NoNamer777/skills/releases/tag/v<version>`.
- Point the `[unreleased]` link reference at the new tag: `[unreleased]: https://github.com/NoNamer777/skills/compare/v<version>...HEAD`.

Changes that leave the version alone, such as edits to this file, get no entry.

## Releasing

Release each version once the pull request that bumps it merges. Installed plugins update as soon as `version` changes on `main`; the tag and the GitHub release mark that version for people who read the repository or install with skills.sh.

1. Switch to `main`, bring it up to date, and check that `plugin.json` holds the version you release:

   ```bash
   git switch main
   git pull --ff-only
   ```

2. Create a signed, annotated tag and push it:

   ```bash
   git tag -s v<version> -m "v<version>"
   git push origin v<version>
   ```

3. Publish a GitHub release for the tag, either on [the new release page](https://github.com/NoNamer777/skills/releases/new):
   1. Select the `v<version>` tag you pushed in step 2, so the release uses the signed tag.
   2. Set the title to `Release v<version>`.
   3. Paste the version's section from `CHANGELOG.md` into the description, without its `## [<version>]` heading.
   4. Click **Publish release**.

   Or with `gh`: save the version's section from `CHANGELOG.md`, without its heading, to a file outside the repository, and pass it as the notes. `--verify-tag` makes `gh` use the signed tag from step 2 and stop when it is missing, instead of creating an unsigned one:

   ```bash
   gh release create v<version> --verify-tag --title "Release v<version>" --notes-file <file>
   ```
