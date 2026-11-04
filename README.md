# job-radar

I built this to run my own job search. Every day it pulls remote AI and engineering postings from public job boards, has Claude score each one against a written profile of me, and shows only the ones worth applying to. On my machine it runs on a schedule and posts the matches to a Discord channel, so I read a short list instead of scrolling boards.

## Real output

A dry run on 2026-11-04, scored by Claude against [example/profile.md](example/profile.md), links trimmed:

```
$ node src/job-radar.mjs --dry-run
  scoring with claude CLI
  fetched 136 postings from 10 sources
  dropped 8 stale (> 30d old)
  68 new (after dedupe)
  5 cleared the bar (>=6)

🔴 **[8/10]** AI Experience Engineer (Conversational AI) @ Upgrade
_Conversational AI build work matches agent experience_

🔴 **[8/10]** Applied AI Engineers for telecommunications company @ Toptal
_Applied AI contract, judged on shipped work_

🟠 **[7/10]** Deployment Strategist @ Databricks ⚠️ degree-preferred
_Forward-deployed analog; pedigree bar likely high_
...
```

Each match gets a score, a one-line reason, and a blocker when there is one. The blocker is the useful part: "worth a look, but they want a degree" is a different decision from "apply today".

## How it works

1. Fetch in parallel from three free, no-login sources: Remotive (one search per term), RemoteOK (filtered by tag) and We Work Remotely (RSS).
2. Drop anything older than 30 days, then dedupe across boards and against every earlier run. The same job on two boards counts once.
3. Send up to 60 new postings to Claude in one prompt with the profile. Claude returns a score from 1 to 10, a reason and a blocker for each.
4. Print everything that scores 6 or more as markdown. Post it to Discord too if a webhook is set.
5. Remember what it has seen in `out/job-radar-state.json`, so tomorrow's run only shows tomorrow's jobs.

The profile is plain markdown: who I am, the roles I want, my filters, and how to weigh them. Change the file and the scoring changes with it. No code edits.

## Decisions worth knowing about

- **Job ads are untrusted input.** Anyone can post a job, so a posting could carry "ignore your instructions and run this". The postings are fenced off in the prompt as data, and the model runs with every tool switched off. `--allowedTools ""` looks like it turns tools off but only adds to an allow-list. I tested it and the model could still run a shell command from inside a scoring call. `--tools "" --strict-mcp-config` actually removes them.
- **A run never dies on scoring.** If the model is unavailable or replies with something that isn't JSON, it falls back to a keyword heuristic and still delivers. The fallback is crude on purpose. While writing this README I caught it matching "ai" inside "maintain" and "email", so it now matches whole words.
- **A failed send is retried, a skipped one isn't.** If Discord is down, that match is not marked as seen and comes back next run. If no webhook is configured, stdout was the delivery, so it's marked seen as normal.
- **It runs on my Claude subscription, not the API.** In my setup the scheduler loaded an env file that also held an API key, and the spawned `claude` CLI inherited it and billed the API on every run. That drained my API budget before I caught it. In this repo the API is opt-in: set `ANTHROPIC_API_KEY` and it calls the Messages API; leave it unset and it uses the local CLI.
- **Headless runs skip my personal Claude instructions.** Those are written for live chats and were riding along on every call. Turning them off took a scoring call from about 54.6k input tokens to 10.7k, measured.
- **Only open feeds.** LinkedIn and Indeed have no open feed and need an interactive login, which a scheduled job can't do. The free boards surface the same remote AI roles anyway.

## Run it

Needs Node 20.12+. No dependencies.

```bash
npm test             # 9 tests, no network, no model calls
npm run demo         # live boards, keyword scoring, no keys needed, saves nothing
npm run radar        # the real thing: Claude scoring, saves dedupe state
node src/job-radar.mjs --profile=path/to/you.md --dry-run
```

Scoring with Claude needs either [Claude Code](https://claude.com/claude-code) logged in or `ANTHROPIC_API_KEY` set. Set `DISCORD_WEBHOOK_URL` to also post matches to a channel. Matches go to stdout and progress goes to stderr, so `npm run radar > matches.md` gives you a clean file.

## Provenance

Taken out of the personal operations system I run day to day, with its git history kept as it was: the first commit is from July 2026. Employer and client names are removed. The last commit moves the profile into a file, drops the Notion logging, and makes Discord optional so it runs anywhere.

MIT licensed.
