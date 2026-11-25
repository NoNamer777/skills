# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.3.0] - 2026-11-25

### Added

- `code-review` skill: reviews a GitHub pull request, or your local changes, in sub agents that run the checks CI would run and vote on each finding that could block it, then drafts a verdict with inline comments and posts the review once you approve.

## [0.2.0] - 2026-11-22

### Added

- `updating-pr-branch` skill: brings a GitHub pull request's head branch up to date by merging its freshly pulled base branch, resolving any conflicts, and pushing the merge.

## [0.1.0] - 2026-10-28

### Added

- `addressing-pr-comments` skill: triages the open review feedback on a GitHub pull request you authored, then fixes, commits, and replies to what you approve.

[unreleased]: https://github.com/NoNamer777/skills/compare/v0.3.0...HEAD
[0.3.0]: https://github.com/NoNamer777/skills/releases/tag/v0.3.0
[0.2.0]: https://github.com/NoNamer777/skills/releases/tag/v0.2.0
[0.1.0]: https://github.com/NoNamer777/skills/releases/tag/v0.1.0
