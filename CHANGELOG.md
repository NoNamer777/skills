# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.3.4] - 2026-11-27

### Added

- `code-review` re-reviews a pull request you reviewed before: it reviews only the new commits, maps each point raised to the commit that fixes it, and revisits what held the earlier verdict back from Approve.
- `code-review` can reply to a review thread or resolve it when you ask.

### Changed

- `code-review` tests a change on the pull request's preview deployment, in your own signed-in browser, when it has one, and runs it locally only when it has none.
- `code-review` lists the workflows that did not run on the pull request's head, such as one that waits for a label, and offers to start them.
- `code-review` keeps its worktree after a verdict of Request changes or Comment, for the re-review, and offers to remove it once the verdict is Approve or the pull request closes.

### Fixed

- `code-review` handles a push that lands mid-review: it settles its findings against the new commits and drafts for the new head, instead of stopping at the post.
- `code-review` reads which review threads are resolved, which GitHub's REST API does not return.

## [0.3.3] - 2026-11-26

### Security

- `updating-pr-branch` reads a pull request's agent instructions from its base branch, so a pull request from a fork cannot change the rules its branch is updated by.

## [0.3.2] - 2026-11-26

### Security

- `code-review` reads a pull request's agent instructions from its base branch, so the pull request cannot change the rules it is reviewed by. It switches to a review account the repository names only once you confirm it.
- `code-review`, `addressing-pr-comments`, and `updating-pr-branch` treat pull request content (the diff, commits, descriptions, comments, and linked issues) as data. When that content addresses an agent, the skill quotes it to you and carries out none of it.

## [0.3.1] - 2026-11-26

### Changed

- `code-review` starts two voters per finding and a third only when its vote can change the outcome. It keeps each batch within the sub agent limit, votes first on the findings that decide the verdict, and runs voters on a smaller model.
- `code-review` writes the context every sub agent shares to one brief file, keeps sub agent reports short, and hands each open question to one review area, so a review spends less context and repeats less work.
- `code-review` reads the design documents a pull request links itself, and reviews output a person reads, such as labels, bot comments, and CLI output.
- `code-review` skips a check that already passed in the pull request's CI on its head commit, and mentions such skips to you instead of in the posted review.
- `code-review` lets you add a comment of your own to the draft, and reports progress per batch instead of per sub agent.

### Fixed

- `code-review` resumes a sub agent that a usage limit cut off instead of rerunning it, and keeps settled votes in a file that survives a context compaction.
- `code-review` makes its scratch directories under `/tmp`, since tools such as `tsx --test` failed under macOS's long default temporary path.

## [0.3.0] - 2026-11-25

### Added

- `code-review` skill: reviews a GitHub pull request, or your local changes, in sub agents that run the checks CI would run and vote on each finding that could block it, then drafts a verdict with inline comments and posts the review once you approve.

## [0.2.0] - 2026-11-22

### Added

- `updating-pr-branch` skill: brings a GitHub pull request's head branch up to date by merging its freshly pulled base branch, resolving any conflicts, and pushing the merge.

## [0.1.0] - 2026-10-28

### Added

- `addressing-pr-comments` skill: triages the open review feedback on a GitHub pull request you authored, then fixes, commits, and replies to what you approve.

[unreleased]: https://github.com/NoNamer777/skills/compare/v0.3.4...HEAD
[0.3.4]: https://github.com/NoNamer777/skills/releases/tag/v0.3.4
[0.3.3]: https://github.com/NoNamer777/skills/releases/tag/v0.3.3
[0.3.2]: https://github.com/NoNamer777/skills/releases/tag/v0.3.2
[0.3.1]: https://github.com/NoNamer777/skills/releases/tag/v0.3.1
[0.3.0]: https://github.com/NoNamer777/skills/releases/tag/v0.3.0
[0.2.0]: https://github.com/NoNamer777/skills/releases/tag/v0.2.0
[0.1.0]: https://github.com/NoNamer777/skills/releases/tag/v0.1.0
