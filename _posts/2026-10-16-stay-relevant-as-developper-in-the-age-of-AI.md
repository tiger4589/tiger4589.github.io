---
layout: post
title: ""
date: 2026-10-16
tags:
---

Software development has been evolving for as long as I can remember. It evolves constantly, and it follows trends. Some trends improve the way we build software, while others don't stand the test of time and eventually fade away.

Currently, it's all about artificial intelligence (AI), agentic development, and everything related. But how should we keep our skills in demand? What does that mean for someone just starting their career? How should people who have been in this field for a long time move forward?

This article is my personal take on the subject. I'll share my honest feeling and how I think we should move forward.

<!--more-->

# Why am I writing this

## Personal experience and feeling

I recently shared an [AI migration story](https://tiger4589.github.io/2026/10/10/ai-migration-story.html), explaining how it went, what went wrong, and what I learned about using AI in real projects.

One specific feeling stuck with me, which I also shared in that article, is: **The code isn't _mine_ anymore**. It felt that way because when I started coding, almost 20 years ago, I fell in love with it. The feeling of writing something that solved a problem had some kind of impact on me or those around me, and later on professionally, gave me a huge satisfaction.

When I use AI, that satisfaction isn't quite the same anymore. Yes, the end result is still the same. The output is a full application that you can probably ship to production after verification and validation. However, the process is no longer the same.

Putting feelings aside, we still have a responsibility to deliver that software. It's our job at the end of the day. As software engineers, we have the bonus of good analytical skills, and we adapt easily to change. Adaptation ranges from understanding new businesses when we switch projects during our careers, to handling new requirements in the same project, to managing our pet projects on the side.

## AI changed the conversation

I've seen a lot of discussions and posts about what AI can do and how it will replace software developers. I've also seen people who are skeptical about the whole thing and pretty sure that the bubble will burst soon. However, given the pace at which AI is evolving and the level it's at right now, I believe the truth lies somewhere between these two groups:

**AI is probably here to stay, and software developers will have to adapt.**

For example, one person can now use AI to migrate applications of varying sizes from one programming language to another, or modernize a legacy codebase, but understanding what is being delivered is still critical. Developers must verify behavior through tests, review security implications, understand operational risks, and check whether the solution meets the actual requirements. The future is uncertain. We can speculate about whether AI will replace developers, but speculation alone doesn't help us decide how to adapt.

# How do we stay competitive in this field?

The way to avoid getting left behind isn't to chase every new tool. It's to develop the judgment to understand what these tools can do, where they fail, and how to use them without losing the skills that made us good engineers.

Staying employable requires both relevant skills and the ability to apply them well. AI changed how we're building software, but understanding problems, evaluating solutions, and making good decisions remain essential.

## Students and new graduates

If you're currently studying, or have recently earned your degree, don't be afraid.

AI can generate code quickly, and it can also be a useful tool for explaining unfamiliar concepts. Writing code has never been quicker, but the code still needs to be understood.

Come up with your own ideas, projects, or tools to build. Try implementing them yourself first, so you develop a feel for the fundamentals. When you get stuck, use AI to guide you toward a solution rather than immediately generating the entire implementation. You still need to understand the fundamentals of software development, the patterns, and the good and bad practices. Without these fundamentals, you'll struggle to make good technical decisions.

Knowing how to use AI tools may help you get a job, but it won't be enough on its own. Employers will also want to know whether you can use those tools to solve problems, evaluate their output, and deliver reliable software. Understanding the basics is what allows you to answer that.

Don't get me wrong, it's also important to study how to use those tools, but focus on answering the "why" questions, and you'll get there.

**Note that even though AI can generate explanations, it might still be confidently wrong. Use AI to accelerate your learning, then verify what you've learned through documentation, experimentation, tests, and independent reasoning.**

## Juniors in the field

If you're already working as a developer, you're building the experience you need. However, you need to **keep learning!** Don't just rely on AI output. There's much more than the basics to learn. Start exploring more design patterns, system design, and different architectures.

This is just the beginning of a long road.

Ask the more experienced developers on your team as many questions as you need. You need to start understanding, in more depth, why certain decisions are made. For example: [why did we use a message broker, or even, why didn't we use a message broker?](https://tiger4589.github.io/2026/09/24/when-to-use-a-message-broker.html)

Continuously asking questioning and learning from the answers will help you grow into a more experienced developer.

### Avoid this pitfall

If you consistently delegate work you don't understand to AI, you risk losing opportunities to develop the judgment needed to detect incorrect output, debug problems, and make good decisions when the tools fail.

The key balance is not avoiding AI-generated solutions. It's making sure that, as a developer, you can explain, test, and take responsibility for the solution.

## Mid-Level Developers

You should already be able to deliver features independently, troubleshoot problems, and write maintainable code.

If your team increasingly relies on AI for implementation, take ownership of the technical decisions, constraints, and validation that ensure the resulting software is reliable. Given the experience you've already built up, it should be easier to work alongside these tools and make the right decisions. Don't get complacent though - this doesn't mean you should rely only on AI tools from now on. Learning never ends. Gain more experience, especially in architecture and system design. Here's a fun experience that you can try: take an existing service in your codebase, map its dependencies, and ask a senior engineer about the decisions behind the current architecture. Then use AI to explore alternatives and compare those suggestions against your actual requirements.

And regardless of whether you're currently working with AI or not, it's the perfect time to start handling more complex problems, try making architectural decisions alongside your senior colleagues, and learn from those decisions.

When AI can generate multiple possible implementations in minutes, the ability to recognize which one will remain maintainable for years to come becomes much more valuable.

## Seniors and above

As AI-assisted development becomes more common, use your experience to make the most of these tools without compromising quality. AI tools are already useful in many development workflows, and familiarity with them may increasingly appear in job descriptions. The more important skill, however, is knowing when and how to use them effectively.

Don't be afraid to adapt. AI tools make executing ideas and building software much easier now, but they still need experienced developers to maintain and validate their output. Experience is still needed to design systems and to make the right decisions where it matters.

If you haven't used these tools yet, go ahead - don't fall behind. Trust, but verify!

### Responsibilities

Your core responsibilities remain the same, even when adopting AI.

You still have to:

- Define the requirements
- Set architectural constraints
- Establish acceptance criteria
- Review system-level changes
- Mentor less experienced developers

AI also makes some other responsibilities more important:

- Protecting sensitive data
- Evaluating tool permissions
- Checking for security vulnerabilities

The tools and workflows may change even though accountability remains.

# How do you preserve your engineering skills?

Professionally: you can review generated code while waiting for other sessions to finish, discuss it with colleagues, and examine the architecture. If you run into a bug, use your debugging skills rather than immediately delegating the problem to another agent. Consider maintaining independent tests that the AI agent cannot modify, so it cannot change both the implementation and the tests to make them pass. These tests still need to reflect the actual requirements. Being able to explain the system you've built with AI is a good recipe.

Personally: keep some pet projects around - toying with them is fun. I love writing code, and when working on fun projects for myself, I still prefer to write as much as I can manually. Once I am happy with the initial architecture and design, I might delegate the work to the AI, instructing it to follow in my footsteps. This keeps me practiced at reading the code and following it through.

## Learning loop

Learning doesn't stop. I always give the example that we keep learning how to use our hands on a daily basis. Coding is a practice, just like anything else in the world.

I personally like reading blogs and technical documentation, and watching videos if I want a quicker overview. I wouldn't advise ditching this and using only AI to learn. Remember that AI tools vary in how they access current information. Their knowledge may be outdated or incomplete, and even tools with web access can misinterpret sources.

Keep your sources close, and keep learning from them. They will always come in handy, no matter what.

Here's the learning loop I personally use:

- I discover what's new and useful by following trusted sources instead of every announcement.
- I experiment by choosing one development workflow or tool that helps me with a problem. I try it on a small task.
- I evaluate by comparing the result to my normal approach, considering the quality, maintainability, and the effort needed.
- I consolidate by recording what worked, what failed, and what I've learned. Then I decide whether the new approach is worth adopting.

### How to decide what is worth learning?

Learning something new is always nice and fun, but ask yourself: does it solve a real problem you have? This comes from experience. Whenever I learn something without actually applying it somewhere, I tend to forget it. Search for what you need at the moment, and tackle it. Another question worth asking: will it matter beyond the current hype cycle? If a change in your favorite language is merely syntactic sugar, it might not be worth exploring in depth. However, a major update that introduces breaking changes, or a new approach to memory management, is probably worth investigating.

# What does experience mean?

When AI does most of the implementations, what does experience mean? It isn't simply the ability to write code. It includes recognizing bad abstractions, choosing the appropriate trade-offs, understanding the consequences of decisions.

Applying the KISS (Keep it simple, stupid) and YAGNI (You aren't gonna need it) principles is still important. If your AI agent is building abstractions on top of abstractions that you know you wouldn't need, it's better to stop it. This will make the system you're building easier to understand.

If the AI agent decides to use a message broker where a simple HTTP call is sufficient, you have to step in. Knowing when to use what is very important, and choosing the right trade-off has a big impact on how your system will behave once you ship it.

If you need to choose between SignalR and SSE for real-time communication, then you need to understand the consequences of your choice: what are you gaining? What are you losing? What should you be aware of?

This is why continuous learning is important: it allows you to build the experience you need to make those decisions and understand their trade-offs.

# How to adapt without losing the fun of programming?

I know that not all developers enjoy the same things or the same workflow. My advice is to do what you enjoy and delegate the rest. Some might enjoy system design, and others might still value writing code. Some might enjoy seeing the end product of an idea they have, while others might still prefer to build it slowly.

This might not apply at your job, where you have to adapt along with your organization, but you can apply it on a personal level to regain some of the satisfaction that might be lost.

I personally don't like getting too involved in front-end development, and I find working on the back end much more fulfilling. I aim to do that on my personal projects as much as possible and delegate the front-end work to AI tools.

# Final thoughts

Does AI mean giving up the satisfaction of creating something? Or does it mean finding satisfaction in different things, like understanding a complex system or delivering something that would have previously taken months? I'll leave those questions for you to answer.

A mentor of mine wrote an article seven years ago, [The day when the software industry makes up its mind](https://adamtibi.com/archives/the-day-when-the-software-industry-makes-its-mind), and it still holds true today. The pace of change is faster now, making adaptation more demanding than ever. Don't avoid what's inevitable, but treat it carefully and with an open mind. Remember, at the end of the day, you're still responsible for what you're delivering, whether an AI wrote it or you did.
