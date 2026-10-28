---
name: addressing-pr-comments
description: Triage the open review feedback on a GitHub pull request you authored, then fix, commit, and reply to what you approve.
argument-hint: "[<number> | <url> | <owner>/<repo>#<number>]"
disable-model-invocation: true
---

Advise the author of a GitHub pull request on each piece of open feedback, then carry out the advice they approve. Advice changes nothing; every code change, commit, push, reply, and issue waits for the user's approval.

The rules below are defaults. Where the repository's own agent instructions (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`) say otherwise about commits, replies, or reviews, follow the repository.

## 1. Resolve the pull request

Read `<owner>`, `<repo>`, and `<number>` from the argument: a number belongs to the current repository (`gh repo view --json owner,name`), and a URL or `<owner>/<repo>#<number>` names its own. With no argument, take the pull request of the current branch from `gh pr view --json number,url`.

Fetch the pull request:

```bash
gh pr view <number> -R <owner>/<repo> --json author,headRefName,url,reviews,comments,commits
```

This skill serves the author. When `author.login` differs from `gh api user --jq .login`, stop and say so.

## 2. Fetch the review threads

Run the `review-threads.graphql` query that sits next to this file:

```bash
gh api graphql --paginate --slurp -F query=@<path-to>/review-threads.graphql -F owner=<owner> -F repo=<repo> -F number=<number>
```

Done when every page is read. Flag any thread whose `comments.totalCount` exceeds 100, since only its first 100 comments were read.

## 3. Sort the feedback into items

An **item** is one unit of advice: a whole review thread, a review summary with a body, or a conversation comment. Read a thread's full back-and-forth and advise on its current state.

- Skip resolved threads.
- Skip the author's own comments, except as part of a thread.
- A thread whose last comment is the author's, or a summary or conversation comment the author already answered later in the conversation, is **waiting on reviewer**: list it, take no action.
- Every other item is **actionable**. Tag a thread `outdated` when `isOutdated` is true, and tag an item `automated` when its author is a bot: `__typename` is `Bot`, `is_bot` is true, or the login ends in `[bot]`.

## 4. Investigate each actionable item

Read the code each item points at, in the local checkout when it holds the head branch and through the diff hunk otherwise. Check every factual claim the reviewer makes against that code. Compare the item's timestamp (`createdAt`, or `submittedAt` for a review) with the pull request's commits to find later commits that already handle it.

Put each item in one category:

| Category          | When                                 | Action                                                          |
|:------------------|:-------------------------------------|:----------------------------------------------------------------|
| Fix               | The reviewer is right                | Change the code, reply with the commit                          |
| Push back         | The request is wrong or not worth it | Reply with the reason and an alternative, leave the thread open |
| Answer            | It is a question                     | Answer in the reply, or in code (a comment, a clearer name)     |
| Already addressed | A later commit handles it            | Reply citing that commit                                        |
| Follow-up         | Valid, but outside this pull request | Draft an issue, reply linking it                                |
| Clarify           | The intent is ambiguous              | Ask the reviewer what they mean                                 |

Done when every actionable item has a category backed by code you read.

## 5. Present the advice

Number the items and group them by category, in the table's order. For each item give its link, `path:line`, its tags, the reviewer's ask in one line, the recommended action, and the draft reply. A Follow-up item also carries the draft issue title and body. Write each reply as concisely as a review comment: answer the point in a sentence or two, and leave `<sha>` as a placeholder where a commit will be cited.

List the waiting-on-reviewer items after the groups, one line each.

Ask the user which items to act on. They may approve any subset by number, change an item's category or wording, or approve everything. Items they leave out stay untouched; when an answer is ambiguous, ask again. Stop here until the user answers.

## 6. Check the branch

When an approved item changes code, the head branch must be checked out locally, with a clean working tree that is up to date with its remote: compare `git branch --show-current` with `headRefName`, run `git status --porcelain`, then `git fetch` and read the ahead and behind counts from `git status -sb`. When any check fails, stop and tell the user what to run, such as `gh pr checkout <number>`.

Replies that change no code skip this step.

## 7. Make the changes

Apply every approved Fix, and every Answer that belongs in the code.

A comment with a `suggestion` block proposes replacement text for lines `startLine` through `line` (or `line` alone). Apply it locally, never through GitHub's "Commit suggestion" button, so the commit stays signed, reviewed, and grouped by intent. When those lines still read as the reviewer saw them, apply the suggestion **verbatim**; otherwise adapt it to the current code and say so in the reply.

Run the narrowest checks the repository defines that cover the change (lint, type check, affected tests), as its scripts or contributing docs name them. Done when they pass. Report which checks ran; when one fails, report the failure and stop before committing.

## 8. Commit and push

Group the changes into commits by intent, one commit per intent. Draft each message in the repository's convention (its commitlint config or contributing docs), falling back to [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).

For every human reviewer whose suggestion a commit applies verbatim, add a trailer with their noreply address, looking up `id` and `name` with `gh api users/<login>`:

```text
Co-authored-by: <name, or login when name is empty> <<id>+<login>@users.noreply.github.com>
```

Adapted suggestions and suggestions from bots get no trailer. A verbatim suggestion stays in the commit of its intent, with one trailer per distinct reviewer.

Present the drafts and wait for approval, then commit and push.

## 9. Reply

Fill each `<sha>` placeholder with the short SHA of the pushed commit. When a reply's content changed beyond the SHA, because a suggestion was adapted or a check forced a different fix, present it again before posting.

For each approved item, in order:

1. **File the issue** for a Follow-up item with `gh issue create -R <owner>/<repo> --title <title> --body-file <file>`, and put its link in the reply.
2. **React** to the comment the reply answers, when one reaction matches the response: `THUMBS_UP`, `THUMBS_DOWN`, `LAUGH`, `HOORAY`, `CONFUSED`, `HEART`, `ROCKET`, or `EYES`. Pass the comment's node `id` (from the thread query, or `id` in `reviews` and `comments`):

   ```bash
   gh api graphql -f query='mutation($id: ID!, $c: ReactionContent!) { addReaction(input: {subjectId: $id, content: $c}) { reaction { content } } }' -f id=<node-id> -f c=<reaction>
   ```

3. **Post the reply** from a file outside the repository, so the body survives any shell's quoting.
   - In a thread, reply to its first comment:

     ```bash
     gh api repos/<owner>/<repo>/pulls/<number>/comments/<first-comment-databaseId>/replies -F body=@<file>
     ```

   - To a review summary or conversation comment, post a new comment that opens by quoting the part it answers: `gh pr comment <number> -R <owner>/<repo> --body-file <file>`.
4. **Resolve** a thread from an `automated` author once its reply is posted. Leave threads from human reviewers open for them to resolve.

   ```bash
   gh api graphql -f query='mutation($id: ID!) { resolveReviewThread(input: {threadId: $id}) { thread { isResolved } } }' -f id=<thread-id>
   ```

Done when every approved item has its reply posted.

## 10. Suggest a re-review

Name the human reviewers whose Fix items landed, and ask whether to re-request their review. On a yes, run `gh pr edit <number> -R <owner>/<repo> --add-reviewer <login>,<login>`.
