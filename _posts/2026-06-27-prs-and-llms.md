---
layout: post
title:  "PRs and LLMs"
date:   2026-06-27 03:00:00 -0700
categories: engineering
---

We are hiring at Feldera, so I have been talking to many software engineers. These calls tend to reveal where the "meta of software engineering" is at. Often the signal is that there isn't a good signal and people are looking for new opportunities for various reasons.

This time it's different: Senior engineers who were interviewing kept raising the same complaint.

These engineers now review an unholy amount of PRs written by coding agents. They struggle with it for three reasons.

1. Reviewing now takes up most of the time they used to spend writing code.
2. They do not perceive AI-generated code as 'good enough'
3. They worry about merging large changes into their product in such a short time

I don't claim to have a general solution for this, but here is what we are starting to do at Feldera:

## Reviewing consumes the time I used to spend writing code

One often quoted response is to stop requiring human reviews. That may work for products where correctness issues have a low cost. But I don't think it's a good solution for us.

Hence, our approach at feldera is to **shift _more_ of the "burden of proof" that a PR is ready to the author**.

Our old rule was that each PR must either add or update existing unit tests, and major features had to include integration tests.

I believe this isn't enough anymore.

Here are some ideas we are experimenting with at the moment:

- Set a 99% coverage target for new repositories. This forces agent-written code to include tests, otherwise it won't get accepted. LLMs can usually write the missing tests once coverage tooling such as `llvm-cov` identifies the gaps.
- Add a rule to `CLAUDE/AGENTS.md` that new tests are validated to fail by reverting the change to make sure the tests actually capture a real issue.
- Each substantial PR must run autonomous end-to-end testing. An agent runs the product end-to-end (started by the PR author). The agent's job is to validate the new feature/bugfix in different configurations, environments, and failure cases. The test summary should be included in the PR description.
- Invest in tests that constrain agent-written code: APIs used by clients must include an executable specification or oracle. Property-based testing runs the same generated inputs through the spec and the implementation. Reviewers should read the spec carefully and skim the implementation. The PR author writes/generates the tests.
- Design executable specifications and implementations so they are interchangeable. It should be easy to replace an implementation with its specification, or a specification with the implementation. This makes it easier to isolate bugs in individual components.

Most production software is not structured this way because maintaining executable specifications for every API and component is very labor-intensive. But in a world where writing code is free it turns out to be a great idea.

## The engineer does not trust AI-generated code

There are different opinions about how good LLM code is. One engineer with little experience might be amazed by it, an engineer with 15 years of experience might be appalled.

Some of that disagreement is taste. Some of it is measurable. To make reviews less subjective, we try to evaluate generated code along five points:

1. It works
2. It uses 'the right' data structures and has 'the right' algorithmic complexity
3. It exposes and uses 'the right' API surface
4. The implementation is written using clean code principles (DRY, etc.)
5. If it matters: It is fast enough

(1) mostly requires someone to state the expected behavior clearly and provide a representative test corpus that gives an LLM enough context. For many applications that gets you quite far. (2) and (3) usually need a careful human review, (4) can often be improved by encoding the right coding conventions in your `CLAUDE.md` or `AGENTS.md`.

(5) is where I expect engineering teams like ours to spend most of their time.

<aside class="not-prose my-10 rounded-2xl border border-teal-100 bg-teal-50/60 p-6 font-sans shadow-paper sm:p-7">
  <p class="text-xs font-semibold uppercase tracking-[0.2em] text-teal-700">
    Rust's testing ecosystem
  </p>
  <p class="mt-4 text-[0.97rem] leading-7 text-slate-700">
    We write mostly Rust. And the ecosystem is great to establish (1). If we need a PR to carry this much proof
    the tooling makes it cheap. We use the following libraries at Feldera excessively:
  </p>
  <ul class="mt-4 space-y-3 text-[0.95rem] leading-7 text-slate-700">
    <li>
      <a href="https://crates.io/crates/rstest" class="underline decoration-teal-300 decoration-1 underline-offset-2 transition hover:decoration-teal-600" target="_blank" rel="noopener"><code class="rounded bg-white px-1.5 py-0.5 font-mono text-[0.85em] text-teal-800">rstest</code></a>
      for fixtures and parameterized cases, so one test body covers a whole
      table of inputs.
    </li>
    <li>
      <a href="https://crates.io/crates/proptest" class="underline decoration-teal-300 decoration-1 underline-offset-2 transition hover:decoration-teal-600" target="_blank" rel="noopener"><code class="rounded bg-white px-1.5 py-0.5 font-mono text-[0.85em] text-teal-800">proptest</code></a>
      to generate inputs and shrink any failure down to a minimal
      counterexample.
    </li>
    <li>
      <a href="https://crates.io/crates/proptest-state-machine" class="underline decoration-teal-300 decoration-1 underline-offset-2 transition hover:decoration-teal-600" target="_blank" rel="noopener"><code class="rounded bg-white px-1.5 py-0.5 font-mono text-[0.85em] text-teal-800">proptest-state-machine</code></a>
      to drive random sequences of operations against a reference model and
      assert the implementation and the model never disagree.
    </li>
    <li>
      <a href="https://crates.io/crates/loom" class="underline decoration-teal-300 decoration-1 underline-offset-2 transition hover:decoration-teal-600" target="_blank" rel="noopener"><code class="rounded bg-white px-1.5 py-0.5 font-mono text-[0.85em] text-teal-800">loom</code></a>
      to exhaustively check lock-free code against every interleaving the memory
      model permits.
    </li>
    <li>
      <a href="https://crates.io/crates/shuttle" class="underline decoration-teal-300 decoration-1 underline-offset-2 transition hover:decoration-teal-600" target="_blank" rel="noopener"><code class="rounded bg-white px-1.5 py-0.5 font-mono text-[0.85em] text-teal-800">shuttle</code></a>
      to hunt the same class of concurrency bugs in larger programs, where
      loom's exhaustive search gets too expensive and randomized exploration
      wins.
    </li>
  </ul>
  <p class="mt-5 text-[0.97rem] leading-7 text-slate-700">
    Libraries are a mechanism. More important is that our engineering org
    thinks about code in terms of specifications, invariants, pre- and post-conditions.
    None of this was a bad idea before LLMs, just that it's much more important now to handle the
    increased velocity.
  </p>
</aside>

## Merging changes faster than the codebase can absorb

A senior engineer knows which areas of the codebase are safe, risky, or outright dangerous to change. This is knowledge a new engineer (or LLM) does not have. It's learned from too many 3 a.m. pages.

That institutional memory is valuable. At Feldera, this is where senior engineers are expected to push back and slow things down.

That being said, having too much code that falls in the 'risky' or 'outright dangerous' territory has now become an even bigger liability. Agents are not slowed down by reading or changing very complicated code. But given a module riddled with footguns an agent often performs even worse than a human.

That code usually falls into one of a few categories. It may have grown into a god module that should be split up. It may have accumulated complexity that needs review. Or it may be underspecified, undertested, and below the standard we now need. Luckily, AI also changes the cost of leaving code in that state.

## Makes sense?

Does it seem appealing to you to work in such an engineering organization? 
We are hiring [Rust backend and product engineers](https://jobs.ashbyhq.com/feldera).

Or do you think it won't solve any of the "PR-slop" problems? I'm happy to hear about that too.
