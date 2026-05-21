---
name: pr-review-triage
description: Use when the user asks to triage, plan, or address PR review comments — especially unresolved ones from GitHub Copilot's PR review bot. Triggers on "review comments", "PR feedback", "unresolved comments", "address the review", "plan the review", or similar. Always operates on the current branch's PR unless the user explicitly provides a different PR number.
---

# PR Review Triage

## When to use
User has a PR (often just reviewed by `copilot-pull-request-reviewer[bot]`) and wants a plan for unresolved threads. Plan only — no edits in this turn.

## Procedure

### 1. Resolve the PR number
Default behavior: detect the PR from the current branch. Do NOT ask the user.

```bash
gh pr view --json number,headRefName,baseRefName,state --jq .
```

- If this returns a PR with `state: OPEN`, use that number. Proceed silently — don't announce the number unless asked.
- If `state: MERGED` or `CLOSED`, tell the user the PR on this branch is closed and stop.
- If the command errors with "no pull requests found", tell the user: "No open PR for the current branch (`<branch>`). Push the branch and open a PR first." Stop.
- Only use a user-provided number if they explicitly include one in their message (e.g. "plan PR 1234"). Otherwise the current branch wins, even if the user mentions other PR numbers in passing.

### 2. Confirm local checkout
You're likely already on the PR branch (that's how step 1 worked). If not, `gh pr checkout <number>`. Needed so you can read the actual code at cited lines.

### 3. Fetch unresolved review threads
Run this directly with `gh` (no extensions):

```bash
gh api graphql -F owner='<owner>' -F repo='<repo>' -F pr=<number> -f query='
query($owner:String!,$repo:String!,$pr:Int!){
  repository(owner:$owner,name:$repo){
    pullRequest(number:$pr){
      reviewThreads(first:100){
        nodes{
          id
          isResolved
          isOutdated
          comments(first:20){
            nodes{ author{login} body path line diffHunk url }
          }
        }
      }
    }
  }
}'
```

Get `<owner>` and `<repo>` from `gh repo view --json owner,name --jq '.owner.login + " " + .name'`.

Then filter client-side: keep nodes where `isResolved == false`. Optionally drop `isOutdated == true` (user can override).

**If the query returns zero unresolved threads:** say so explicitly and stop. Don't invent comments. Don't fall back to `gh pr view --json comments` — that returns issue-level comments, not review threads, and conflating them is a common mistake.

**If the query errors with auth:** tell the user to run `gh auth status` and `gh auth refresh -s repo`. Don't try to work around it.

### 4. Read the referenced code
For each unresolved thread, open the file at `comments[0].path` around `comments[0].line` (read ~20 lines of context above and below). Triaging without reading the code produces bad plans.

### 5. Produce the plan — exact shape

**A. Triage table.** One row per thread:

| # | File:Line | Reviewer | Summary | Category | Rationale |
|---|-----------|----------|---------|----------|-----------|

Categories: **Agree** / **Partial** / **Disagree** / **Need-info**.

**B. Fix order.** Numbered list of threads to address, grouped by file to minimize churn. For each, one-line description of the intended change.

**C. Proposed dismissals.** Threads to skip, one-line justification each. User can override any of these.

**D. Open questions.** Anything in "Need-info" — list the specific question to ask the reviewer or the user.

### 6. Stop
Do not edit files. Do not resolve threads. Wait for user approval or revision of the plan.

## Hard rules
- Never ask the user for a PR number — always detect from the current branch first.
- Never auto-resolve threads. User-only decision.
- Never apply fixes in the same turn as planning.
- If a comment contains a `suggestion` block (triple-backtick `suggestion`), quote it verbatim in the table so the user can eyeball the proposed diff.
- Style/nitpick category → default to **Disagree, defer to linter/formatter** unless user says otherwise.
- Outdated threads (the line no longer exists) → default to **Disagree, code has moved** and flag for user confirmation.

## Common failure modes to avoid
- Don't query `pulls/{pr}/comments` REST endpoint — it returns *all* review comments including resolved ones, with no `isResolved` field. GraphQL `reviewThreads` is the only clean source.
- Don't trust `gh pr view --json reviews` for inline comments — it only returns review *summaries*, not the line-level threads.
- Don't assume the Copilot bot login is `copilot[bot]` — it's `copilot-pull-request-reviewer[bot]`. If filtering by reviewer, use the exact login.
