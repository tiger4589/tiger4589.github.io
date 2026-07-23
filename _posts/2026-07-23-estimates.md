---
layout: post
title: "Estimates: Science, Art, or Educated Guess?"
date: 2026-07-23
tags: estimates software-development agile scrum story-points
---

In my experience, many experienced developers still struggle with estimates. The reason is different for each team, but the outcome is usually the same: if estimates are not handled well, misunderstandings happen between teams and management. Developers might be seen as struggling to complete a sprint, or the opposite, it might look like they don’t have enough to do.

This is my personal opinion based on what I’ve seen in projects. It’s not a universal truth, and different teams can absolutely succeed with different approaches. But these are the pitfalls I keep seeing, and what helped us get better.

<!--more-->

# Why this topic?

In one of my recent client projects, I saw a clear lack of shared understanding around how to estimate, and what should be ready before estimating. That created mistrust with management around forecasting, deadlines, and deliveries.

Most estimates became guesswork translated into person-days, instead of staying abstract. A common rule of thumb was: “If this takes you half a day, we’ll give it 1 story point.” In my experience, that led to a lot of distorted estimates over time.

# Estimates importance

Estimates matter for several reasons:

- They help decide priorities: what to pick first and what to keep for later.
- They help teams plan sprint scope more realistically.
- They force conversations that uncover missing details.
- They often reveal stories that are too big and should be split.
- They support forecasting over time when used consistently.

One important nuance: estimates don’t _determine_ velocity by themselves. Velocity is an observed outcome of completed work. Estimates mainly help planning and alignment.

# What do we estimate?

For me, three dimensions matter most.

### Complexity

How complex is the change? Do we fully understand the requirements? Do we need deeper discovery before we can deliver confidently?

### Relative effort

A task can be low complexity and still require high effort. It may touch many files, involve a large refactor, or require broad regression testing even if business logic is straightforward.

### Uncertainty and risks

This is usually the hardest part. Are there unknowns? New libraries? External dependencies? Legacy behavior we don’t fully trust? The better the team gets at identifying risk, the better estimates become.

# How do we estimate?

There isn’t one perfect method. Different teams can make different methods work.

### Planning poker

Planning poker is one of the most common methods. It usually uses a Fibonacci-like sequence (1, 2, 3, 5, 8, 13, 21) to reflect increasing uncertainty and effort.

Team members vote independently. If votes differ, discuss why, then converge to a score the team can support. In my experience, the discussion is the most valuable part.

### Shirt Sizes

Some teams use shirt sizes (XS, S, M, L, XL, XXL, XXXL). Same idea, different representation. It can be easier for teams that want less numerical precision.

### Numeric sizing

Some teams use linear numeric scales (1, 2, 3, 4, ...). I personally prefer non-linear systems because they signal uncertainty better at larger sizes, but linear scales can still work if the team has clear anchors.

# Best practices

### Definition of Ready (DoR)

In my opinion, estimating is more reliable when a ticket is _ready enough_. Otherwise, you risk estimating something very different from what will actually be built.

A practical DoR can include:

- A clear description
- Links to relevant documentation
- Clear requirements
- Acceptance criteria

And anything else your team agrees on.

That said, DoR should not become rigid gatekeeping. If work is exploratory, call it out and estimate with explicit uncertainty.

### Baseline

To estimate consistently, you need reference stories (anchors). Compare new stories against known ones.

Your baseline should evolve. If your anchor is too small or too large, it will distort everything around it. Revisit it from time to time.

### Threshold

A threshold can help avoid oversized stories. For example, if a story scores high (like 13), that can be a signal to split it.

Use thresholds as guidance, not absolute law. Forcing every large story to split can sometimes add unnecessary coordination overhead.

Note that 13 here is just an example, teams can choose any threshold they agree on.

### Questions anyone?

Ask as many questions as needed during estimation. Even with a good DoR and baseline, unclear points always show up. Clarifying early improves both estimate quality and delivery quality.

### Team Sport

Estimation should involve everyone relevant to delivery, including QA/testers and other contributors. Different roles see different risks, and that input improves outcomes.

# How to get better?

Practice. Repetition. Retrospectives.

Estimate, deliver, compare, learn, adjust.

Review past sprints:

- Where did we overestimate?
- Where did we underestimate?
- Which signals did we miss?

Refine your DoR and your method continuously. If the same question appears repeatedly, add it to your readiness checklist. If something in your process adds no value, remove it.

# Why not estimate directly in hours or days?

I’ve been asked this many times.

My opinion: abstract sizing (points/sizes) usually works better for team planning because it avoids false precision early on. It also encourages discussion around scope, complexity, and risk instead of turning estimation into a time commitment.

Still, I don’t think hours/days are **wrong** in every context. Some teams use ideal hours or cycle-time based forecasting successfully. The issue is not the unit itself, it’s how rigidly it’s interpreted.

Time variance is real:

- interruptions and context switching
- dependency delays
- different experience levels

A senior might finish a task in 4 hours while a newcomer might need a day. Team-level estimation should account for that reality without becoming personal performance scoring.

# Estimates are not

### A wild guess out of thin air

Good estimates should be grounded in facts, experience, and known references. They should not be random numbers.

### A blood-oath

An estimate is a forecast, not a promise carved in stone, and not an immediate measure of individual performance.

### Always right

Estimates will be wrong sometimes, especially in the early stages of a project. The goal is not perfection, the goal is better decisions and better predictability over time.

# Final thoughts

Again, this is my personal view from real projects, not a one-size-fits-all rulebook.

- Keep a practical Definition of Ready.
- Use meaningful baselines.
- Treat thresholds as guidance.
- Ask questions early.
- Make estimation a team activity.
- Learn continuously from outcomes.
- Use a scaling method your team understands and applies consistently.

For me, estimates are educated guesses. But with better practices, they become educated guesses you can trust more.
