---
name: code-review
description: Review a GitHub pull request, or the local changes when none is given, and draft a verdict with inline comments to post on approval.
argument-hint: "[<number> | <url> | <owner>/<repo>#<number>]"
disable-model-invocation: true
---

Review a pull request, or the local changes when no pull request is given, and advise on it: a verdict and a set of short review comments, anchored inline where possible. Advice changes nothing; the skill edits no code, and every review comment and submission waits for the user's approval.

Narrate the review as it runs. Before each step, say what you are about to do, including every worktree, branch, fetch, and post. Say what you run into as you meet it: a command git or `gh` refuses, a check that fails, a stop condition, an assumption you make, a step you skip and why.

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

Review top-down, and read every changed file.

1. **Scope and fit.** Decide whether the change should exist. Compare the description and each linked issue with what the diff does, and report the goals it misses and the changes it does not declare. Check size and focus: a large change (about 20 files, or several hundred hand-written changed lines) or one that mixes a refactor or reformatting with behavior gets a note first in the summary and a recommendation to split. Skim generated and lock files, and record every part you only skimmed.
2. **Design.** Read the files that hold the logic first, then the rest. Report a major design problem as soon as you find it, before reading on.
3. **Dimensions.** Check each changed file against these, applying a dimension marked "when" only where the diff touches it:
   - **Functionality:** edge cases, error handling, and concurrency when shared state or async code changes.
   - **Security,** when the diff touches input handling, authentication, authorization, cryptography, secrets, serialization, or file, process, or query construction: review the change for security. Trace data from its sources to its sinks across each changed trust boundary, and check the impact on existing controls, new attack vectors, new integrations, and regressions. When unsure about authentication, authorization, cryptography, or secrets, raise a blocking Question and recommend a human reviewer for that part.
   - **Tests:** new or changed logic has tests that would fail if the code broke. Read the tests as code: meaningful assertions, no false positives.
   - **Complexity:** nothing more generic or elaborate than the problem needs.
   - **Compatibility,** when the diff changes a public surface (exported API, CLI flags, config keys, file or wire formats, database schema, skill names and arguments): the break is declared and comes with a deprecation or migration path.
   - **Dependencies,** when a manifest or lockfile changes: necessity, maintenance state, license, known vulnerabilities, lockfile consistency.
   - **Performance:** only concrete risks visible in the diff, such as an unbounded query or repeated work in a loop.
   - **Documentation:** updated when user-facing behavior, the build, or the release changes.
   - **Naming and comments** (why, not what), and consistency with the surrounding code and the repository's rules.
4. **Automation.** For a pull request, read the CI status (`gh pr checks`) and the repository's lint configuration. Leave out what a formatter or linter enforces, and report a failing required check once, in the summary.

Sort each finding into one level:

| Level      | When                                                                                        |
|:-----------|:--------------------------------------------------------------------------------------------|
| Blocking   | Merging makes things worse: see below                                                       |
| Suggestion | The change works and could be better; the author may decline                                |
| Question   | The intent or a choice is unclear; blocking when its answer could reveal a Blocking finding |
| Nit        | Taste, or a point no rule covers; never gates a verdict                                     |

A finding is Blocking when merging leaves the codebase or its users worse off than not merging: incorrect or unhandled behavior, a regression, a vulnerability, an undeclared break of a public surface, new logic without meaningful tests, complexity or duplication the change adds, or a description or linked issue the change misses or contradicts. A problem that predates the change is a Suggestion with a follow-up issue.

Done when every changed file is read, every finding carries a level and a `path:line`, and you can list what you skimmed or did not run.

## 5. Present the review

Recommend one verdict for a pull request:

- **Request changes** when any finding is Blocking.
- **Approve** when nothing is Blocking, you reviewed all of the change, and it clearly improves the code even if imperfect. Suggestions and Nits ride along with an approval.
- **Comment** otherwise: a blocking Question is open, or you cannot vouch for the change because part of it is skimmed or unassessed, a required check is failing, or a security-sensitive part is beyond your confidence. Say what is missing.

Give the verdict with one line of reasoning, then the draft review. The summary body is a few sentences at most: the size note if there is one, what you skimmed, did not run, or did not assess, a failing check if there is one, and at most one sincere line of praise. Anchor each inline comment on a line that is part of the diff; a finding about the change as a whole, or about a line outside the diff, goes in the summary.

Write each comment so the author can act on it at a glance:

- Open with its label: `issue (blocking):`, `suggestion:`, `question (blocking):` or `question (non-blocking):`, or `nitpick:`.
- Cover one finding per comment. Name the problem and the change you ask for in a sentence or two, and pair an `issue` with a fix or direction when you know one. When you are unsure there is a problem, ask a `question`.
- Speak about the code, not the author, and name the concrete drawback. Add the reason only when the author cannot infer it.
- Point at code by its line instead of quoting it, and put a small, concrete fix in a GitHub `suggestion` block.
- List Blocking comments first. Leave Nits out when anything is Blocking, and keep them few otherwise.

For local changes, present the findings by `path:line` and level, with no verdict or draft; there is nothing to post.

Ask the user which comments to post and which verdict to use. They may approve any subset by number, reword a comment, or change the verdict. Stop here until the user answers.

## 6. Post

Skip this step for local changes.

Check that the pull request's head still equals `headRefOid`; when it moved, say so and stop, since the drafted lines may no longer match. Otherwise submit the approved comments as one review, built in a file outside the repository so the body survives any shell's quoting:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews --input <file>
```

The file holds `commit_id` (`headRefOid`), `event`, `body`, and `comments`, each comment with `path`, `line`, `side`, and `body`, plus `start_line` and `start_side` for a range. The label opens each `body`, since GitHub has no severity field. `event` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`.

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
