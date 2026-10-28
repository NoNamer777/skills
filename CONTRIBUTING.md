# Contributing

[AGENTS.md](AGENTS.md) holds the rules for adding and changing skills. This file covers the changelog and releases.

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

3. Publish a GitHub release for the tag on [the new release page](https://github.com/NoNamer777/skills/releases/new):
   1. Select the `v<version>` tag you pushed in step 2, so the release uses the signed tag.
   2. Set the title to `v<version>`.
   3. Paste the version's section from `CHANGELOG.md` into the description, without its `## [<version>]` heading.
   4. Click **Publish release**.
