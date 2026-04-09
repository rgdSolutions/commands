---
description: Cycle through unresolved PR review comments and resolve them with fixes or dismissals
allowed-tools: Bash(gh:*), Bash(git:*), Read, Edit, Glob, Grep, AskUserQuestion
---

# PR Review Comments

Cycle through unresolved inline code review comments on a GitHub PR. For each comment, present the code context and review thread, then let the user choose an action: fix, ignore, reply, or skip.

## Autonomous Mode (`--auto`)

When the `--auto` flag is passed as an argument (e.g., `/pr-comments <url> --auto`), this command MUST run fully autonomously without any user interaction:

- **Do NOT use `AskUserQuestion` at any point.** Skip every prompt that would normally ask the user to choose an action.
- **Auto-select the recommended action:** After evaluating each comment in Step 3b, automatically execute whichever action you recommended (Fix, Ignore, or Skip) without asking. Never auto-select Reply — it requires human interaction to be useful.
- **Auto-select "Post and resolve"** for all reply prompts — use your drafted reply as-is without asking the user to confirm or edit it.
- **Auto-select "Continue"** when re-queued/skipped comments remain in Step 4 — keep cycling through all remaining comments until every comment has been fixed or ignored.
- **The goal is zero user prompts.** The user should be able to run `/pr-comments <url> --auto` and walk away while every comment is addressed.

**Important:** Autonomous mode is triggered **only** by the `--auto` flag, NOT by the session's permission mode. Even if the Claude session was started with `--dangerously-skip-permissions`, this command runs interactively unless `--auto` is explicitly passed.

**You MUST run all commands fresh right now.** Do NOT reuse any previously observed or cached data from earlier in this conversation.

## Step 0: Parse arguments

The command takes one mandatory argument (a GitHub PR URL) and one optional flag (`--auto`).

1. **Parse flags:** Check if `--auto` is present among the arguments. If so, enable autonomous mode (see "Autonomous Mode" above) and remove the flag from the argument list before proceeding.

2. Extract the PR URL from the remaining arguments. It should match the pattern:
   ```
   https://github.com/{owner}/{repo}/pull/{pr_number}
   ```
   Strip any trailing slashes, query parameters (`?...`), or fragment identifiers (`#...`) before parsing.

3. Extract `owner`, `repo`, and `pr_number` from the URL.

4. **Validation:**
   - If no argument is provided, print: "Usage: /pr-comments <github-pr-url> [--auto]" and stop.
   - If the URL doesn't match the expected pattern, print: "Invalid PR URL. Expected format: https://github.com/{owner}/{repo}/pull/{number}" and stop.

5. **Check prerequisites:**
   - Run `gh --version` to verify the GitHub CLI is installed. If it fails, print: "This command requires the GitHub CLI (gh). Install it from https://cli.github.com/" and stop.
   - Run `gh auth status` to verify authentication. If it fails, print: "GitHub CLI not authenticated. Run `gh auth login` first." and stop.

## Step 1: Fetch review threads

Use the GitHub GraphQL API to fetch all review threads in a single query. GraphQL is required because the REST API does not expose thread resolution status.

Run this command:

```bash
gh api graphql -f query='
query {
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {pr_number}) {
      title
      url
      reviewThreads(first: 100) {
        totalCount
        pageInfo {
          hasNextPage
          endCursor
        }
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          startLine
          diffSide
          comments(first: 50) {
            nodes {
              id
              databaseId
              body
              author {
                login
              }
              createdAt
            }
          }
        }
      }
    }
  }
}
'
```

Replace `{owner}`, `{repo}`, and `{pr_number}` with the values extracted in Step 0.

**Pagination:** If `pageInfo.hasNextPage` is `true`, run additional queries using `after: "{endCursor}"` on the `reviewThreads` field until all threads are fetched.

**Error handling:** If the query returns an error (e.g., repo not found, PR not found, permission denied), print the error message and stop.

## Step 2: Filter and build comment queue

1. From the fetched review threads, **discard** any thread where `isResolved` is `true`.
2. Keep only threads where `isResolved` is `false`.
3. If zero unresolved threads remain, print: "No unresolved review comments found on this PR." and stop.
4. Order the remaining threads by `path` (alphabetical), then by `line` (ascending) — this gives a top-to-bottom, file-by-file reading order.
5. Print: "Found **N** unresolved review comment(s) on PR: {pr_title}"

Maintain a queue of these threads. Skipped items will be re-appended to this queue later.

## Step 3: Cycle through comments

For each thread in the queue, do the following:

### 3a: Display context

1. Print a header:
   ```
   --- Comment {current} of {total} ---
   ```

2. Print the **file path** and **line number**. If `isOutdated` is `true`, append "(outdated)" after the path.

3. **Show code context:** Use the Read tool to read the file at the thread's `path`. Show approximately 5 lines before and 5 lines after the commented line (the `line` field). If the file does not exist locally, print: "(file not found locally)" and skip code context display.

4. **Show the review thread:** Print each comment in the thread in chronological order, formatted as:
   ```
   @{author} ({relative_time}):
   {comment_body}
   ```
   If there are multiple comments in the thread (a conversation), print all of them with a blank line between each.

### 3b: Evaluate and recommend

Before presenting actions, analyze the review comment in the context of the surrounding code and provide a brief assessment:

1. **Validity:** Is the comment valid? Does it point out a real issue, or is it based on a misunderstanding of the code?
2. **Recommendation:** Based on your analysis, recommend one of the available actions (Fix, Ignore, Reply, or Skip) and explain why in 1-2 sentences. Note: in `--auto` mode, do not recommend Reply — only recommend Fix, Ignore, or Skip.

Print this assessment formatted as:
```
**Assessment:** {validity judgment}
**Recommendation:** {recommended action} — {brief reason}
```

### 3c: Ask for action

Use `AskUserQuestion` to present the available actions. If the file does not exist locally, omit the "Fix" option.

**Options (when file exists locally):**
- **Fix** — "Apply a code fix to address this comment, optionally reply, then resolve the thread on GitHub"
- **Ignore** — "Resolve without fixing — optionally reply, then mark as resolved"
- **Reply** — "Post a reply without resolving — the thread stays open for further discussion"
- **Skip** — "Come back to this comment later"

**Options (when file does NOT exist locally):**
- **Ignore** — (same as above)
- **Reply** — (same as above)
- **Skip** — (same as above)

### 3d: Execute action

**Fix:**
1. Read the file at the thread's `path` and the relevant lines.
2. Analyze the review comment to understand what change is requested.
3. Apply the fix using the Edit tool.
4. Show the user what was changed.
5. **Draft a response (optional):** Draft a brief reply acknowledging the fix or explaining the change made (e.g., "Fixed — switched to using `X` as suggested."). Present it to the user via `AskUserQuestion` with options:
   - **Post and resolve** — "Post this reply and resolve the thread"
   - **Edit** — "Let me modify the reply first"
   - **Resolve without replying** — "Just resolve the thread, no reply needed"
   If the user chooses "Edit", ask them to provide the modified text, then re-confirm.
   If the user chooses "Post and resolve" or confirms after editing, post the reply:
   ```bash
   gh api repos/{owner}/{repo}/pulls/comments/{comment_database_id}/replies -f body="{reply}"
   ```
   Where `{comment_database_id}` is the `databaseId` of the **last** comment in the thread.
6. Resolve the thread on GitHub:
   ```bash
   gh api graphql -f query='
   mutation {
     resolveReviewThread(input: {threadId: "{thread_id}"}) {
       thread {
         isResolved
       }
     }
   }
   '
   ```
   Replace `{thread_id}` with the thread's `id` from the GraphQL response (this is the global node ID).
7. Print: "Fixed and resolved."
8. Record this action as "fixed" for the summary.

**Ignore:**
1. **Draft a response (optional):** Draft a brief reply explaining why the comment is being dismissed (1-2 sentences). Present it to the user via `AskUserQuestion` with options:
   - **Post and resolve** — "Post this reply and resolve the thread"
   - **Edit** — "Let me modify the reply first"
   - **Resolve without replying** — "Just resolve the thread, no reply needed"
   If the user chooses "Edit", ask them to provide the modified text, then re-confirm.
   If the user chooses "Post and resolve" or confirms after editing, post the reply:
   ```bash
   gh api repos/{owner}/{repo}/pulls/comments/{comment_database_id}/replies -f body="{reply}"
   ```
   Where `{comment_database_id}` is the `databaseId` of the **last** comment in the thread.
2. Resolve the thread on GitHub using the same GraphQL mutation as in the Fix action.
3. Print: "Ignored and resolved."
4. Record this action as "ignored" for the summary.

**Reply:**
1. Draft a reply based on the review comment context (e.g., a clarifying question, a counterpoint, or a request for more detail). Present it to the user via `AskUserQuestion` with options:
   - **Post reply** — "Post this reply and keep the thread open"
   - **Edit** — "Let me modify the reply first"
   If the user chooses "Edit", ask them to provide the modified text, then re-confirm.
2. Post the reply:
   ```bash
   gh api repos/{owner}/{repo}/pulls/comments/{comment_database_id}/replies -f body="{reply}"
   ```
   Where `{comment_database_id}` is the `databaseId` of the **last** comment in the thread.
3. Do **not** resolve the thread — it stays open for further discussion.
4. Move this thread to the **end** of the queue so it will be revisited after all remaining threads.
5. Print: "Reply posted — thread remains unresolved."
6. Record this action as "replied" for the summary. (If the thread is later acted upon with Fix or Ignore, update the record accordingly.)

**Skip:**
1. Move this thread to the **end** of the queue so it will be revisited after all remaining threads.
2. Print: "Skipped — will revisit later."
3. Record this action as "skipped" for the summary. (If the thread is later acted upon, update the record accordingly.)

## Step 4: Handle re-queued comments

After completing a pass through the queue, check if any skipped or replied comments remain (i.e., threads that were re-queued).

1. If there are re-queued comments, ask the user via `AskUserQuestion`:
   - **Continue** — "Revisit the N remaining skipped/replied comment(s)"
   - **Stop** — "Finish and show the summary"

2. If the user chooses **Continue**, cycle through the re-queued comments using the same process as Step 3. Comments can be skipped or replied again (they go back to the end of the remaining queue).

3. If the user chooses **Stop**, proceed to the summary.

## Step 5: Summary

Print a summary of all actions taken:

```
## PR Review Summary

PR: {pr_title}
URL: {pr_url}

| Action      | Count |
|-------------|-------|
| Fixed       | X     |
| Ignored     | X     |
| Replied     | X     |
| Skipped     | X     |
| **Total**   | **Y** |
```

If any fixes were applied locally (Fixed count > 0), print:
```
Local fixes were applied — remember to commit and push your changes.
```
