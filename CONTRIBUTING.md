# Contributing

[AGENTS.md](AGENTS.md) holds the rules for adding and changing skills. This file covers the changelog and releases.

## Changelog

[CHANGELOG.md](CHANGELOG.md) follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Every change that bumps `version` in `.claude-plugin/plugin.json` adds a section for that version to the top of the changelog, in the same pull request:

- Head the section `## [<version>] - <date>`, with the date the pull request merges in `YYYY-MM-DD` form.
- Group the entries under `### Added`, `### Changed`, `### Deprecated`, `### Removed`, `### Fixed`, or `### Security`. Write each entry for someone who uses the skills.
- Add a link reference for the version at the bottom of the file, above the older ones: `[<version>]: https://github.com/NoNamer777/skills/releases/tag/v<version>`.

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

3. Publish a GitHub release for the tag, with the version's changelog section as its notes:

   ```bash
   awk -v v="<version>" '/^## \[/ { f = index($0, "## [" v "]") == 1; next } /^\[[^]]+\]: / { f = 0 } f' CHANGELOG.md | gh release create v<version> --verify-tag --title v<version> --notes-file -
   ```
