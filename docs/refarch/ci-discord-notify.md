# Reference Architecture: CI-to-Discord Notifications

> **This doc is meant to be used with Kiro or any coding CLI.** Prompt your AI agent with something like:
>
> ```
> per https://github.com/openabdev/openab/blob/main/docs/refarch/ci-discord-notify.md set up CI-to-Discord notifications for my repo
> ```
>
> and it will guide you through (or handle) the full setup.

Send GitHub Actions CI results (pass/fail) to a Discord channel thread via webhook, with clickable links and dynamic thread routing extracted from PR descriptions.

## Architecture

```
+-- GitHub Actions ----------------------------------------+
|                                                          |
|  +-- CI Workflow (ci.yml) ----------------------------+  |
|  |                                                    |  |
|  |  [checkout] -> [build/test] -> [collect metadata]  |  |
|  |                                    |               |  |
|  |                                    v               |  |
|  +----------------------------------------------------+  |
|                                       |                  |
|  +-- Notify Workflow (notify-discord.yml) ------------+  |
|  |  (reusable, workflow_call)                         |  |
|  |                                                    |  |
|  |  inputs: status, failed_step, duration,            |  |
|  |          commit_msg, commit_author, pr_body        |  |
|  |                                                    |  |
|  |  1. Extract Thread ID from PR body                 |  |
|  |     (regex: ^Thread:\s*<id>)                       |  |
|  |  2. Fallback to env var DISCORD_THREAD_ID          |  |
|  |  3. Build Discord embed (green/red sidebar)        |  |
|  |  4. POST to webhook ?thread_id=<id>               |  |
|  +----------------------------------------------------+  |
|           |                                              |
+-----------+----------------------------------------------+
            |
            v  (HTTPS POST)
+-- Discord -----------------------------------+
|                                              |
|  Webhook URL ──► Channel / Thread            |
|                                              |
|  ┌─────────────────────────────────────────┐ |
|  │ ✅ CI success — 7iac/chiac@main         │ |
|  │ 👤 pahud — feat(x): add feature (#42)   │ |
|  │ ⏱️ 2m 15s                               │ |
|  │ @超渡法師(KIRO)                          │ |
|  └─────────────────────────────────────────┘ |
|                                              |
+----------------------------------------------+
```

## How It Works

1. **CI workflow** runs build/test steps and collects metadata (status, duration, failed step, commit info) as job outputs.
2. **Notify workflow** is called as a reusable `workflow_call` workflow, receiving metadata as inputs.
3. **Thread routing**: The notify workflow extracts a Discord thread ID from the PR body (`Thread: <id>`), falling back to a repository-level environment variable.
4. **Embed format**: Uses Discord embed (not plain `content`) so markdown links are clickable. Green sidebar for success, red for failure.
5. **Mention**: Optionally pings a Discord user/bot via `DISCORD_MENTION_USER_ID` env var.

## Prerequisites

- A Discord server with a channel for CI notifications
- A Discord webhook URL (Channel Settings → Integrations → Webhooks)
- A GitHub repository with Actions enabled
- A GitHub Actions environment named `discord-notify` with the following configured:
  - **Secret**: `DISCORD_WEBHOOK_URL` — the webhook URL
  - **Variable**: `DISCORD_THREAD_ID` — default thread ID (fallback)
  - **Variable**: `DISCORD_MENTION_USER_ID` — Discord user ID to ping (optional)

## Setup Steps

### Step 1: Create the Discord Webhook

1. In your Discord server, go to the target channel → Edit Channel → Integrations → Webhooks.
2. Create a new webhook, name it (e.g., "GH CI Webhook").
3. Copy the webhook URL.

### Step 2: Configure GitHub Environment

1. In your repo, go to Settings → Environments → New environment → name it `discord-notify`.
2. Add secret `DISCORD_WEBHOOK_URL` with the webhook URL.
3. Add variable `DISCORD_THREAD_ID` with your default Discord thread ID.
4. (Optional) Add variable `DISCORD_MENTION_USER_ID` with the Discord user ID to mention.

### Step 3: Create the Reusable Notify Workflow

Create `.github/workflows/notify-discord.yml`:

```yaml
name: Discord Notify

on:
  workflow_call:
    inputs:
      status:
        required: true
        type: string
      failed_step:
        required: false
        type: string
      duration:
        required: false
        type: string
      commit_msg:
        required: false
        type: string
      commit_author:
        required: false
        type: string
      pr_body:
        required: false
        type: string
    secrets:
      DISCORD_WEBHOOK_URL:
        required: true

jobs:
  notify:
    runs-on: ubuntu-latest
    environment: discord-notify
    steps:
      - name: Send Discord notification
        env:
          WEBHOOK_URL: ${{ secrets.DISCORD_WEBHOOK_URL }}
          DEFAULT_THREAD_ID: ${{ vars.DISCORD_THREAD_ID }}
          MENTION_USER_ID: ${{ vars.DISCORD_MENTION_USER_ID }}
          STATUS: ${{ inputs.status }}
          FAILED_STEP: ${{ inputs.failed_step }}
          DURATION: ${{ inputs.duration }}
          COMMIT_MSG: ${{ inputs.commit_msg }}
          COMMIT_AUTHOR: ${{ inputs.commit_author }}
          PR_BODY: ${{ inputs.pr_body }}
          RUN_URL: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          REPO: ${{ github.repository }}
          REF: ${{ github.ref_name }}
          PR: ${{ github.event.pull_request.number }}
          SERVER_URL: ${{ github.server_url }}
        run: |
          # Extract Thread ID from PR body, fallback to variable
          THREAD_ID=""
          if [ -n "$PR_BODY" ]; then
            THREAD_ID=$(echo "$PR_BODY" | grep -ioP '^Thread:\s*\K[0-9]+' | head -1)
          fi
          [ -z "$THREAD_ID" ] && THREAD_ID="$DEFAULT_THREAD_ID"

          if [ "$STATUS" = "success" ]; then
            COLOR=3066993
            EMOJI="✅"
          else
            COLOR=15158332
            EMOJI="❌"
          fi

          # Build description with clickable links
          DESC="${EMOJI} **CI ${STATUS}** — [\`${REPO}@${REF}\`](${RUN_URL})"
          if [ -n "$PR" ]; then
            DESC="${DESC} | [PR #${PR}](${SERVER_URL}/${REPO}/pull/${PR})"
          fi
          if [ -n "$COMMIT_AUTHOR" ]; then
            DESC="${DESC}\n👤 ${COMMIT_AUTHOR}"
            if [ -n "$COMMIT_MSG" ] && [ -n "$PR" ]; then
              DESC="${DESC} — [${COMMIT_MSG}](${SERVER_URL}/${REPO}/pull/${PR})"
            elif [ -n "$COMMIT_MSG" ]; then
              DESC="${DESC} — \`${COMMIT_MSG}\`"
            fi
          fi
          [ -n "$DURATION" ] && DESC="${DESC}\n⏱️ ${DURATION}"
          [ "$STATUS" != "success" ] && [ -n "$FAILED_STEP" ] && \
            DESC="${DESC}\n💥 Failed at: **${FAILED_STEP}**"

          # Build JSON payload with embed
          CONTENT=""
          [ -n "$MENTION_USER_ID" ] && CONTENT="<@${MENTION_USER_ID}>"

          PAYLOAD=$(jq -n \
            --arg content "$CONTENT" \
            --arg desc "$DESC" \
            --argjson color "$COLOR" \
            '{content: $content, embeds: [{description: $desc, color: $color}]}')

          URL="${WEBHOOK_URL}"
          [ -n "$THREAD_ID" ] && URL="${URL}?thread_id=${THREAD_ID}"

          curl -sf -X POST "$URL" \
            -H "Content-Type: application/json" \
            -d "$PAYLOAD"
```

### Step 4: Call from Your CI Workflow

Add a notify job at the end of your CI workflow:

```yaml
jobs:
  check:
    runs-on: ubuntu-latest
    outputs:
      failed_step: ${{ steps.report.outputs.failed_step }}
      duration: ${{ steps.report.outputs.duration }}
      commit_msg: ${{ steps.report.outputs.commit_msg }}
      commit_author: ${{ steps.report.outputs.commit_author }}
    steps:
      # ... your build/test steps ...

      - name: Collect CI metadata
        id: report
        if: always()
        run: |
          echo "commit_msg=$(git log -1 --pretty=%s)" >> "$GITHUB_OUTPUT"
          echo "commit_author=$(git log -1 --pretty=%an)" >> "$GITHUB_OUTPUT"
          # Add duration/failed_step logic as needed

  notify:
    needs: [check]
    if: always()
    uses: ./.github/workflows/notify-discord.yml
    with:
      status: ${{ needs.check.result }}
      failed_step: ${{ needs.check.outputs.failed_step }}
      duration: ${{ needs.check.outputs.duration }}
      commit_msg: ${{ needs.check.outputs.commit_msg }}
      commit_author: ${{ needs.check.outputs.commit_author }}
      pr_body: ${{ github.event.pull_request.body }}
    secrets:
      DISCORD_WEBHOOK_URL: ${{ secrets.DISCORD_WEBHOOK_URL }}
```

## Dynamic Thread Routing

To route notifications to a specific Discord thread per PR, add this line anywhere in the PR description:

```
Thread: 1505664791719710810
```

The notify workflow extracts this value and appends `?thread_id=<id>` to the webhook URL. This lets different PRs notify different threads (e.g., per-feature discussion threads).

If no `Thread:` line is found, it falls back to the `DISCORD_THREAD_ID` environment variable.

## Customization

| What | How |
|------|-----|
| Change colors | Edit `COLOR` values (Discord decimal color codes) |
| Add more metadata | Add inputs to the reusable workflow, pass from CI |
| Multiple channels | Create additional webhooks, use conditional logic |
| Suppress mentions | Remove `DISCORD_MENTION_USER_ID` variable |
| Thread per branch | Maintain a mapping of branch → thread ID |

## Reference Implementation

See [7iac/chiac](https://github.com/7iac/chiac) for a working example using this pattern with a Rust CI pipeline and self-hosted runners.
