---
name: slang-pr-resolve-comments
description: Resolve GitHub PR review feedback and CI failures. Use when asked to monitor a PR, handle LLM review threads, report draft/WIP/DNI status and review-readiness notices without treating them as blockers, leave human review threads for human resolution, record specific reviewer directives (from human or LLM reviewers) in the PR description so they are not reverted on later passes, be conservative about edits once the PR is approved/LGTM, fix failing checks, rebase merge conflicts, and keep watching until the PR is merged or closed.
argument-hint: "<PR URL or number> [--single-pass] [--wsl]"
allowed-tools: Bash Read Write Edit Grep Glob ScheduleWakeup
required-capabilities: shell git github-cli file-read file-edit search
---

# Resolve GitHub Review Feedback

Use this skill to keep a GitHub PR moving until it is merged or closed: each pass makes CI checks pass and addresses and resolves LLM review threads, and once no agent-actionable work remains the skill keeps watching at a slower cadence for later human feedback. Human-owned threads are left unresolved — they are outside the agent's control and must be resolved by the human reviewers themselves.

## Workflow Overview

The diagram below summarizes the full flow. Each later section documents the details; this is the map.

```mermaid
flowchart TD
    Start([Invoke skill with PR arg]) --> Parse[Parse args: PR, --single-pass, --wsl<br/>select git/gh tools]
    Parse --> Auth{gh auth OK &<br/>push permission?}
    Auth -->|No| StopBlocked([Stop: report missing credentials])
    Auth -->|Yes| Dirty{Local working<br/>tree dirty?}
    Dirty -->|Yes| AskUser{Ask user: commit / stash / abort}
    AskUser -->|Commit / Stash| Loop
    AskUser -->|Abort| StopAbort([Stop: user aborted])
    Dirty -->|No| Loop

    subgraph Loop [Main Loop - one pass]
        Checkout[Checkout PR branch, fetch,<br/>submodule update] --> Inspect[Inspect state, checks, mergeability,<br/>readiness notices, review threads]
        Inspect --> Approved{Approval / LGTM<br/>signal present?}

        Approved -->|Yes| ChangesReq{Human requested changes,<br/>user asked, or required CI<br/>fix needed?}
        ChangesReq -->|No| NoEdit[Be conservative:<br/>make no discretionary edits]
        ChangesReq -->|Yes| Work
        Approved -->|No| Work

        Work[Fix actionable review feedback and CI failures<br/>respecting recorded reviewer directives] --> Conflict{Merge conflicts<br/>DIRTY?}
        Conflict -->|Yes| Rebase[Rebase onto base, resolve,<br/>submodule update]
        Conflict -->|No| Commit
        Rebase --> Commit[Commit as new commits<br/>and push]
        Commit --> Desc[Update PR description if stale;<br/>record reviewer directives]
        Desc --> Threads[Reply to LLM threads & resolve addressed ones;<br/>leave human threads; log reviewer directives]
        NoEdit --> Threads
    end

    Threads --> Merged{PR state<br/>MERGED or CLOSED?}
    Merged -->|Yes| Done([Stop: report merged/closed - loop done])
    Merged -->|No - still OPEN| Actionable{Agent-actionable work left?<br/>failing/running checks, open LLM threads,<br/>DIRTY/UNKNOWN, unpushed, change-request}
    Actionable -->|Yes| ShortInt[Pick short interval ~240s]
    Actionable -->|No - clean/approved/<br/>waiting on human| LongInt[Pick long interval 3600-7200 - 1-2h]
    ShortInt --> SinglePass{--single-pass or<br/>no scheduler?}
    LongInt --> SinglePass
    SinglePass -->|Yes| Report([Report state + check-back time + rerun command, return])
    SinglePass -->|No| Schedule[Schedule non-blocking follow-up<br/>delaySeconds = chosen interval] --> NextPass([Return; next wakeup re-enters loop])
```

## Agent Compatibility

This skill is written for any agent that can run shell commands, inspect and edit files, use `git`/`gh`, and optionally schedule a non-blocking follow-up. Treat named tools as capability examples, not hard requirements:

- Shell commands: `Bash`, terminal, `exec_command`, or any equivalent command runner.
- File work: `Read`/`Write`/`Edit`, `apply_patch`, or any equivalent file inspection and editing tools.
- Search: `Grep`/`Glob`, `rg`, or any equivalent repository search tools.
- Follow-up scheduling: `ScheduleWakeup`, a native reminder/resume tool, a background task scheduler, or no scheduler.

The `argument-hint`, `allowed-tools`, and `required-capabilities` metadata are Claude Code compatibility hints. Other agents may ignore them. When `$ARGUMENTS` appears below, use the PR URL or PR number from the user's prompt or from the current agent host's invocation argument.

## Prerequisites

- GitHub CLI (`gh` or `gh.exe`) is installed and authenticated for the PR repository.
- The selected `gh` token can read PR reviews/checks and push to the PR branch.
- A PR URL or PR number is provided by the user or agent invocation. In Claude Code slash commands, this arrives in `$ARGUMENTS`. If it is missing, ask the user for the PR.
- Optional: `--single-pass` disables automatic follow-up scheduling. In single-pass mode, report what remains pending, when to check again, and the exact rerun prompt/command instead of scheduling the next pass.
- Optional: `--wsl` forces native WSL `git`/`gh` when running under WSL. Without it, WSL requires Windows-native `git.exe`/`gh.exe` and stops if either is missing.

Initialize the PR selector once before any use, adapting the input variable to the current agent host:

```bash
# For Claude Code slash commands, ARGUMENTS contains the prompt argument.
# For other agents, replace ARGUMENTS with the host's invocation variable or set PR directly.
ARGS="${ARGUMENTS:-}"
SINGLE_PASS=false
if printf '%s\n' "$ARGS" | grep -Eq '(^|[[:space:]])--single-pass([[:space:]]|$)'; then
  SINGLE_PASS=true
  ARGS="$(printf '%s\n' "$ARGS" | sed -E 's/(^|[[:space:]])--single-pass([[:space:]]|$)/ /; s/^[[:space:]]+//; s/[[:space:]]+$//')"
fi
USE_WSL_TOOLS=false
if printf '%s\n' "$ARGS" | grep -Eq '(^|[[:space:]])--wsl([[:space:]]|$)'; then
  USE_WSL_TOOLS=true
  ARGS="$(printf '%s\n' "$ARGS" | sed -E 's/(^|[[:space:]])--wsl([[:space:]]|$)/ /; s/^[[:space:]]+//; s/[[:space:]]+$//')"
fi
PR="$ARGS"
if [ -z "$PR" ]; then
  echo "Missing PR argument (URL or number)."
  exit 1
fi

is_wsl() {
  [ -n "${WSL_DISTRO_NAME:-}" ] || grep -qi microsoft /proc/version 2>/dev/null
}

choose_tool() {
  tool="$1"
  if is_wsl && [ "$USE_WSL_TOOLS" = false ]; then
    if command -v "${tool}.exe" >/dev/null 2>&1; then
      printf '%s.exe\n' "$tool"
      return 0
    fi
    printf 'Missing Windows-hosted tool: %s.exe\n' "$tool" >&2
    printf 'Install it on Windows or rerun with --wsl to use native WSL %s.\n' "$tool" >&2
    return 1
  fi

  if command -v "$tool" >/dev/null 2>&1; then
    printf '%s\n' "$tool"
    return 0
  fi
  printf 'Missing native tool: %s\n' "$tool" >&2
  return 1
}

GIT="$(choose_tool git)" || exit 1
GH="$(choose_tool gh)" || exit 1
clean_line() { tr -d '\r'; }
```

Use `$GIT` and `$GH` for all subsequent `git` and `gh` commands in this skill.
Strip `\r` from command substitutions that capture selected-tool output.

Check before making changes:

```bash
"$GH" auth status
"$GIT" status --short
"$GH" pr view "$PR" --json number,title,url,baseRefName,headRefName,headRepository,headRepositoryOwner,mergeStateStatus,isDraft
```

If `"$GIT" status --short` shows any output, **stop and ask the user** how to proceed before continuing. Do not commit, stash, or discard anything automatically. Present the list of changed/untracked files and offer these options:

1. **Commit all changes** — ask for a commit message, then `"$GIT" add -A && "$GIT" commit -m "<message>"`.
2. **Commit only staged changes** — if `"$GIT" diff --cached --name-only` is non-empty, ask for a commit message, then `"$GIT" commit -m "<message>"` (leaves unstaged changes untouched).
3. **Stash changes** — run `"$GIT" stash push -m "slang-pr-resolve-comments stash"` to set them aside, then proceed with the current HEAD.
4. **Abort** — stop the skill so the user can handle the changes manually.

Wait for the user's choice before continuing.

## Main Loop

Repeat this workflow periodically until the PR is merged or closed. Each pass resolves agent-actionable work (CI failures, LLM review threads, merge conflicts); once none remains the loop keeps watching at a slower cadence for later human feedback until the PR finally merges or closes (see **Completion Criteria** below). Between iterations, **do not use `sleep`** or block the live session. Use the current agent host's non-blocking follow-up facility when one exists; otherwise report the pending state and the exact prompt/command the user or orchestrator should rerun later, then return.

1. Check out the PR branch:

   ```bash
   "$GH" pr checkout "$PR"
   "$GIT" fetch --all --prune
   "$GIT" submodule update --init --recursive
   ```

   This repo uses git submodules. Run `"$GIT" submodule update --init --recursive` after any update to the local branch — e.g. `"$GH" pr checkout`, `"$GIT" pull`, or `"$GIT" rebase` (and again after resolving merge conflicts — see below) — so submodule references stay in sync with the checked-out commit. A bare fetch only updates remote-tracking refs and does not require a submodule sync on its own.

2. Inspect PR state, checks, mergeability, review-readiness notices, and review threads.
3. Check for an approval / LGTM signal (see **Approval / LGTM Signal** below). If one is present, be conservative about further edits — modify the code only with a strong justification (e.g. a human reviewer requested changes), as described in that section.
4. Fix actionable review feedback and CI failures.
5. Commit PR modifications as new commits and push them to the PR branch.
6. After pushing new commits, update the PR description if the new commits made it stale or inaccurate (see **PR Description Updates** below).
7. Reply to LLM review feedback and resolve only the LLM-owned threads that have been addressed.
8. Address every human-owned comment — make the change, give a reasoned reply, or ask a clarifying question when the intent is unclear (never guess or skip one) — then leave the thread unresolved for the human reviewer to resolve manually. Record any specific reviewer directives — from human or LLM reviewers — in the PR description (see **Recording Reviewer Directives** below) so they are not reverted on a later pass. See **Review Threads** below for details.
9. At the end of each pass, check the Completion Criteria below:
   - If the PR is **merged or closed**: report the outcome and **do not reschedule** — the loop is done.
   - Otherwise the PR is still open: schedule or request the next pass as described below — a short interval if agent-actionable work remains, a long 1–2 h interval if the PR is clean/approved and only waiting on a human — then return. The next pass re-enters this skill with the same PR argument.

Stop (do not reschedule) only if blocked by missing credentials, missing push permission, an ambiguous human decision, or local changes that cannot be safely preserved.
Draft status, WIP/DNI/DNM-style title markers, and LLM skipped-review notices are not blockers by themselves.

**Continuing the next iteration** — at the end of every pass where work remains, prefer a non-blocking follow-up.

**Choosing `<interval>`:** Pick a value that keeps the conversation context cache warm —
staying under the cache TTL avoids paying a full cold re-read on every wakeup. Use
`cache_ttl_seconds - 60` as the interval, giving a 60 s safety margin. At the current
5-minute (300 s) TTL the default is **240 s**. If you know the cache TTL has changed,
recalculate accordingly. Never use a value at or above the TTL itself.

In Claude Code, call `ScheduleWakeup`:

```text
ScheduleWakeup(
  delaySeconds = <interval>,
  prompt       = "/slang-pr-resolve-comments <PR>",
  reason       = "polling PR <PR> for new review feedback"
)
```

For other agents, use the host's native equivalent if available. If no scheduling/resume tool exists, stop after reporting:

- What is still pending.
- When to check again — use 240 s by default (see **Choosing `<interval>`** above if the cache TTL differs).
- The exact rerun prompt, for example `/slang-pr-resolve-comments <PR>` (substituting `<PR>` with the actual PR URL or number) or the equivalent invocation in the current agent. If the original run used `--single-pass`, include `--single-pass` in the rerun prompt.

## Review-Readiness Notices

Before processing normal review feedback, check whether PR metadata may explain why LLM reviewers intentionally skipped or delayed review:

```bash
"$GH" pr view "$PR" --json title,isDraft,url
```

Do not treat draft status, title markers such as `WIP`, `DNI`, `DNM`, `do not review`, `do not merge`, or LLM skipped-review comments as blockers for this skill. They are context only: continue processing CI failures, merge conflicts, and any actionable review threads that already exist.

If an LLM left a review-readiness notice:

1. Notify the user with the PR URL, the LLM comment URL, and the exact readiness reason.
2. Do not change the draft state or title unless the user explicitly asks.
3. Do not treat the message as code feedback, and do not mark the thread resolved on behalf of the user.
4. Continue with the normal workflow. Do not stop, reschedule, or withhold success solely because the PR remains draft, the title contains WIP/DNI/DNM-style wording, or an LLM left a skipped-review notice.

## Approval / LGTM Signal

Once the PR has been approved, **be conservative about modifying the code.** An approval means the reviewers are satisfied with the current state, and further automated edits could invalidate that approval or introduce unwanted changes. This is a strong bias against editing, not an absolute stop — make changes only when there is a clear justification (see below).

Detect an approval signal in two ways:

1. **Formal GitHub approval** — the PR's review decision or a reviewer's review state is `APPROVED`:

   ```bash
   "$GH" pr view "$PR" --json reviewDecision --jq .reviewDecision
   "$GH" pr view "$PR" --json reviews --jq '.reviews | group_by(.author.login?) | map(last)[] | {author: .author.login?, state: .state}'
   ```

   Treat the PR as approved when `reviewDecision` is `APPROVED`, or when any reviewer's latest review `state` is `APPROVED`.

2. **Informal approval phrases** — a review comment, review body, or review thread from a human reviewer that clearly signals approval, such as `LGTM`, `Looks good to me`, `Approved`, `Ship it`, or an equivalent. Match case-insensitively. Only count these from human reviewers (per the **Review Threads** classification) — ignore such phrases echoed by the PR author, bots, or LLM reviewers.

When an approval signal is present, default to **not** making further code changes:

1. Do not push new commits or rebase unless GitHub reports merge conflicts that block merge.
2. Finish any already-pushed, in-flight work (e.g., wait for running CI on commits you already pushed), but do not start speculative or stylistic edits.
3. Reply to and resolve any remaining addressed LLM threads as usual, then report that the PR is approved.
4. **Keep watching, but slowly.** An approval is not the end — later human review feedback can still arrive and may request changes. Do not stop the loop on approval. Instead, reschedule the next pass with a **long interval of 1–2 hours** (`delaySeconds` of `3600`–`7200`, clamped to the host's maximum) instead of the normal short polling interval. This long-poll mode trades a cold context cache for far fewer wakeups while waiting on humans. Continue this slow watch until the PR is merged/closed, the user tells you to stop, or new feedback arrives that pulls you back into the normal workflow. If `--single-pass` was requested or no scheduler exists, report the approved state, when to check again (1–2 hours), and the exact rerun command instead of scheduling.

**Strong justifications that override the approval bias** — modify the code even when the PR is approved/LGTM when any of these apply:

- **A human reviewer has requested changes.** A human change-request always wins over an approval/LGTM signal: the requested changes must be made. This holds whether the change-request is a formal `CHANGES_REQUESTED` review (`reviewDecision` is `CHANGES_REQUESTED`, or a reviewer's latest review `state` is `CHANGES_REQUESTED`) or a clear human request in a comment or thread. When a newer human review or comment supersedes an earlier approval, follow the newer one.
- The user explicitly asks for a change.
- A required CI check is failing and the fix is necessary for the PR to merge — make the minimal fix and note that it may dismiss the approval.

If none of these apply, leave the approved code as-is rather than making discretionary edits. When in doubt about whether an edit is justified on an approved PR, prefer to report the situation to the user rather than editing.

## Commit Policy

When the PR is modified for any reason, preserve the change history by creating a new commit for the modification. Do not use `"$GIT" commit --amend` for review fixes, CI fixes, conflict-resolution follow-up edits, formatting changes, or any other PR update.

Use concise commit messages that describe the reason for the follow-up change, for example:

```bash
"$GIT" add <changed-files>
"$GIT" commit -m "Address review feedback"
"$GIT" push
```

## PR Description Updates

After pushing any new commit to the PR branch, check whether the PR description still accurately reflects what the PR does. Update it when the new commits change the scope, behavior, or rationale in a way that makes the existing description stale, misleading, or incomplete. Examples of changes that warrant a description update:

- A review fix changes user-visible behavior, the public API, or configuration that the description documents.
- New commits add, remove, or rename functionality beyond what the description mentions.
- The description references a test plan, follow-up TODOs, or known limitations that are no longer accurate after the new commits.
- Conflict resolution or rebase work materially changes what is included in the PR.

Skip the update when the new commits are purely cosmetic (formatting, typo fixes, comment tweaks) or when they only address narrow review feedback that does not change the summary-level meaning of the PR.

Fetch the current description, edit it, and push the update with `$GH`:

```bash
"$GH" pr view "$PR" --json body --jq .body > /tmp/pr-body.md
# Edit /tmp/pr-body.md to reflect the current state of the PR.
"$GH" pr edit "$PR" --body-file /tmp/pr-body.md
```

Preserve any existing sections (summary, test plan, generated-by footers, issue links, the **Reviewer Directives** section) unless they are now inaccurate. Do not rewrite the description from scratch when a targeted edit will do. Never drop a recorded reviewer directive during a routine description update — see **Recording Reviewer Directives** below.

## Review Threads

Use GitHub GraphQL to list review threads, because `$GH pr view` does not expose all thread resolution state:

```bash
PR_NUMBER="$("$GH" pr view "$PR" --json number --jq .number | clean_line)"
OWNER="$("$GH" pr view "$PR" --json baseRepository --jq .baseRepository.owner.login | clean_line)"
REPO="$("$GH" pr view "$PR" --json baseRepository --jq .baseRepository.name | clean_line)"

"$GH" api graphql -F owner="$OWNER" -F repo="$REPO" -F pr="$PR_NUMBER" -f query='
query($owner:String!, $repo:String!, $pr:Int!, $after:String) {
  repository(owner:$owner, name:$repo) {
    pullRequest(number:$pr) {
      reviewThreads(first:100, after:$after) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          startLine
          comments(last:100) {
            nodes {
              id
              url
              body
              author { login __typename }
              createdAt
            }
          }
        }
      }
    }
  }
}'
```

Classify threads conservatively:

Check these in order — the first matching rule wins:

- **`bmillsNV`**: this account exists only to absorb review-request email spam and is not an actual reviewer. Ignore any review requests, assignments, or threads attributed to `bmillsNV` — do not treat them as human or LLM feedback, do not reply, and do not block completion on them. Checked first so this holds even if the account is ever a service/bot account.
- **LLM review feedback**: the author's `__typename` is `Bot` (from the GraphQL response), or the author is clearly an automated LLM reviewer by login — such as Copilot, CodeRabbit, Claude, Codex, OpenAI, Gemini, Greptile or another bot whose comment identifies itself as AI review feedback.
- **Human feedback**: the author is a person, the `author` field is `null` (deleted account — treat as human to be safe), or the source is ambiguous.
- **CI/static-analysis bot output**: handle it as CI feedback unless it is clearly an LLM review thread.

For each unresolved, non-outdated (`isResolved = false` and `isOutdated = false`) LLM thread:

1. Read the full thread and relevant code.
2. Apply the fix, or determine that the suggestion is invalid with evidence.
3. Run focused validation. For Slang compiler/test invocations, use the
   `slang-run-tests` binary selection rule: under WSL with a Windows-hosted
   build, require `slangc.exe` and `slang-test.exe` and do not fall back to
   WSL-native binaries.
4. Push the fix if code changed.
5. Reply on the thread with what changed, what validation ran, or why no code change was needed. **Always start the reply body with `[Agent] `** so readers can distinguish agent-posted comments from comments left by the human account owner.
6. Resolve the thread only after the reply is posted and the issue is actually addressed.
7. If the LLM's suggestion established a durable decision worth protecting from later reversal (e.g. "intentionally don't do X here"), record it in the PR description per **Recording Reviewer Directives** below, tagged as an LLM-sourced directive.

Reply to an LLM thread:

```bash
"$GH" api graphql -F thread="$THREAD_ID" -F body="$REPLY_BODY" -f query='
mutation($thread:ID!, $body:String!) {
  addPullRequestReviewThreadReply(input:{pullRequestReviewThreadId:$thread, body:$body}) {
    comment { url }
  }
}'
```

Resolve an addressed LLM thread:

```bash
"$GH" api graphql -F thread="$THREAD_ID" -f query='
mutation($thread:ID!) {
  resolveReviewThread(input:{threadId:$thread}) {
    thread { id isResolved }
  }
}'
```

**Every human review comment must be addressed — never skip or silently ignore one.** "Addressed" means you either made the requested change, or replied with a concrete reason why no change was made (e.g. evidence the suggestion is already satisfied or incorrect), or asked a clarifying question (see below). Leaving a human comment with no agent action and no reply is not acceptable.

**If the requested action or the reviewer's intent is unclear, do not guess.** Reply on the thread with a specific clarifying question — state your current understanding, the options you see, and exactly what you need the reviewer to confirm — and wait for the human's answer before making a change that could be wrong. An unanswered clarifying question keeps the thread actionable, so the loop keeps watching (it is not a blocker that stops the loop; it just means the work is not yet complete).

**Do not mark human threads resolved yourself.** Resolution is the human reviewer's decision: after you address the comment, reply with a concise summary and ask the reviewer to resolve the thread if satisfied. **Always start the reply body with `[Agent] `** so readers can distinguish agent-posted comments from comments left by the human account owner. If a human thread contains a specific directive about what the PR should or should not contain (e.g. "don't add tests for this"), also record it in the PR description per **Recording Reviewer Directives** below so it is not silently reverted later.

If `pageInfo.hasNextPage` is true, paginate and inspect every review thread before deciding that the PR has no remaining feedback.
For pagination, repeat the query adding `-F after="$END_CURSOR"` (using the value from `pageInfo.endCursor`) to the `$GH api graphql` command, with `reviewThreads(first:100, after:$after)` in the query.

## Recording Reviewer Directives

When a reviewer — **human or LLM** — makes a specific directive about how this PR should (or should not) be implemented, and you have adopted it, record it as a durable note in the **PR description**, not just in the thread reply. Directives are constraints such as "do not add tests for this change", "keep this function as-is", "don't refactor X", or "leave Y out of scope". A good LLM suggestion that establishes a durable decision belongs here too — the goal is to capture standing decisions regardless of who proposed them.

This is required because review threads are easy for the next reviewer to miss. A directive that lives only in a comment can be silently undone later: for example, a reviewer asks not to add test cases for a change and the agent removes them, but on a later pass another reviewer flags the now-missing lines as a coverage gap, and the agent re-adds the very tests that were rejected. Persisting the directive in the description keeps it visible to every reviewer — including LLM reviewers that only read the current description and diff and do not replay the full comment history — so the same unwanted change does not keep coming back.

Maintain a dedicated section in the PR description titled **`## Reviewer Directives (maintained by agent)`**. Use this exact heading so the section is found and updated consistently across passes. Note the source on each entry, since human directives take precedence (see below):

```markdown
## Reviewer Directives (maintained by agent)

- [human @reviewer-login] Do not add test cases for the `<feature>` change — intentionally untested per review. (<comment URL>)
- [LLM CodeRabbit] Keep `<function>` unchanged; refactor is out of scope for this PR. (<comment URL>)
```

When recording and honoring directives:

1. **Add an entry** when any reviewer states a specific, durable constraint on the PR's contents that you adopt. Capture the source (human login, or the LLM reviewer's name), the directive in your own concise words, and a link to the originating comment. Apply the directive to the code in the same pass.
2. **Record from human or LLM reviewers.** Human reviewers' directives are recorded as stated. Record an LLM reviewer's suggestion as a directive once you adopt it as a standing decision (don't pre-record every LLM comment — only durable constraints worth protecting). Tag each entry with its source. (Per the **Review Threads** classification, treat `author: null` as human.)
3. **Treat recorded directives as standing constraints on every later pass.** Before making any change suggested by another reviewer or CI, check it against the recorded directives. **Do not undo a recorded directive to satisfy a later conflicting suggestion or a coverage check.**
4. **When a thread conflicts with a recorded directive**, do not make the change. Reply to that thread (starting with `[Agent] `) explaining that the behavior is intentional per the recorded directive, link to the description section, and resolve the thread as addressed.
5. **Human directives outrank LLM directives.** If a human directive conflicts with a recorded LLM directive, follow the human: update or remove the LLM entry, note the change, and apply the human's instruction. Never override or silently drop a human directive to satisfy an LLM one.
6. **Keep the section accurate.** Remove or update an entry only when a reviewer of equal-or-higher precedence (or the user) explicitly lifts or changes the directive; note who lifted it. Do not silently drop directives.
7. Edit the description using the same fetch/edit/push flow in **PR Description Updates** above, preserving all other sections.

## CI Failures

Inspect checks with:

```bash
"$GH" pr checks "$PR"
"$GH" run list --branch "$("$GIT" branch --show-current | clean_line)" --limit 10
RUN_ID="$("$GH" run list --branch "$("$GIT" branch --show-current | clean_line)" --status failure --limit 1 --json databaseId --jq '.[0].databaseId' | clean_line)"
if [ -n "$RUN_ID" ] && [ "$RUN_ID" != "null" ]; then
  "$GH" run view "$RUN_ID" --log-failed
else
  echo "No failed workflow runs found for current branch."
fi
```

For each failure:

1. Identify the failing job and command from the logs.
2. **Determine if the failure looks intermittent or infra-related** (see below). If so, retry instead of attempting a code fix.
3. Otherwise, reproduce locally when feasible.
4. Fix the code or test.
5. Run the narrowest reliable validation first, then broader validation when the change warrants it.
6. Push to the PR branch.
7. Continue monitoring until the new checks finish.

### Intermittent / Infra Failures

Treat a failure as intermittent or infrastructure-related when the logs show any of:

- Network errors: timeouts, connection resets, DNS failures, `curl`/`wget` failures fetching dependencies or artifacts
- Resource exhaustion: out-of-memory kills, disk-full errors, CPU throttling, runner eviction
- Runner/infra issues: runner setup failures, Docker pull failures, missing environment variables injected by CI, agent disconnects
- Flaky test output: assertions about timing, port conflicts, race conditions with no code change that could explain it
- Lock or concurrency errors in the CI infrastructure itself (e.g., package-manager lock conflicts unrelated to code changes)
- Errors in unrelated jobs (e.g., a deploy job fails while the compile job that touches your code succeeds)

When a failure matches any of these, **do not attempt a code fix**. Instead, retry the failed run:

```bash
"$GH" run rerun "$RUN_ID" --failed
```

The `--failed` flag re-runs only the failed jobs, not the entire workflow.

**Waiting for the retry option to become available:** GitHub only allows rerunning once the run `status` is `completed` (the only terminal status value; `queued` and `in_progress` are non-terminal). The run outcome — `success`, `failure`, `cancelled`, etc. — is stored in the separate `conclusion` field, which is only populated after `status` becomes `completed`. The script below correctly gates on `.status == "completed"` before attempting the rerun. If the run is still in progress when you first inspect it, schedule a wakeup and try again:

```bash
RUN_STATUS="$("$GH" run view "$RUN_ID" --json status --jq .status | clean_line)"
if [ "$RUN_STATUS" != "completed" ]; then
  echo "Run $RUN_ID is still $RUN_STATUS — will retry rerun after next wakeup."
else
  "$GH" run rerun "$RUN_ID" --failed
fi
```

After issuing a rerun, proceed to the **Completion Criteria** section to schedule or request the next pass, then verify in the following pass whether the retried run passed. If the same job fails again with the same infra-looking error, retry once more (up to **3 total attempts** for the same run). After 3 consecutive infra-looking failures, stop retrying and report the pattern to the user — the infra issue may be persistent and require human intervention.

If checks are still running and there is no review work to do, do not block — use a non-blocking check, then proceed to the **Completion Criteria** section to schedule or request the next pass and return:

```bash
"$GH" pr checks "$PR"
```

## Merge Conflicts And Auto-Rebase Failures

**Do not rebase proactively.** Only rebase onto the base branch when GitHub explicitly reports merge conflicts (i.e. `mergeStateStatus` is `DIRTY` or the PR shows a conflict that blocks merge). If a push is rejected because the remote PR branch has newer commits, synchronize with the remote first (e.g. `"$GIT" pull --rebase`). If the branch is merely behind the base branch but has no conflicts and you have no local changes to push, leave it alone — GitHub's auto-merge will rebase or merge it when the time comes.

If GitHub reports that auto-merge or auto-rebase cannot continue because conflicts must be resolved, update the PR branch manually.

Inspect merge state:

```bash
"$GH" pr view "$PR" --json baseRefName,headRefName,mergeStateStatus,headRepository,headRepositoryOwner
```

Resolve by rebasing onto the latest base branch:

```bash
BASE="$("$GH" pr view "$PR" --json baseRefName --jq .baseRefName | clean_line)"
HEAD_BRANCH="$("$GH" pr view "$PR" --json headRefName --jq .headRefName | clean_line)"
BASE_REPO="$("$GH" pr view "$PR" --json baseRepository --jq .baseRepository.nameWithOwner | clean_line)"
BASE_REPO_ESC="$(printf '%s' "$BASE_REPO" | sed -e 's/[][(){}.^$*+?|\\]/\\&/g')"
BASE_REMOTE="$("$GIT" remote -v | grep -Em1 "github\.com[:/]${BASE_REPO_ESC}(\.git)?([[:space:]]|$)" | awk '{print $1}' | clean_line)"
if [ -z "$BASE_REMOTE" ]; then
  BASE_REMOTE="upstream"
  if ! "$GIT" remote get-url "$BASE_REMOTE" 2>/dev/null; then
    echo "Could not determine base remote for $BASE_REPO and 'upstream' does not exist"
    exit 1
  fi
fi
"$GIT" fetch "$BASE_REMOTE" "$BASE"
"$GIT" rebase "$BASE_REMOTE/$BASE"
```

Resolve conflicts in the files, then continue:

```bash
"$GIT" add <resolved-files>
"$GIT" rebase --continue
"$GIT" submodule update --init --recursive
```

Always re-run `"$GIT" submodule update --init --recursive` after a rebase or conflict resolution so submodule pointers match the new HEAD.

Run relevant validation, then push with a lease:

```bash
HEAD_REPO="$("$GH" pr view "$PR" --json headRepository --jq .headRepository.nameWithOwner | clean_line)"
HEAD_REPO_ESC="$(printf '%s' "$HEAD_REPO" | sed -e 's/[][(){}.^$*+?|\\]/\\&/g')"
PUSH_REMOTE="$("$GIT" remote -v | grep -Em1 "github\.com[:/]${HEAD_REPO_ESC}(\.git)?([[:space:]]|$)" | awk '{print $1}' | clean_line)"
if [ -z "$PUSH_REMOTE" ]; then
  "$GH" pr checkout "$PR"
  PUSH_REMOTE="$("$GIT" remote -v | grep -Em1 "github\.com[:/]${HEAD_REPO_ESC}(\.git)?([[:space:]]|$)" | awk '{print $1}' | clean_line)"
fi
if [ -z "$PUSH_REMOTE" ]; then
  echo "Could not determine push remote for $HEAD_REPO"
  exit 1
fi
"$GIT" push --force-with-lease "$PUSH_REMOTE" "HEAD:$HEAD_BRANCH"
```

## Completion Criteria

**The loop terminates only when the PR is merged or closed.** Everything else is a question of how soon to run the next pass, not whether to stop. As long as the PR is open, there may still be work to do or later human feedback to catch, so the agent keeps watching.

Check the terminal condition every pass:

```bash
"$GH" pr view "$PR" --json state --jq .state
```

**Stop and report — do not schedule another pass** when `state` is `MERGED` or `CLOSED`:

- `MERGED`: report that the PR landed; the loop is done.
- `CLOSED` (not merged): report that the PR was closed without merging and stop. Do not reopen it or push further changes.

**Otherwise the PR is still `OPEN` — keep watching.** Choose the next interval by how much actionable work remains:

- **Short interval (~240s, see "Choosing `<interval>`" above)** when there is agent-actionable work pending: required checks failing or still running, unresolved non-outdated LLM review threads, **any human review comment not yet addressed** (no change made, no reply, or you owe an answer to your own clarifying question), `mergeStateStatus` is `DIRTY` (merge conflicts) or `UNKNOWN` (still calculating), unpushed local commits, or a human/user change-request to address. Every human comment must be addressed before the PR can be considered clean.
- **Long interval (1–2 hours, `delaySeconds` of `3600`–`7200`, clamped to the host's maximum)** when there is no agent-actionable work left and the PR is just waiting — e.g. it is approved/LGTM, all required checks pass, no open LLM threads, and it is only waiting on a human merge or further human review. This slow-watch mode catches late human feedback without burning wakeups.

In either case, schedule a non-blocking follow-up when the agent host supports one, then return. The next pass re-enters this skill with the same PR argument. If a single-pass run was requested (`--single-pass` or `SINGLE_PASS=true`) or scheduling is unavailable, report the current state, when to check again (short vs. long interval), and the exact rerun prompt/command instead of scheduling, then return.

**The following are not terminal and do not stop the loop** — they only mean the slow-watch (long) interval applies if nothing else is actionable:

1. **Human review threads the agent has already addressed but that remain unresolved**: marking them resolved is the human's decision, not the agent's. Report "PR is ready — waiting for human reviewers to resolve N thread(s)" and keep slow-watching until the PR merges or closes. This applies only once every human comment has been addressed (change made, reasoned reply, or a clarifying question posted) — an unaddressed human comment is actionable work, not a "waiting on human" state.
2. **Approved / LGTM**: report it, make no discretionary changes, and slow-watch for later human feedback (see **Approval / LGTM Signal** above). Approval is not merge — only `MERGED`/`CLOSED` ends the loop.
3. **Draft/WIP/DNI/DNM status and readiness notices**: report them as context, but do not stop or treat them as blockers.
