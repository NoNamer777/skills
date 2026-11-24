---
name: code-review
description: Review a GitHub pull request, or the local changes when none is given, and draft a verdict with inline comments to post on approval.
argument-hint: "[<number> | <url> | <owner>/<repo>#<number>]"
disable-model-invocation: true
---

Review a pull request, or the local changes when no pull request is given, and advise on it: a verdict and a set of short review comments, anchored inline where possible. Advice changes nothing; the skill edits no code, and every review comment and submission waits for the user's approval.

The rules below are defaults. Where the repository's own agent instructions (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`) say otherwise about reviews, worktrees, or standards, follow the repository.

## 1. Resolve the target

With no argument, the target is the local changes: staged, unstaged, and untracked files (`git status --porcelain`). When there are none, say so and stop.

Otherwise read `<owner>`, `<repo>`, and `<number>` from the argument: a number belongs to the current repository (`gh repo view --json owner,name`), and a URL or `<owner>/<repo>#<number>` names its own. Fetch the pull request:

```bash
gh pr view <number> -R <owner>/<repo> --json author,state,title,body,url,baseRefName,headRefName,headRefOid,closingIssuesReferences
```

Stop and say why when `state` is not `OPEN`. Note whether `author.login` equals `gh api user --jq .login`: the user may review their own pull request, which changes only how step 6 posts.

## 2. Get the code

Local changes are reviewed in place; skip to step 3.

For a pull request, review its head branch in a local checkout, in this order:

1. **Worktree.** When `git worktree list` shows `headRefName` checked out, use that worktree.
2. **Branch.** When the current checkout is on `headRefName`, use it. When the branch exists locally and is checked out nowhere (`git branch --list <headRefName>`), add a worktree for it as in the next item, without `-b`, so the current checkout stays as it is.
3. **Neither.** Create a worktree at the location the repository's instructions name, else at `../.worktrees/<repo>/<slug>`, where `<slug>` is the branch name without its type prefix. Detach it at the base branch, then check the pull request out inside it, which also sets up a fork's remote:

   ```bash
   git worktree add --detach <path> <base-remote>/<baseRefName>
   cd <path> && gh pr checkout <number> -R <owner>/<repo>
   ```

Bring the checkout to the pull request's head: run `git fetch` and fast-forward a branch that is only behind with `git merge --ff-only`. Done when `git rev-parse HEAD` equals `headRefOid`. When the checkout has diverged, is ahead, or has uncommitted changes, stop and tell the user rather than review code that is not the pull request's.

Note what this step created, for step 7: the worktree and its path (items 2 and 3) and the branch (item 3 only, since item 2 reuses an existing one). Report the path.

## 3. Read the change

Read the full diff: `gh pr diff <number> -R <owner>/<repo>` for a pull request, and `git diff HEAD` plus the untracked files for local changes. Read the code around each hunk, not only the hunk.

For a pull request, also read its description, the issues in `closingIssuesReferences`, and the existing review threads and comments, so the review does not repeat a point that is already raised.

## 4. Review

Read every changed file. For each, check the change against:

- **Correctness:** logic errors, unhandled cases, broken callers.
- **Intent:** what the description and linked issues ask for.
- **Standards:** the repository's agent instructions, contributing docs, and lint configuration.
- **Tests:** coverage of the behavior the change adds or alters.

Sort each finding into one level:

| Level      | When                                        |
|:-----------|:--------------------------------------------|
| Blocking   | The change is wrong or breaks something     |
| Suggestion | The change works and could be better        |
| Question   | The intent or a choice is unclear           |

Done when every changed file is read and every finding carries a level and a `path:line`.

## 5. Present the review

Recommend one verdict for a pull request:

- **Request changes** when any finding is Blocking.
- **Approve** when there are no findings, or only Suggestions the author can take or leave.
- **Comment** otherwise, for example when only Questions remain.

Give the verdict with one line of reasoning, then the draft review: a summary body of a few sentences at most, and the inline comments, each with its `path:line`, its level, and its text. Anchor a comment inline whenever its line is part of the diff; a finding about the change as a whole, or about a line outside the diff, goes in the summary.

Write each comment so the author can act on it at a glance: name the problem and the change you ask for in a sentence or two, add the reason only when the author cannot infer it, and point at code by its line instead of quoting it. Put a small, concrete fix in a GitHub `suggestion` block.

For local changes, present the findings by `path:line` and level, with no verdict or draft; there is nothing to post.

Ask the user which comments to post and which verdict to use. They may approve any subset by number, reword a comment, or change the verdict. Stop here until the user answers.

## 6. Post

Skip this step for local changes.

Check that the pull request's head still equals `headRefOid`; when it moved, say so and stop, since the drafted lines may no longer match. Otherwise submit the approved comments as one review, built in a file outside the repository so the body survives any shell's quoting:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews --input <file>
```

The file holds `commit_id` (`headRefOid`), `event`, `body`, and `comments`, each comment with `path`, `line`, `side`, and `body`, plus `start_line` and `start_side` for a range. `event` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`.

On the user's own pull request, GitHub rejects `APPROVE` and `REQUEST_CHANGES`, so submit `COMMENT` and say that the verdict stays advice.

Done when the review is posted. Report its URL.

## 7. Offer to clean up

Skip this step when step 2 created nothing. Once the review is posted, or when the user decides to post nothing, offer to remove what step 2 created, naming the worktree path and the branch, and wait for the answer. The user may keep them to re-review after the author pushes.

On a yes, run from the original checkout, since a worktree cannot remove the folder you stand in:

```bash
git worktree remove <path>
git branch -d <headRefName>
```

Run the second command only for a branch step 2 created. Never pass `--force` or `-D`: when git refuses because of uncommitted or unpushed work, report that and leave it. Done when git has removed what the user approved, or has refused and the refusal is reported.
