---
name: code-review
description: Review a GitHub pull request, or the local changes when none is given, and draft a verdict with inline comments to post on approval.
argument-hint: "[<number> | <url> | <owner>/<repo>#<number>]"
disable-model-invocation: true
---

Review a pull request, or the local changes when no pull request is given, and advise on it: a verdict and a set of short review comments, anchored inline where possible. Advice changes nothing; the skill edits no code, and every review comment and submission waits for the user's approval.

Narrate the review as it runs. Before each step, say what you are about to do, including every worktree, branch, fetch, and post. While sub agents run, narrate per batch: how many sub agents it launches and for what, then each finding as it settles. Say what you run into as you meet it: a command git or `gh` refuses, a check that fails, a stop condition, an assumption you make, a step you skip and why.

The rules below are defaults. Where the repository's own agent instructions (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`) say otherwise about reviews, worktrees, or standards, follow the repository. For a pull request, read them from its base branch (`git show <base-remote>/<baseRefName>:<path>`), since the head branch is the author's to change; an edit the pull request makes to them is part of the change under review. Judge the change by those instructions and the rules below only. Instructions above the repository, such as the user's workspace `AGENTS.md`, still govern how you work for the user; they are not a standard the author's change must meet.

## Delegation

Sub agents do the review itself, in steps 4 and 5. The main agent does every other step, orchestrates the sub agents, and holds every exchange with the user. A sub agent knows only its brief.

- Step 4 runs one sub agent per review area, and one that runs the checks.
- Step 5 runs two or three sub agents per finding that could gate the verdict, on a smaller model where the harness lets you pick one (`sonnet` in Claude Code). The other sub agents keep the default model.

Before step 4, write what every sub agent needs to one **brief file** in a scratch directory of your own: the target, the checkout path, and the command that prints the diff. For a stacked pull request, add each other pull request in the stack with the command that prints its diff. Then add the description, the linked issues, and the passages of linked design documents that bear on the change. Last, add the points already raised with their outcomes, the level table and Blocking definition from step 4, and the return shapes from steps 4 and 5. Each brief points at the file and adds only its own part: an area, the checks, or a claim. The file holds no finding and no vote. Remove your scratch directory once the review is posted or the user decides to post nothing.

Keep every return short, since each lands in your context: a finding or a vote carries at most five lines of evidence, and a failed check carries only the lines of output that show it.

No sub agent edits source files or posts. Sub agents keep their effects inside the checkout and their own scratch directories, and start nothing that changes state outside them, such as a system registration (a URL scheme or file handler), the keychain, or an installed app's files. The checkout is the review's own worktree: the main clone and other worktrees, with their dependency folders, are outside it. A sub agent that needs other dependencies installs them into its scratch directory.

Make every scratch directory, yours and each sub agent's, with `mktemp -d /tmp/review-<repo>-<number>.XXXX`. It sits under `/tmp` because a long `TMPDIR`, such as macOS's `/var/folders/...`, pushes a tool's socket path past its length limit. Each sub agent points `TMPDIR` at it (and `-Djava.io.tmpdir` for a JVM), and runs the repository's tools from inside the checkout, so its pinned runtimes apply. It may fill the user-level caches of those tools (a package store, `~/.gradle`, a toolchain download) and lists what it added. Before it returns, it stops what it started, removes its scratch directories, and returns the path of any it could not remove.

The checks sub agent alone writes into the checkout: build output, and images under a tag of its own. It may also start apps and a browser tab. A voter reads and measures through the repository, the checkout, and the browser, and starts only what a measurement needs; its brief names any other tool it may use, read-only. Launch no more sub agents at once than the harness runs, counting those still running: Claude Code refuses a launch past 20, or past `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` when set. When a sub agent is cut off, by a usage limit say, and the harness can resume it (`SendMessage` in Claude Code), resume it so it keeps its work. When a sub agent fails or returns something unusable, run it once more; when it fails again, say so and treat its part as unassessed. A sub agent that is stopped or fails leaves its servers, tabs, and scratch directories behind: list them (listening ports, `/tmp`, the browser's tabs) and remove what you can attribute to it before its rerun.

## 1. Resolve the target

With no argument, the target is the local changes: staged, unstaged, and untracked files (`git status --porcelain`). When there are none, say so and stop.

Otherwise read `<owner>`, `<repo>`, and `<number>` from the argument: a number belongs to the current repository (`gh repo view --json owner,name`), and a URL or `<owner>/<repo>#<number>` names its own. Fetch the pull request:

```bash
gh pr view <number> -R <owner>/<repo> --json author,state,title,body,url,baseRefName,headRefName,headRefOid,isCrossRepository,closingIssuesReferences,mergeable
gh api repos/<owner>/<repo>/pulls/<number> --jq .author_association
```

The second command reads `authorAssociation`, which `gh pr view` has no field for.

Stop and say why when `state` is not `OPEN`. When `mergeable` is `CONFLICTING`, say so, and name the conflicting files in the summary. A finding that holds only once the base is merged in is still a finding of this change; it is measured in a scratch clone with the base merged in. Reading and drafting use the user's own account. A separate account for reviews comes from the user's own instructions, or from the repository's when the user confirms it here; step 7 switches to it just before posting, and back right after. Note whether `author.login` equals the login that will post: the review account when one is named, else the result of `gh api user --jq .login`. The user may review their own pull request, which changes only how step 7 posts.

## 2. Get the code

Local changes are reviewed in place; skip to step 3.

For a pull request, review its head branch in a local checkout, in this order:

1. **Worktree.** When `git worktree list` shows `headRefName` checked out, use that worktree.
2. **Branch.** When the current checkout is on `headRefName`, use it. When the branch exists locally and is checked out nowhere (`git branch --list <headRefName>`), add a worktree for it as in the next item, without `-b`, so the current checkout stays as it is.
3. **Neither.** Create a worktree at the location the repository's instructions name, else at `../.worktrees/<repo>/<slug>`, where `<slug>` is the part of the branch name after its last `/`. Detach it at the base branch, then check the pull request out inside it, which also sets up a fork's remote:

   ```bash
   git worktree add --detach <path> <base-remote>/<baseRefName>
   cd <path> && gh pr checkout <number> -R <owner>/<repo>
   ```

Bring the checkout to the pull request's head: run `git fetch` and fast-forward a branch that is only behind with `git merge --ff-only`. Done when `git rev-parse HEAD` equals `headRefOid`. When the checkout has diverged, is ahead, or has uncommitted changes, stop and tell the user rather than review code that is not the pull request's.

Note what this step created, for step 8: the worktree and its path (items 2 and 3) and the branch (item 3 only, since item 2 reuses an existing one). Report the path.

## 3. Read the change

Read the full diff: `gh pr diff <number> -R <owner>/<repo>` for a pull request, and `git diff HEAD` plus the untracked files for local changes. Read the code around each hunk, not only the hunk.

For a pull request, also read its description, the issues in `closingIssuesReferences`, and each issue its title, description, or branch name refers to in the repository's tracker (Jira, say), with its comments, read through the tracker's read-only tools. Read the design documents these link, such as an RFC, an ADR, or a Confluence page, through the same read-only tools, and keep the passages that bear on the change for the brief file. Then read every review with its body, its inline comments, and every reply to them, and every conversation comment, on this pull request and on any closed one with the same head branch (`gh pr list -R <owner>/<repo> --state closed --head <headRefName>`):

```bash
gh pr view <number> -R <owner>/<repo> --json reviews,comments
gh api repos/<owner>/<repo>/pulls/<number>/comments --paginate
```

The second command returns the inline comments; a reply carries the `in_reply_to_id` of the comment it answers. Done when you hold every point already raised, each with its outcome: fixed (with its commit), declined (with the reason), or open. A reviewer raises such a point again only when it is still open or its fix does not hold.

A pull request is **stacked** when its base branch is not the repository's default branch (`gh repo view -R <owner>/<repo> --json defaultBranchRef`), or when an open pull request is based on its head branch. Walk the stack both ways:

```bash
gh pr list -R <owner>/<repo> --state open --head <branch> --json number,baseRefName
gh pr list -R <owner>/<repo> --state open --base <branch> --json number,headRefName
```

The first finds the pull request below a branch, and the second those above it. Walk down from `baseRefName` until the base is the default branch or no open pull request has it as head, and up from `headRefName` until none is based on it. Read every other pull request in the stack in its entirety: its description, its full diff, and its reviews, comments, and replies, as above. The stack is context only. The review covers this pull request's diff, and a line elsewhere in the stack becomes a finding only when this change makes it false.

A **companion** is a pull request in another repository that ships the same issue, such as the client half of a server change. Find companions through the description, the linked issues, and `gh search prs <issue key> --owner <owner>`, and read each through `gh` like a stack member, for context only.

## 4. Review

Launch one sub agent per area below, together, each pointed at the brief file and told its area. Tell each reviewer to read every changed file for its area only. Hand each question you want settled, such as one a point already raised leaves open, to exactly one area: the one it belongs to. Launch an area marked "when" only where the diff touches it. Cover a small change (about 10 files or fewer) with fewer reviewers: fold Complexity and Usability into Design, Performance into Functionality, and Naming and comments into Documentation.

The checks sub agent runs the pull request's code. When the head is in a fork (`isCrossRepository`) or the author association is not `OWNER`, `MEMBER`, or `COLLABORATOR`, ask the user before launching it; when they decline, skip it and treat the checks as unassessed.

- **Checks:** run locally what CI would run for this diff, and try the change in a browser where it renders. It returns no findings.
  - **Select** by relevance, never by completeness. Read the CI definitions in the checkout (`.github/workflows` and any other CI configuration), with their path filters and affected-project detection, with `<base-remote>/<baseRefName>` as the base, which for a stacked pull request is the branch below it. Read the checks the repository's agent instructions name for a change, and, for a pull request, its required checks and their results on `headRefOid` from `gh pr checks <number>`. It exits 1 while any check fails or is pending, so read its output whatever the exit code. Run a check when the diff touches its inputs or what it guards.
  - **Match** the checks to the change, where the repository has them. Documentation only: markdown lint, format check, and a link check where the repository has one. Code: lint, format check, type check, build, and the tests of the affected projects. A Dockerfile, its context, or the build it wraps: build the image locally, with no login and no push. A UI component or its styles: build Storybook and inspect the affected stories in a browser. Behavior a user can reach: the end-to-end tests, against the local dev server or the locally built image. Every change: the commit lint over its commits and, for a pull request, over its title, where the repository has one.
  - **Run** each check in its check-only form, never a fixer, with the repository's pinned tooling, or the runtime CI sets up when the repository pins none. Install first when dependencies are missing.
  - **Skip** a check that passed in the pull request's CI on `headRefOid`, and spend the time on what CI does not cover: the browser, and the selected checks CI does not run. Skip any other step only when it needs secrets, deploys, publishes, or targets another platform, when a tool it needs is unavailable, when running it would change state outside the checkout, or when the diff cannot affect it. Never skip a step for its cost. A check that cannot start for a missing credential, private registry, or login is a skip that needs secrets: name what it needs. List each skip with its reason, including the linters the changed file types call for that are not installed.
  - **Browser:** when the diff affects something that renders (a web app, a Storybook story, a page a server returns), start it from the repository's own instructions, in the checkout and against local services only, in the background, and poll its URL until it answers or a timeout passes. Exercise what the diff changes: the changed screens, stories, endpoints, or commands, the behavior the description promises, the viewports and themes the diff or the issue names, and the narrowest and widest the project supports. Measure what a claim depends on, such as a width or an overflow, instead of estimating it. When the window cannot get as narrow as the narrowest viewport, measure in a same-origin iframe of that width over a static copy of the build. The app's own server may forbid framing, and a classic scrollbar takes about 15 px. Otherwise return the width as untested, so the main agent can ask the user to dock the dev tools at the side and widen them, which narrows the page. Watch for console and server errors. Use the user's own browser (Claude in Chrome) when a tool for it is available, over a built-in one, in a tab of your own that you close afterward, and touch no other tab. When the app needs configuration the checkout lacks, such as an `.env`, report the test as skipped instead of copying or inventing secrets.
  - **Stop** what you started. Keep the build output and the images you built, which voters use, and return the images' names; the main agent removes them once the vote is settled.
  - **Return** each check and app test as pass, fail, or skipped, with the reason it was selected or skipped. For each failure return the command, the output that shows it, whether it points at files the diff changes, and the steps to reproduce it. Run every failing check at `<base-remote>/<baseRefName>` too, in a worktree inside your scratch directory, and return whether it fails there. When it passes there and the head branch forks from an older base commit, also run it on the merge of the two (`git merge-tree --write-tree`), and return a **stale-base failure** when that passes. Then remove the worktree with `git worktree remove --force <path>` and check that `git worktree list` no longer shows it. A failure you meet outside the selected checks comes back as an observation, not a failed check. Also return the files the run changed (`git status --porcelain`). A measurement that shows a defect, such as a slow response or an overflow, comes back as a measured fact with its numbers, not as a finding.
- **Scope and fit:** the change should exist, and the description and each linked issue match what the diff does. Report the goals it misses and the changes it does not declare. Compare with how the repository landed earlier changes of the same kind (`git log` of sibling files), and report what they changed that this one does not, such as documentation, tests, or registrations in shared lists. A large change (about 20 files, or several hundred hand-written changed lines), or one that mixes a refactor or reformatting with behavior, gets a note and a recommendation to split. Skim generated and lock files, and list every part skimmed.
- **Design:** read the files that hold the logic first, then the rest, and judge how the pieces fit the codebase.
- **Functionality:** edge cases, error handling, and concurrency when shared state or async code changes.
- **Security,** when the diff touches input handling, authentication, authorization, cryptography, secrets, serialization, or file, process, or query construction, or bumps a dependency that does: review the change for security. Trace data from its sources to its sinks across each changed trust boundary, and check the impact on existing controls, new attack vectors, new integrations, and regressions. When unsure about authentication, authorization, cryptography, or secrets, raise a blocking Question and recommend a human reviewer for that part.
- **Tests,** when the diff adds or changes logic: new or changed logic has tests that would fail if the code broke. Read the tests as code: meaningful assertions, no false positives.
- **Complexity:** nothing more generic or elaborate than the problem needs.
- **Compatibility,** when the diff changes a public surface (exported API, CLI flags, config keys, file or wire formats, database schema, skill names and arguments): the break is declared and comes with a deprecation or migration path.
- **Dependencies,** when a manifest or lockfile changes: necessity, maintenance state, license, known vulnerabilities, lockfile consistency.
- **Performance,** when the diff touches a query, a loop over data, a rendering path, or what a bundle contains: only concrete risks visible in the diff, such as an unbounded query or repeated work in a loop.
- **Documentation:** updated when user-facing behavior, the build, or the release changes.
- **Usability,** when the diff changes output a person reads, such as labels, bot comments, CLI output, or notifications: its wording, its clarity, and how it looks where it appears, such as a label's color.
- **Naming and comments** (why, not what), and consistency with the surrounding code and the repository's rules.

Each reviewer returns its findings, or none. A finding has a `path:line`, a level, a one-sentence claim, the evidence it read, and a fix when it knows one. Its `path:line` is `none` for a point about the change as a whole, or about a file the diff should have changed and did not. It leaves out what a formatter or linter enforces.

| Level      | When                                                                                                               |
|:-----------|:-------------------------------------------------------------------------------------------------------------------|
| Blocking   | Merging makes things worse: see below                                                                              |
| Suggestion | The change works and could be better; the author may decline                                                       |
| Question   | The intent or a choice is unclear; the reviewer labels it blocking when its answer could reveal a Blocking finding |
| Nit        | Taste, or a point no rule covers; never gates a verdict                                                            |

A finding is Blocking when merging leaves the codebase or its users worse off than not merging. That covers incorrect or unhandled behavior, a regression, a vulnerability, an undeclared break of a public surface, new logic without meaningful tests (a check that every earlier change of the same kind landed with, and that an acceptance criterion of the linked issue depends on, is part of them), and complexity or duplication the change adds. It also covers a linked issue the change misses or contradicts, a description that contradicts the behavior the change ships, and a break of a rule the repository's instructions make mandatory. It covers documentation that is false after the change, too, code comments included, wherever it sits in the repository: a statement the change makes stale, or one it adds that is wrong. A missing entry is a Suggestion. A description that misstates what the change tests or covers is a Suggestion, since correcting the sentence settles it. A problem that predates the change is a Suggestion with a follow-up issue. So is a statement already false at the base (`git show <base-remote>/<baseRefName>:<path>`), which the reviewer reads before calling one stale. A statement the change adds that claims behavior the code lacks stays Blocking even when the behavior predates the change.

Merge findings from different areas that claim the same problem at the same place into one finding, whatever level each area gave it. A merged finding votes when any area rated it Blocking or a blocking Question, and otherwise takes the highest level given, ordered Nit, non-blocking Question, Suggestion. Two claims stay two findings, even at one place: a wrong default value (Blocking) and the unclear name of its option (Nit). A check or app test that fails at files the diff changes becomes a Blocking finding with its reproduction, and skips the vote, unless it also fails at the base. A failure that also occurs at the base is a **pre-existing failure**, and a stale-base failure comes with a recommendation to rebase; both go to the summary. A failure over the commits or the title, such as a commit lint, is a Suggestion unless the repository's CI enforces it. Every other failure goes to the summary and prevents Approve. A skip whose check passed in the pull request's CI on `headRefOid` goes to the user in step 6, outside the summary. Every other skip goes to the summary, and prevents Approve unless the repository has no such check or the diff cannot affect it. Done when every sub agent has returned or been reported as unassessed, you have checked each finding's `path:line` against the head, and you hold the merged findings, the check results, and the list of what was skimmed, did not run, or did not assess.

## 5. Vote on the findings

Vote on the findings that could gate the verdict: each Blocking finding and each blocking Question. Suggestions, Nits, and non-blocking Questions skip the vote and go to step 6 unvoted. Skip this step when no finding could gate the verdict, and say so.

Launch two voters per finding, for as many findings as half the free sub agent slots allows, so a finding's voters run in one batch. Launch the findings that decide the verdict first: Blocking findings before blocking Questions. Launch a third voter only when its vote can change the outcome: one of the first two confirms and the other does not, or both confirm and either rates the finding Blocking, under any of its conditions. Brief each voter with the brief file and the claim, stated as one effect a voter can confirm or refute, with its `path:line`. Withhold the finding's level, its dimension, and the other votes. Keep the votes of a finding still in vote in your own context, out of any folder a voter can read, so each voter decides alone. Once a finding settles, add its outcome to a tally file in your scratch directory, so a compaction mid-vote loses no settled outcome. Each voter tries to refute the claim from the code. When the claim is about runtime behavior or a measurable quantity, such as a response time, a memory peak, or a width, the voter measures it through the command the claim names (the build target, the CLI, the built app); a harness that calls one function gives an estimate. It may use the build output and images the checks sub agent kept, measures with what the change ships with (the repository's own tools, fixtures, fonts, and styles), and compares a secret without printing it (a count or a hash). For a page it uses the user's own browser (Claude in Chrome) when a tool for it is available. It works in a tab of its own that it closes afterward and touches no other tab, since the voters share the browser. It returns `confirm`, `refute`, or `unverifiable` (the repository and the browser hold no evidence either way), the evidence with each number marked measured or estimated, a level (Nit, Suggestion, or Blocking), and any other defect it measured on the way, as a measured fact. A condition is a setting the claim holds under, such as an input size, a platform, a configuration flag, or a viewport width; a claim that names several gets one outcome and one level per condition, and its reasoning gets one outcome as a whole. A level that depends on a project requirement, a supported runtime version say, cites the passage of the repository's docs that states it, found by searching the docs, `CONTRIBUTING.md`, and the agent instructions for the requirement's terms; when none does, the voter names what it searched.

Settle each finding:

- It survives when at least two of its voters confirm; otherwise drop it.
- Settle a claim with several conditions per condition, by the same two-vote rule and the level rules below; the conditions that survive form the finding, at the highest level among them.
- When no voter confirms and at least one returns `unverifiable`, the claim is not refuted: it goes to step 6 as a non-blocking Question for the author.
- Its level is the median of the confirming votes, ordered Nit, Suggestion, Blocking, and the lower of the middle two for an even count. A Question keeps the blocking or non-blocking label it was raised with, unless the claim voted on states a defect: the median then replaces the label.
- It stays Blocking only when all three voters confirm and the median is Blocking; with two confirmations it becomes a Suggestion.
- When voters disagree on a fact that one command settles, such as what a file contains or how earlier changes landed, run the command and settle by the fact.
- A finding that survives on estimates alone stays and is marked unverified for step 6, with the measurement that would settle it and what that measurement needs.
- When every voter refutes the claim as worded but confirms a narrower fact (the timeout happens only with a cold cache, say), drop the claim and keep the narrower fact.
- A measured fact that no finding covers goes to step 6 as an unvoted Suggestion, marked with who measured it: a narrower fact the voters kept, a defect a voter measured beyond its claim, or a fact from the checks sub agent. A measured fact that backs an unvoted finding gives it its numbers and the same mark.

Keep each dropped finding's claim and `path:line` for step 6. Done when every voted finding has the votes it needs and an outcome.

## 6. Present the review

Recommend one verdict for a pull request, from the findings that survived:

- **Request changes** when any finding is Blocking.
- **Approve** when nothing is Blocking, you reviewed all of the change, and it clearly improves the code even if imperfect. Suggestions and Nits ride along with an approval.
- **Comment** otherwise: a blocking Question is open, or you cannot vouch for the change because part of it is skimmed or unassessed, a failure or skip that step 4 says prevents Approve, or a security-sensitive part is beyond your confidence. Say what is missing.

Give the verdict with one line of reasoning, then the draft review. The summary body opens with each Blocking finding and blocking Question that has no line in the diff, one labeled paragraph each. A few sentences follow at most: the size note if there is one, the conflicting files, and what you skimmed, did not run, or did not assess. They name any failed check or app test and any skip step 4 sends to the summary, and end with at most one sincere line of praise. Anchor each inline comment on a line that is part of the diff; a finding about the change as a whole, or about a line outside the diff, goes in the summary.

Write each comment so the author can act on it at a glance:

- Open with its label: `issue (blocking):`, `suggestion:`, `question (blocking):` or `question (non-blocking):`, or `nitpick:`.
- Cover one finding per comment. Name the problem and the change you ask for in a sentence or two, and pair an `issue` with a fix or direction when you know one. When you are unsure there is a problem, ask a `question`.
- Speak about the code, not the author, and name the concrete drawback. Add the reason only when the author cannot infer it.
- Point at code by its line instead of quoting it, and put a small, concrete fix in a GitHub `suggestion` block.
- List Blocking comments first. Leave Nits out when anything is Blocking, and keep them few otherwise.

After the draft, list for the user:

- The findings dropped by vote, one line each with its claim and `path:line`, so the user can restore one.
- The findings marked unverified, each with the measurement that would settle it and what it needs.
- The Suggestions and Nits you left out of the draft, one line each.
- The checks skipped because they passed in the pull request's CI on `headRefOid`.

For local changes, present the surviving findings by `path:line` and level, with no verdict or draft; there is nothing to post.

Ask the user which comments to post and which verdict to use. They may approve any subset by number, reword a comment, restore a dropped finding, add a comment of their own, or change the verdict. Stop here until the user answers.

## 7. Post

Skip this step for local changes.

Check that the pull request's head still equals `headRefOid`; when it moved, say so and stop, since the drafted lines may no longer match. Otherwise switch to the review account when step 1 names one, and submit the approved comments as one review, built in a file outside the repository so the body survives any shell's quoting:

```bash
gh api repos/<owner>/<repo>/pulls/<number>/reviews --input <file>
```

The file holds `commit_id` (`headRefOid`), `event`, `body`, and `comments`, each comment with `path`, `line`, `side`, and `body`, plus `start_line` and `start_side` for a range. The label opens each `body`, since GitHub has no severity field. `event` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`.

On the user's own pull request, GitHub rejects `APPROVE` and `REQUEST_CHANGES`, so submit `COMMENT` and say that the verdict stays advice.

Switch back to the user's account right after the post, whether it succeeded or not. Done when the review is posted and the user's account is active again. Report the review's URL.

## 8. Offer to clean up

Skip this step when step 2 created nothing. Once the review is posted, or when the user decides to post nothing, offer to remove what step 2 created, naming the worktree path and the branch, and wait for the answer. The user may keep them to re-review after the author pushes.

On a yes, run from the original checkout, since a worktree cannot remove the folder you stand in:

```bash
git worktree remove <path>
git branch -d <headRefName>
```

Run the second command only for a branch step 2 created. Never pass `--force` or `-D`: when git refuses because of uncommitted or unpushed work, report that and leave it. Done when git has removed what the user approved, or has refused and the refusal is reported.
