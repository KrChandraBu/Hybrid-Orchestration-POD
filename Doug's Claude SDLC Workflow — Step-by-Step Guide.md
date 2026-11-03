# Doug's Claude SDLC Workflow — Step-by-Step Guide

Oct 30, 2026 · @Chandra Buchha

## Overview

Doug demoed his team's Claude Code SDLC plugin ("DMS TLC", v0.7, maintained by Humpy) on the DM Services repo during the 30 Oct checkpoint. It's a repo-based workflow built on Anthropic's intents idea. Each piece of work moves from idea to close, a person approves every transition, and Jira is updated only at the end.

&#91;embedded content: Doug's SDLC workflow · 7 stages, human review at each arrow\]

Simple work can skip the spec and plan. Larger work gets both, and the plan is the last review before any code is written.

## Step 1 — Set up your tools

The team already has Claude Code. Kroger isn't licensing Claude Desktop, so Doug's teams use T3 Code as the front end.

1. **Claude Code** — already set up for the team.
2. **T3 Code** (optional, recommended) — runs several Claude sessions across several repos at once, so you can switch between them without changing windows. It's open source, so it isn't in BigFix.
   - Mac: install with Homebrew (`brew install …`).
   - Windows: install with WinGet or Scoop, if a Windows build is available.
3. **IDE** — use whatever you're comfortable with. Some of Doug's team use Zed, a lightweight editor, alongside T3 instead of IntelliJ.
4. **Model** — no restriction; use what's available. Doug uses Opus 5.5 for almost everything and only drops to Sonnet for very basic tasks. He said Opus 5.5 costs much less than 5.4.
5. **Install the SDLC plugin from the marketplace.** In Claude Code, update the plugin marketplace, then install or update the DMS TLC plugin. If reload misbehaves, restart Claude Code.

> Plugin, skill and command names in this guide are as heard in the recording ("DMS TLC", "DM setup", "DM init", "DM intent"). Check the exact spelling in the plugin README Doug is sending.

## Step 2 — Set up the project in your repo

Run setup on a repo that already has code in it, so the plugin can learn your structure. That's why Doug asked us to get the Hybrid Orchestration branches merged to `main` first.

1. **Run DM setup** (brownfield). It reads the project, works out where everything is, updates the repo's Claude config (CLAUDE.md), and creates the `DMS TLC` directory. You don't build the structure by hand.
2. **Run DM init.** It creates the project YAML and templates under `DMS TLC`, and asks setup questions.
3. **Pick a sizing method** when init asks: story points, hours or T-shirt sizes. The workflow uses it to compare human estimates against actual AI effort, which is how it shows cost savings.

Doug wasn't sure whether setup or init runs first (he thought setup, then init). The README and the upcoming videos confirm the order.

What you get: a `tickets` area with **active** tickets (work in progress) and **archived** tickets (done).

## Step 3 — Capture an intent

Every piece of work starts as an intent: a short statement of the problem and the outcome you want. You don't write it yourself; the intent skill interviews you.

1. **Start the intent skill** and describe the need in a sentence or two (Doug typed a one-line request for self-service SQL warehouse provisioning).
2. **Answer its questions.** It runs a Socratic "grill me" interview and offers multiple-choice answers. In the demo it asked how to size the warehouse, who can use it, how many warehouses, what the first slice covers, which component creates it and where the endpoint lives.
3. **Let it read your ADRs.** It reads the repo's architecture decision records and follows them. In the demo it chose a new platform module over the RASC service because an existing ADR says RASC services stay thin.
4. **Review the intent files.** Each intent has:
   - a **Markdown file** for people: problem, references, proposed outcome, hard constraints, explicit non-goals
   - a **YAML ticket file** for the agent: epic, creator, ticket type, estimates (human without AI, human with AI, AI before and after the work) and Jira details
5. **Revise or park it.** You can revise the intent, capture another one, or stop here. An intent can stay as an idea (with its ADR) without becoming a ticket, so the discussion isn't lost.

Doug's emphasis: get the **proposed outcome** right. It drives everything downstream.

## Step 4 — Turn the intent into tickets

1. **Attach an epic.** Epics sit above intents and map to Jira features. Attach an existing epic or create one.
2. **Check the size.** The skill suggests whether the work fits one ticket. If you think it's too big, tell it to split ("split into two"). Doug split a 13-point story into REST service and tables (5 points) and provisioning (8 points), which became work items 63 and 64.
3. **Review each split intent** to confirm the split makes sense.
4. **Check the cost log.** Each work item records model inputs, outputs and Opus spend, so you can see what generating it cost.
5. **Choose whether to create Jira stories.** The skill previews each story under the epic. Pick create to publish, or **keep local** while experimenting.
6. **Commit or skip.** It asks before committing the ticket files. Then approve and continue to the spec.

## Step 5 — Review the spec and the plan

Simple intents can go straight to build. Bigger ones get a spec, and sometimes a plan.

1. **Spec** — sits between the intent and the code. It covers expected behaviours, what must be recorded, API contracts, interface changes and acceptance checks. The skill asks design questions as it goes (in the demo: resource lifecycle states, and REST conventions). It applied the KTD V4 REST style guide because those conventions are loaded into the agent.
2. **Plan** — for larger specs. It lists which files will change, how, and what good looks like. **This is the last review point before code is written.**
3. **Change anything, at any stage.** Edit the file by hand, or tell the agent, e.g. "update the spec on work item 27 to include …". The agent also flags when your change conflicts with another requirement. You can go back up to the spec or the intent if the plan shows the intent was wrong.
4. **Focus your review on interface changes.** Doug treats these as more important than how the code works inside.
5. **Weigh review time against build cost.** A build might cost about $7 in Claude usage. Sometimes letting it build and judging the result is cheaper than hours spent interrogating the spec. Review expensive work more closely.

## Step 6 — Build, test, verify

The build phase runs a build → test → verify loop until the work passes, and only then allows a pull request.

1. **Define success before the first build.** Doug's main lesson from a greenfield project: when he didn't state the outcome, the build agents assumed one and built the wrong thing.
2. **Write hard rules down early**, as ADRs, as intents that update the Claude config, or in CLAUDE.md directly. Examples Doug gave:
   - how to test a container image
   - run tests with pytest
   - 100% code coverage
3. **Work test-first.** Write the test, show it fails, then let the agent iterate until it passes.
4. **Let it fan out.** The skills spin off sub-agents for development and testing and run some work in the background.

Some rules are already built into the skills. Docker and Kubernetes specifics probably aren't, so we need to add ours.

## Step 7 — Pull request, Jira and close

1. **Create the pull request** once the build loop passes. The skill asks first; it never opens PRs on its own.
2. **Publish to Jira** when you're ready. Jira becomes the manifest of finished work; the detailed tracking stays in source control. Scrum masters get their tickets without anyone hand-writing them.
3. **Close the ticket.** It moves to the archived tickets, and the YAML records the AI's after-work estimate next to the original estimates.

Throughout, anything that changes the outside world counts as a mutation and needs your confirmation: commits, pushes, PRs, Jira tickets, moving between stages. The skill never creates a release on its own.

## Tips and cautions from Doug

- **Write ADRs for every decision, even small ones.** Agents read them even when people don't. A testing-strategy ADR keeps the agent from going against the agreed approach.
- **Stay on standard patterns.** The models know common designs and architectures well, so standard choices get better results.
- **Don't aim for perfect work items.** Create another intent later to correct or refine one. Small housekeeping intents (fix a race condition, clean up docs or the Makefile) are normal.
- **Watch for merge conflicts on work-item state.** If two people have work items like 62–64 open, the state files collide. Doug's team is moving work-item state to its own branch that won't need PRs.
- **Context is managed on purpose.** Models are used read-only, and the folder structure keeps context small so the workflow holds up at enterprise scale (about 100 work items, stale docs).
- **An Ask/Help skill is coming.** It will tell you what state a project is in and what to do next. It isn't in the current version.
- **Expect a learning curve.** The plugin gives you skills plus docs on which to use and in what order. The videos are meant to confirm you're using it correctly. It's a process, not sample code.
- **Pace yourself.** Several parallel sessions mean constant critical review, which Doug says is tiring after about six hours.

## Applying it to Hybrid Orchestration

The Hybrid Orchestration repo has architecture docs but its code is still on a feature branch, so Doug's "project" for it is empty. Getting code into `main` is the first step.

- [ ] Merge the ready components (TWS adapter, Databricks adapter) to `main`, and settle how their separate CI/CD pipelines will work (Tanmay)
- [ ] Install T3 Code and update the SDLC plugin from the marketplace (each developer)
- [ ] Run DM setup, then DM init, on the repo, and pick a sizing method
- [ ] Add our ADRs to the repo, including ADR-003 (KPS selected) and a testing-strategy ADR
- [ ] Write the acceptance rules before the first build: pytest, coverage target, container image test steps, no prod sources in non-prod
- [ ] Capture the first intents: the KPS deployment (Helm, CronJobs, Listener Deployment) as an epic, plus smaller housekeeping items
- [ ] Use it for diagram and image generation early on, as Chandra suggested
- [ ] Doug: send the T3 Code video and the SDLC marketplace README
- [ ] Doug and Chandrakant: get the meeting recording from Steve and post it in the chat

Source: Weekly Hybrid Orchestration Checkpoint meeting, 30 Oct 2026 (transcript and AI notes).
