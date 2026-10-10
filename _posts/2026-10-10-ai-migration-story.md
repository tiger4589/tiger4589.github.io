---
layout: post
title: "Migrating a legacy .NET Core 3.1 app to .NET 10 with Claude Code"
date: 2026-10-10
tags: ai claude-code agentic-development dotnet csharp legacy-migration domain-driven-design technical-debt
---

How do you modernize a substantial legacy application when the architecture is unclear, automated tests are missing, and production behavior must be preserved?

I've been delaying this for years now, while piling up technical debt. But finally, I decided to give AI a chance. So, how did it go? What have I learned? What's coming next?

This article will go through the process of the migration, what was used, how it was used, and where things stand today, since it's still a work in progress.

<!--more-->

# What is the product?

Back in 2019, I started helping the owner of [Kings Of Chaos](https://www.kingsofchaos.com) (KoC) with maintaining the game. One part was really slow: statistics.  
We had to remove the page because it was affecting the whole server. On top of that, for performance issues, all actions taken in the game were being deleted once they were more than two weeks old, which made the statistics incomplete at best.

That's where the decision was taken to move the statistics outside the game into another server, where we can save all the actions without deleting anything. On top of that, we introduced a web application for players to fetch their statistics, and a Discord bot that they can use in their own servers.

This is the simplified story of what the whole product is.

## Legacy code

The initial code was very simple (.NET Core 3.1):

- An API project
- A web app project
- A Discord console app project
- A background service
- A shared project: mostly services, shared between the 3 applications, to fetch data from the database.

I wasn't looking for a clean architecture back then, just working code. It was supposed to be a small application, a proof of concept, developed by me, hosted on my server. However, with time, I kept adding to it, and it accumulated so much technical debt that I kept postponing the rewrite. You have probably had the same issues with side projects.

## How it's all connected

![Legacy Architecture Diagram](/assets/ai-migration/legacy-arch.png)

Users only interacted with KoC, the web app and the Discord bot.  
KoC called the API after each action to register it.  
The background service went through the database, checked for missing action ids, and fetched them from KoC to record them.  
As you can see, all the different applications shared the same database, and each had a direct connection to it.

## Problems

I never wrote any documentation. The knowledge was all in-memory, pun intended.  
Introducing changes was not simple.  
All projects calling the database independently put stress on SQL Server.  
There were many hardcoded values, and configuration values that had to be updated in several places.  
When my server or API went down, users noticed lag in-game, because the game called my API after each action.

Yes, it became a disaster after a while.

## It's finally time

After all these years, it was time to finally migrate the project. Given that AI can do the heavy lifting, I decided to give it a try. I'll share below how I started, what I've learned, and how you can apply the same strategy to migrate similar projects that have no documentation or tests and must keep the same behavior.

# Agentic Dev Migration

## Tools

I used Claude Code, and Sonnet 5.5 is the only model I've tested this with.

## Beginning

### First step

The first thing I wanted to test out was Claude's ability to understand the code and the behavior of the whole repository. So, my first prompt was a general one:

```text
Scan all the repo, and tell me what you understand from it.
```

Claude was able to identify the projects, how the data flows between them, and even immediately flag some small issues such as: committed configuration, a stale CI pipeline that I no longer use, the reliability of the in-memory queue I designed, and some dead code that I left there because someday, I'll fix it.

### Explaining the goal

After making sure that Claude was able to scan the repo correctly, it was time to explain the goal, give the context, and ask for a plan.

I used the second prompt to establish this. I explained the goal:

- Migrate to .NET 10.
- Use clean architecture with domain-driven design.
- Store all the actions that happen in-game.
- The current behavior will change a little bit: KoC no longer calls the API.
- The Discord bot should no longer have access to the database.
- The web application should no longer have access to the database.
- The API will be the main entry point for these two applications.
- The background service will be split into workers, with each worker fetching data for one action.

I also explained where I'd like to deploy all of that and what kind of server I had.  
I also suggested a plan to migrate one action at a time.

And since this was my first time working with Claude, I also asked it to generate the files it needs to reduce prompt repetition, save decisions, and keep track of status, so each new session can pick up where the previous one stopped.

I also asked it to come up with a plan based on everything I explained, and to share it with me before generating any new files.

After refining the plan, Claude started working. It created multiple files:

- CLAUDE.md
- settings.json
- guard-legacy.sh (hook to prevent any agent from modifying the legacy code)
- All the files related to the decisions I made, the migration status, the architecture, the conventions, the fetching workflow, the configuration, the endpoints to use, the migration playbook, and a legacy map.
- 6 skills: migrate-action, migrate-ui-page, migrate-bot-command, scaffold-foundation, verify-v2, and build-staff-app.
- 2 agents: legacy-inventory and ddd-reviewer.

A closer look at two of these files:

CLAUDE.md is loaded into the context at the start of every session. It contains a short description, the repo layout, the trigger phrases (prompts), the hard rules, the tech stack, and, most importantly, a session start protocol:

```markdown
## Session start protocol (do this first, every session)

1. Read `v2/docs/status.md` (what is done / next) and `v2/docs/decisions.md`.
2. For migration work read `v2/docs/migration-playbook.md`, `v2/docs/architecture.md`, `v2/docs/ddd-conventions.md`. For the staff app and the bot fixes (rows 9a to 9h) read `v2/docs/staff-app.md` (the spec) instead of the playbook's slice recipe.
3. Don't ask the user to repeat context that is in these files.
```

The decisions.md file contained all the decisions that I made, and the decisions that I let Claude make, for example:

```markdown
## Decided by the product owner (2026-10-05)

| #   | Decision                                                                                                                   | Notes                                                                                          |
| --- | -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| D1  | Clients no longer POST data. Background fetchers pull from KoC.                                                            | Legacy `Log*Controller` ingest endpoints are **not** ported.                                   |
| D2  | KoC will expose a `get-latest-id` endpoint **per action**.                                                                 | Owner implements it on the KoC side. Exact URLs: see `koc-endpoints.md`.                       |
| D3  | Attack, Theft and Poison each have their own `report_id` space (`attack_id`). **Sab and Recon share one** report-id space. | Fetch cursors are per ID space: `Attacks`, `Poison`, `Theft`, `SabRecon`.                      |
| D4  | Every call to KoC carries an API key header.                                                                               | Owner implements it on the KoC side. Header name and key are configuration.                    |
| D5  | One Blazor Web App for all UI.                                                                                             | Replaces `KoCStatsWeb` (MVC) and `KoCStats` (WASM).                                            |
| D6  | Discord bot: slash commands and components only.                                                                           | Legacy `!prefix` commands are re-expressed as slash commands/subcommands where still relevant. |
```

```markdown
## Technical decisions (made by Claude, open to change)

| #   | Decision                                                                                              | Why                                                                        |
| --- | ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| T1  | .NET 10, EF Core 10, SQL Server; Aspire orchestrates dev; services run without Aspire on IIS.         | Requested target.                                                          |
| T3  | CQRS-lite: commands go through aggregates; queries project straight from EF (`AsNoTracking`) to DTOs. | Stats queries are aggregation-heavy; loading aggregates would be wasteful. |
| T4  | No MediatR / AutoMapper / FluentAssertions. Hand-rolled handlers, Mapperly, AwesomeAssertions.        | Commercial licences.                                                       |
```

### Developers are lazy

Yes, I am another lazy developer! After the first two prompts, I also asked Claude to introduce a `UserReadMe` file. I asked it to include the prompts I would use, in playbook order, so that any session can pick up the next step.

### Prompts

All I had to do after this was pick the prompt from that file, copy it, and paste it into Claude.

For example, one of my prompts was:

```text
start migrating attacks
```

That prompt was mapped to a skill by the trigger table in CLAUDE.md.

## Migration Process

### Sessions

I started using the prompts Claude prepared, step by step. Each session took roughly 10 to 30 minutes.

### Verification Process

After each step, I read the summary that Claude generated. If anything didn't seem correct, I flagged it to be fixed.

When it comes to code verification: the code review agent made sure that the architecture was correctly implemented. The infrastructure tests made sure that there were no forbidden references. Unit and integration tests made sure the logic was correct. What I verified was my domain business logic. Is it doing what it was doing before? Are there any changes? On top of that, I tested all the functionality in the web app, the Discord bot, and the new staff web app.

Once all my verifications were done, I moved on to the next prompt.

### What went wrong

Everything was green and some of it still didn't work. The Previous/Next buttons on the `/top` commands did nothing in Discord. All the tests passed, but Discord.Net had added the command group name in front of every button id, so no button matched its handler. I only found out because I tried the bot myself. Claude then did the right thing: it first wrote a test that reproduced the failure (all 14 button ids failed), and then fixed it.

A worse problem was silent omissions. When I asked Claude to compare the features in the legacy system and the new one, it found nine differences I had never decided on, such as a missing page and charts that had quietly become tables. A second pass found 23 more. One of them was a page that KoC's own site depends on. Every slice had passed its own tests, so nobody had noticed.

My conclusion: tests written by the same AI tell you the code does what the AI thought it should do. They don't tell you it does what the old system did. Compare against the legacy system, and test the real thing yourself.

## Migration End

### Numbers

The full migration took around 31 sessions.  
The git diff showed +126,162 lines added and nothing deleted. These figures were measured before the fine-tuning.  
The total number of generated files was 1,001, including the agents, hooks, skills, settings, workflows and documentation.  
All of this took 5 days, working only at night.

### New Design

![A diagram showing how the new system design looks](/assets/ai-migration/new-arch.png)

The new design removed most of the problems we faced before.  
The game servers no longer interact with my server, so there's no more lag in case of downtime.  
The new workers fetch data from KoC and insert it into the database.  
Writes through the API are limited to staff operations, which I expect to use once every one to two months.  
The game rules the bot needs for its calculations are now fetched from KoC, so they're no longer hardcoded.  
A new staff web app was added, making it easier to manage the statistics platform.

## What came next

### fine-tuning

After finalizing the initial migration plan, I started fine-tuning some of the functionalities that were migrated. These sessions were much quicker due to the context and documentation Claude had created during the migration.

One example was the ability to start and stop workers from the staff app. I previously had to do that manually, directly on the server.

### New functionalities

I also started adding new features for both the staff and the players.
Any idea that I had in mind and kept delaying over and over again was finally added.

One example was the ability to reset all the statistics from the staff app.

## Next steps?

### UAT testing

At the moment, the new system is running alongside the old system. Both staff and players are testing to see if everything is working correctly.

### Numbers comparison

Besides the testing itself, we need to make sure that all the statistics produced are equal to what we currently have. Even though the code compiles and the applications are running, we'll need to double-check the output. This is important, as the legacy code had no written tests. All the new tests were written by the same AI.

### Full documentation

Full technical documentation will be generated to give a clean overview of how everything works together. This documentation will target humans more than just Claude itself.

### Switch over

Once everything is validated, we'll have the green light to switch to the new system. The current legacy system will be retired, and the new system will take over.

The estimated numbers after deleting the old system would be:

- -232,014 lines removed
- -630 files

# What I learned along the way

## Context is the real prompt

AI doesn't need a better prompt, it needs a better memory.

Each session starts from zero. Claude doesn't remember yesterday's decisions, the shortcut we agreed on, or where we stopped. So everything it needs to know has to live somewhere it can read. In this migration, that was three layers:

- **The legacy code:** what exists today and how it behaves.
- **My intent:** the goal, the target architecture, and the changes in behavior I wanted.
- **Files that carry the memory between sessions:** what we decided, why, and where we stopped.

Remember the "in-memory, pun intended" problem from the start of this article? This is the other side of it. The knowledge that lived only in my head had to be written down, and Claude helped me do it.

### Decisions need reasons

The decisions file lists every choice I made with a short reason, such as "no import from the legacy database, backfill from KoC instead". It does two things. Claude doesn't reopen questions we already settled, and I don't re-explain them each session. Months from now, I'll also know why we chose that.

### Rules should be enforced, not only stated

Some context is a constraint: never edit the legacy code, never copy secrets, no commercial libraries. I wrote these in `CLAUDE.md`, but a rule that only exists as text can be missed. So the important ones are enforced: a hook blocks edits to the legacy projects, and architecture tests fail the build if a layer references something it shouldn't. Context tells the model what to do, and guardrails make sure it happens.

### Context goes stale

One of the hard rules in my `CLAUDE.md` is that every step ends by updating the status and decision files. That's what keeps the next session reliable. A wrong note is worse than a missing one, because Claude will trust it. Treat these files like code: if the behavior changes, update them.

If you work with AI tools, think about what a new team member would need on their first day: what the system does, what was decided and why, what's off limits, and what to do next. Write that down, and keep it current.

## Verification is non-negotiable

Reading 100,000+ lines of code in five days is impossible. Validating all these changes manually is even harder. However, you still need to verify the output. Adding guardrails, tests, code-review agents, and extra validation steps will certainly help. After all, whatever you ship will have your name on it, and you're responsible for it. In my case, the easiest way to verify the output was to compare the numbers calculated by the legacy system with those calculated by the new system.

## The code isn't _mine_ anymore

This is one of the most important things that stuck with me. Even with all the spaghetti code I managed in the legacy system, which I am guilty of allowing to get to that point, I was still able to navigate it very easily. It lived in my head as much as in the IDE. I am responsible for the new code, and I have to maintain it, but the feeling toward it is not the same. I believe this is something that many developers are facing. There's a really nice [post by David Whitney](https://davidwhitney.co.uk/blog/2026/02/17/existential_dread_and_the_end_of_programming/) that explains how I feel about this.

# A reusable recipe

If you have a similar project, I'd suggest following these steps:

- Scan the repo: see what the AI understands from your current repo.
- Explain your goal: explain what the target is and what is expected.
- Get a plan and confirm it: don't just let the AI start working without a solid plan.
- Generate docs, hooks and skills that relate to the plan and the decisions you made.
- Write prompts per step: to reduce repeated prompts and re-explaining.
- Verify per slice: verify the output after each step.
- Keep the context files updated.
- Compare the output against the legacy system.

# Final thoughts

AI tools can do a lot of the heavy lifting, but they are tools. We should still treat them as tools. We should not forget how models work, and how they might hallucinate things that we don't need or want. Don't be afraid of working with and adopting agentic development. The field is evolving quickly, and it will probably keep evolving. A software engineer with good fundamentals and extensive experience will be able to achieve much more with AI since they can set the rules and validate them along the way.
