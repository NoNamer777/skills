---
name: code-review
description: Review a GitHub pull request, or the local changes when none is given, and draft a verdict with inline comments to post on approval.
argument-hint: "[<number> | <url> | <owner>/<repo>#<number>]"
disable-model-invocation: true
---

Review a pull request, or the local changes when no pull request is given, and advise on it: a verdict and a set of short review comments, anchored inline where possible. Advice changes nothing; the skill edits no code, and every review comment and submission waits for the user's approval.

Narrate the review as it runs. Before each step, say what you are about to do, including every worktree, branch, fetch, and post. Say what you run into as you meet it: a command git or `gh` refuses, a check that fails, a stop condition, an assumption you make, a step you skip and why.

The rules below are defaults. Where the repository's own agent instructions (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`) say otherwise about reviews, worktrees, or standards, follow the repository.

## Delegation

Sub agents do the review itself, in steps 4 and 5. The main agent does every other step, orchestrates the sub agents, and holds every exchange with the user. A sub agent knows only its brief, so give it the target, the checkout path, the command that prints the diff, and the exact shape of what it returns.

- Step 4 runs one sub agent per review area, and one that runs the checks.
- Step 5 runs three sub agents per finding.

No sub agent edits source files, posts, or removes anything. The checks sub agent may write build output, build images, and start apps and a browser tab, and stops what it started. Announce each batch before launching it: how many sub agents, and for what. When a sub agent fails or returns something unusable, run it once more; when it fails again, say so and treat its part as unassessed.

## 1. Resolve the target

With no argument, the target is the local changes: staged, unstaged, and untracked files (`git status --porcelain`). When there are none, say so and stop.

Otherwise read `<owner>`, `<repo>`, and `<number>` from the argument: a number belongs to the current repository (`gh repo view --json owner,name`), and a URL or `<owner>/<repo>#<number>` names its own. Fetch the pull request:

```bash
gh pr view <number> -R <owner>/<repo> --json author,state,title,body,url,baseRefName,headRefName,headRefOid,isCrossRepository,closingIssuesReferences
gh api repos/<owner>/<repo>/pulls/<number> --jq .author_association
```

The second command reads `authorAssociation`, which `gh pr view` has no field for.

Stop and say why when `state` is not `OPEN`. When the repository's instructions name a separate account for reviews, switch to it first, and switch back once the review is posted or the user decides to post nothing. Note whether `author.login` equals `gh api user --jq .login`, which reads the active account: the user may review their own pull request, which changes only how step 7 posts.

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

Note what this step created, for step 8: the worktree and its path (items 2 and 3) and the branch (item 3 only, since item 2 reuses an existing one). Report the path.

## 3. Read the change

Read the full diff: `gh pr diff <number> -R <owner>/<repo>` for a pull request, and `git diff HEAD` plus the untracked files for local changes. Read the code around each hunk, not only the hunk.

For a pull request, also read its description, the issues in `closingIssuesReferences`, and the existing review threads and comments, so the review does not repeat a point that is already raised.

## 4. Review

Launch one sub agent per area below, together, each briefed with the description, the linked issues, the points already raised, the checkout path, the command that prints the diff, the level table and the Blocking definition below. Tell each reviewer to read every changed file for its area only. Launch an area marked "when" only where the diff touches it.

The checks sub agent runs the pull request's code. When the head is in a fork (`isCrossRepository`) or the author association is not `OWNER`, `MEMBER`, or `COLLABORATOR`, ask the user before launching it; when they decline, skip it and treat the checks as unassessed.

- **Checks:** run locally what CI would run for this diff, and try the change in a browser where it renders. It returns no findings.
  - **Select** by relevance, never by completeness. Read the CI definitions in the checkout (`.github/workflows` and any other CI configuration), with their path filters and affected-project detection. Read the checks the repository's agent instructions name for a change, and, for a pull request, its required checks from `gh pr checks <number>`. Run a check when the diff touches its inputs or what it guards.
  - **Match** the checks to the change, where the repository has them. Documentation only: markdown lint, format check, and a link check where the repository has one. Code: lint, format check, type check, build, and the tests of the affected projects. A Dockerfile, its context, or the build it wraps: build the image locally, with no login and no push. A UI component or its styles: build Storybook and inspect the affected stories in a browser. Behavior a user can reach: the end-to-end tests, against the local dev server or the locally built image.
  - **Run** each check in its check-only form, never a fixer, with the repository's pinned tooling. Install first when dependencies are missing.
  - **Skip** a step only when it needs secrets, deploys, publishes, or targets another platform, when a tool it needs is unavailable, or when the diff cannot affect it. Never skip a step for its cost. List each skip with its reason.
  - **Browser:** when the diff affects something that renders (a web app, a Storybook story, a page a server returns), start it from the repository's own instructions, in the checkout and against local services only. Exercise what the diff changes: the changed screens, stories, endpoints, or commands, the behavior the description promises, the viewports and themes the diff or the issue names, and the narrowest and widest the project supports. Measure what a claim depends on, such as a width or an overflow, instead of estimating it. Watch for console and server errors. Use the user's own browser (Claude in Chrome) when a tool for it is available, over a built-in one, in a tab of your own that you close afterward, and touch no other tab. When the app needs configuration the checkout lacks, such as an `.env`, report the test as skipped instead of copying or inventing secrets.
  - **Stop** what you started.
  - **Return** each check and app test as pass, fail, or skipped, with the reason it was selected or skipped. For each failure return the command, the output that shows it, whether it points at files the diff changes, and the steps to reproduce it. Also return the files the run changed (`git status --porcelain`).
- **Scope and fit:** the change should exist, and the description and each linked issue match what the diff does. Report the goals it misses and the changes it does not declare. A large change (about 20 files, or several hundred hand-written changed lines), or one that mixes a refactor or reformatting with behavior, gets a note and a recommendation to split. Skim generated and lock files, and list every part skimmed.
- **Design:** read the files that hold the logic first, then the rest, and judge how the pieces fit the codebase.
- **Functionality:** edge cases, error handling, and concurrency when shared state or async code changes.
- **Security,** when the diff touches input handling, authentication, authorization, cryptography, secrets, serialization, or file, process, or query construction: review the change for security. Trace data from its sources to its sinks across each changed trust boundary, and check the impact on existing controls, new attack vectors, new integrations, and regressions. When unsure about authentication, authorization, cryptography, or secrets, raise a blocking Question and recommend a human reviewer for that part.
- **Tests:** new or changed logic has tests that would fail if the code broke. Read the tests as code: meaningful assertions, no false positives.
- **Complexity:** nothing more generic or elaborate than the problem needs.
- **Compatibility,** when the diff changes a public surface (exported API, CLI flags, config keys, file or wire formats, database schema, skill names and arguments): the break is declared and comes with a deprecation or migration path.
- **Dependencies,** when a manifest or lockfile changes: necessity, maintenance state, license, known vulnerabilities, lockfile consistency.
- **Performance:** only concrete risks visible in the diff, such as an unbounded query or repeated work in a loop.
- **Documentation:** updated when user-facing behavior, the build, or the release changes.
- **Naming and comments** (why, not what), and consistency with the surrounding code and the repository's rules.

Each reviewer returns its findings, or none. A finding has a `path:line`, a level, a one-sentence claim, the evidence it read, and a fix when it knows one. It leaves out what a formatter or linter enforces.

| Level      | When                                                                                        |
|:-----------|:--------------------------------------------------------------------------------------------|
| Blocking   | Merging makes things worse: see below                                                       |
| Suggestion | The change works and could be better; the author may decline                                |
| Question   | The intent or a choice is unclear; blocking when its answer could reveal a Blocking finding |
| Nit        | Taste, or a point no rule covers; never gates a verdict                                     |

A finding is Blocking when merging leaves the codebase or its users worse off than not merging: incorrect or unhandled behavior, a regression, a vulnerability, an undeclared break of a public surface, new logic without meaningful tests, complexity or duplication the change adds, or a description or linked issue the change misses or contradicts. A problem that predates the change is a Suggestion with a follow-up issue.

Merge findings from different areas that claim the same problem at the same place. A check or app test that fails at files the diff changes becomes a Blocking finding with its reproduction, and skips the vote; every other failure and every skip goes to the summary and prevents Approve. Done when every sub agent has returned or been reported as unassessed, and you hold the merged findings, the check results, and the list of what was skimmed, did not run, or did not assess.

## 5. Vote on the findings

Skip this step when there are fewer than 3 findings, and say so.

Launch three sub agents per finding, together. Brief each with only the claim, its `path:line`, the checkout path, and the level table and Blocking definition from step 4. Withhold the finding's level, its dimension, and the other votes, so each voter decides alone. Each voter tries to refute the claim from the code, then returns `confirm` or `refute`, the evidence, and a level.

Settle each finding:

- It survives when at least two of its voters confirm; otherwise drop it.
- Its level is the median of the confirming votes, ordered Nit, Suggestion, Blocking. A Question keeps the level it was raised with.
- It stays Blocking only when all three voters confirm and the median is Blocking; with two confirmations it becomes a Suggestion.

Keep each dropped finding's claim and `path:line` for step 6. Done when every finding has three votes and an outcome.

## 6. Present the review

Recommend one verdict for a pull request, from the findings that survived:

- **Request changes** when any finding is Blocking.
- **Approve** when nothing is Blocking, you reviewed all of the change, and it clearly improves the code even if imperfect. Suggestions and Nits ride along with an approval.
- **Comment** otherwise: a blocking Question is open, or you cannot vouch for the change because part of it is skimmed or unassessed, a check failed or could not run, or a security-sensitive part is beyond your confidence. Say what is missing.

Give the verdict with one line of reasoning, then the draft review. The summary body is a few sentences at most: the size note if there is one, what you skimmed, did not run, or did not assess, any failed or skipped check or app test, and at most one sincere line of praise. Anchor each inline comment on a line that is part of the diff; a finding about the change as a whole, or about a line outside the diff, goes in the summary.

Write each comment so the author can act on it at a glance:

- Open with its label: `issue (blocking):`, `suggestion:`, `question (blocking):` or `question (non-blocking):`, or `nitpick:`.
- Cover one finding per comment. Name the problem and the change you ask for in a sentence or two, and pair an `issue` with a fix or direction when you know one. When you are unsure there is a problem, ask a `question`.
- Speak about the code, not the author, and name the concrete drawback. Add the reason only when the author cannot infer it.
- Point at code by its line instead of quoting it, and put a small, concrete fix in a GitHub `suggestion` block.
- List Blocking comments first. Leave Nits out when anything is Blocking, and keep them few otherwise.

After the draft, list the findings dropped by vote, one line each with its claim and `path:line`, so the user can restore one.

For local changes, present the surviving findings by `path:line` and level, with no verdict or draft; there is nothing to post.

Ask the user which comments to post and which verdict to use. They may approve any subset by number, reword a comment, restore a dropped finding, or change the verdict. Stop here until the user answers.

## 7. Post

Skip this step for local changes.

Check that the pull request's head still equals `headRefOid`; when it moved, say so and stop, since the drafted lines may no longer match. Otherwise submit the approved comments as one review, built in a file outside the repository so the body survives any shell's quoting:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews --input <file>
```

The file holds `commit_id` (`headRefOid`), `event`, `body`, and `comments`, each comment with `path`, `line`, `side`, and `body`, plus `start_line` and `start_side` for a range. The label opens each `body`, since GitHub has no severity field. `event` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`.

On the user's own pull request, GitHub rejects `APPROVE` and `REQUEST_CHANGES`, so submit `COMMENT` and say that the verdict stays advice.

Done when the review is posted. Report its URL.

## 8. Offer to clean up

Skip this step when step 2 created nothing. Once the review is posted, or when the user decides to post nothing, offer to remove what step 2 created, naming the worktree path and the branch, and wait for the answer. The user may keep them to re-review after the author pushes.

On a yes, run from the original checkout, since a worktree cannot remove the folder you stand in:

```bash
git worktree remove <path>
git branch -d <headRefName>
```

Run the second command only for a branch step 2 created. Never pass `--force` or `-D`: when git refuses because of uncommitted or unpushed work, report that and leave it. Done when git has removed what the user approved, or has refused and the refusal is reported.
