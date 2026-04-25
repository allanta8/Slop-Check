# Slop Check Twitter Bot — Deployment Guide

Deployment runbook for the bot described in `BOT_HANDOFF.md`. Bot lives in the `viewft` repo (suggested path: `apps/slop-bot/`). Runtime is GitHub Actions cron — no server.

---

## 1. Twitter app setup (one-time, manual)

1. Go to https://developer.twitter.com → create a Project + App
2. Subscribe to **Basic tier** ($200/mo) — Free tier blocks `/2/users/:id/mentions`
3. In the app's User Authentication settings:
   - **App permissions:** Read and write
   - **Type of app:** Web App, Automated App or Bot
   - **Callback URL:** any placeholder (not used)
4. Generate and save:
   - API Key (`TWITTER_APP_KEY`)
   - API Key Secret (`TWITTER_APP_SECRET`)
   - Access Token (`TWITTER_ACCESS_TOKEN`) — must be generated **after** setting Read+Write
   - Access Token Secret (`TWITTER_ACCESS_SECRET`)
5. Get the bot's numeric user ID:
   ```bash
   curl "https://api.twitter.com/2/users/by/username/<bot_handle>" \
     -H "Authorization: Bearer <bearer_token>"
   ```
   Save as `TWITTER_BOT_USER_ID`.

**Gotcha:** If you generate Access Tokens before flipping to Read+Write, the tokens stay read-only. Regenerate after the permission change.

---

## 2. Anthropic API key

1. https://console.anthropic.com → API Keys → Create
2. Save as `ANTHROPIC_API_KEY`
3. Set a monthly spend limit on the account (suggest $20/mo for v1 — sonnet calls are cheap, but cap abuse)

---

## 3. viewft API credentials

Depends on the contract you define. Expected:
- `VIEWFT_API_URL` — endpoint that accepts `POST { tweet_id, html }` and returns `{ url }`
- `VIEWFT_API_KEY` — bearer token for the bot service account

If the API doesn't exist yet, stub `publish.ts` to write to a local `docs/r/<id>.html` and return a placeholder URL — ship the bot, wire viewft later.

---

## 4. GitHub repo secrets

In the `viewft` repo: Settings → Secrets and variables → Actions → New repository secret

Add all eight:
- `TWITTER_APP_KEY`
- `TWITTER_APP_SECRET`
- `TWITTER_ACCESS_TOKEN`
- `TWITTER_ACCESS_SECRET`
- `TWITTER_BOT_USER_ID`
- `ANTHROPIC_API_KEY`
- `VIEWFT_API_URL`
- `VIEWFT_API_KEY`

---

## 5. GitHub Actions workflow

`.github/workflows/slop-bot.yml`:

```yaml
name: slop-bot

on:
  schedule:
    - cron: '*/5 * * * *'
  workflow_dispatch:

concurrency:
  group: slop-bot
  cancel-in-progress: false

jobs:
  poll:
    runs-on: ubuntu-latest
    permissions:
      contents: write    # for committing state.json
    defaults:
      run:
        working-directory: apps/slop-bot
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
          cache-dependency-path: apps/slop-bot/package-lock.json
      - run: npm ci
      - run: npm run start
        env:
          TWITTER_APP_KEY: ${{ secrets.TWITTER_APP_KEY }}
          TWITTER_APP_SECRET: ${{ secrets.TWITTER_APP_SECRET }}
          TWITTER_ACCESS_TOKEN: ${{ secrets.TWITTER_ACCESS_TOKEN }}
          TWITTER_ACCESS_SECRET: ${{ secrets.TWITTER_ACCESS_SECRET }}
          TWITTER_BOT_USER_ID: ${{ secrets.TWITTER_BOT_USER_ID }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          VIEWFT_API_URL: ${{ secrets.VIEWFT_API_URL }}
          VIEWFT_API_KEY: ${{ secrets.VIEWFT_API_KEY }}
      - name: Commit state
        run: |
          git config user.name 'slop-bot'
          git config user.email 'slop-bot@users.noreply.github.com'
          git add state.json
          git diff --staged --quiet || git commit -m "bot: update since_id [skip ci]"
          git push
```

**Notes:**
- `concurrency` prevents overlapping runs if one is slow.
- `[skip ci]` in the state commit message prevents recursive triggers.
- 5-min cron is best-effort — GitHub may delay by a few minutes under load. Acceptable for a mention bot.
- Cost: ~8,640 minutes/month on free tier. Well within the 2,000 min limit if each run is <15s. **Verify run duration after first day** — if runs creep over 15s, switch to a long-running host.

---

## 6. Local dev

```bash
cd apps/slop-bot
cp .env.example .env
# fill in secrets
npm install
npm run dev
```

`npm run dev` should be the same one-shot script as production. To replay a specific mention without waiting for cron, support a CLI flag:
```bash
npm run dev -- --tweet-id 1893456789012345678
```

---

## 7. First-run smoke test

1. Manually trigger the workflow: Actions → slop-bot → Run workflow
2. From a test account, reply to any tweet tagging the bot
3. Within 5 min, the bot should reply with a slop card
4. Check `state.json` got committed back to the repo
5. Click the report link — confirm it loads

**If it doesn't reply:**
- Action logs first — Anthropic 4xx, Twitter 401 (perms), `since_id` already past the mention?
- Reset `state.json` to `{ "since_id": null }` and re-run if the test mention was missed

---

## 8. Monitoring

v1: GitHub Actions UI. Failed runs are emailed to repo watchers by default.

v2 (if needed):
- Add a Slack webhook on failure (one extra step in the workflow)
- Log structured JSON to stdout, ship to Logtail / Axiom if volume justifies it

---

## 9. Rolling back

The bot is stateless beyond `state.json`. To pause:
1. Disable the workflow: Actions → slop-bot → ⋯ → Disable workflow
2. Or revoke the Twitter access token

To roll back code: revert the commit, re-run. `since_id` carries forward, so no duplicate replies.

---

## 10. When to graduate off Actions

Move to Fly.io / Railway / a VPS when any of:
- Run duration exceeds 30s consistently (Action minutes get expensive)
- Need <1 min mention latency
- Want webhooks instead of polling (Twitter Account Activity API — more setup, faster)
- Multiple bots / shared state across them

For v1, Actions is fine.
