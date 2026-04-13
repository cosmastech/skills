---
name: planning-conventions
description: Personal conventions for planning software changes. Covers planning doc structure, failure scenario analysis, refactoring sequencing, observability planning, and multi-model review. Activate when planning features, writing design docs, or scoping technical work.
---

# Planning Conventions

These are non-negotiable personal conventions unless explicitly overridden by the user.

- **Upfront planning docs** — When planning work, produce a planning document *before* writing code. Structure it for humans who will skim:
    - **TL;DR section at the top** — summary of approach, key decisions, risks.
    - Detailed sections below for those who want depth.

- **Multi-model plan review** — If sub-agents are available, pass the plan to a different thinking model (ideally from a different provider) for review with *no context about your findings*. Iterate between multiple models until general consensus emerges. Challenge sub-agent feedback when it's wrong — consensus doesn't mean capitulation.

- **Include failure scenarios** - Make sure to think of different failure scenarios, how we should recover from them (or IF we should attempt to recover from them), and how we will communicate those failures.

- **Identify refactoring prerequisites during planning** — If the current code structure will make the planned change difficult, call this out early. Propose preparatory refactoring MRs that ship first.

- **Plans should mention observability** - Iterate with the user to determine how the feature or code change can be observed and what monitors would indicate code health.

- **No drive-by refactors in ticketed work** — Never make unrelated refactors inside a PR/MR tied to a ticket. Unrelated changes increase blast radius for bugs and make reviews harder. Instead:
    - During planning, identify code that would benefit from refactoring.
    - Propose a *separate* refactoring MR that ships independently, ahead of the feature work.
    - Follow the principle: **"Make the change easy, then make the easy change."**
    - This requires judgment — always flag it to the user.

- **Think about the 3 AM oncall engineer** — When writing code, imagine a sleep-deprived human investigating an incident involving this code. Add context to logs, use structured logging keys (following key structure per repo convention; if none is defined, default to snake-case), and make error paths descriptive.

- **Suggest monitors** - If during coding there is an obvious opportunity for "this would make a great monitor to signal application health," you should mention it to the user. If you have awareness of the user's observability platform, offer to help them construct the monitor. Features are not done until observability is in place.
