---
author: Edgar Mlowe
title: "Zoom Out, In, Out: A Simple Way to Make Better Technical Decisions"
description: "A practical solution design habit: zoom out to define the problem and compare options, zoom in to plan the work, then zoom out again to stress-test the choice."
date: 2026-06-25
tags:
- artificial-intelligence
- tools
---

In [my post about upgrading a Rails 5 app to Rails 8 with AI as a teammate](/blog/2026/06/ai-as-teammate-not-autopilot-upgrading-rails-5-to-rails-8/), I mentioned a simple rhythm I kept using before I wrote code:

zoom out, zoom in, zoom out again.

This post is the fuller version of that idea.

It is not a heavy framework. It is just a way to slow down long enough to get clear before you commit yourself to a technical path.

I started using it because I noticed a pattern in hard engineering work:

- people jump to solution mode too early
- trade-offs and risks get mixed together
- the plan gets detailed before the problem is fully clear
- the team argues about option A versus option B before naming what failure actually matters

AI makes this worse if you let it. It is very good at rushing into build mode.

So I wanted something simple that would keep the thinking honest.

## The rhythm

The rhythm has three moves:

1. Zoom out to define the problem and compare options.
2. Zoom in to plan the work.
3. Zoom out again to do a pre-mortem.

That is it.

The first zoom out is about clarity.

The zoom in is about execution.

The final zoom out is about humility.

The pattern works for architecture questions, upgrade strategy, ugly legacy files, and even smaller implementation choices. I have used it for "transplant or step-by-step upgrade?" and also for "should I delete this weird initializer or port it carefully?"

## The template

This is the version I write down.

```text
# Zoom Out, In, Out — A Solution Design Template

## 1. Zoom out — Problem Definition

Pain:    [who] can't [do what] because [why]

Now:     Currently, [what happens]. This is bad because [why].

After:   Should be: [what should happen]. This is better because [why].

Context: This problem lives in [where in the system]. It touches [X], [Y], [Z].

Stakes:  If this problem continues, [what else breaks / who else suffers].

## 2. Zoom out — Solutions

For each possible solution:

  Solution:   [name]
  How:        [how it works]
  Assumes:    This works ONLY IF [what must be true]
  Trade-off:  If I pick this, I lose [what — for sure]
  Risk:       If I pick this AND things go wrong, [what could happen]

Compare:
  Which trade-off am I most willing to accept?
  Which risk am I least willing to face?

→ Selected: [A / B / C]
→ Why:      [one sentence]

## 3. Zoom in — Plan

Steps:
  1. First, [___]
  2. Then, [___]
  3. Then, [___]

Done when: [___]

## 4. Zoom out — Pre-mortem

Six months from now, this failed.

Most likely reason: [___]

What I could do now to prevent it: [___]
```

You do not have to fill every field in a formal way every time. But each field exists because it forces a kind of clarity that is usually missing when a technical decision is still muddy.

## Why the first zoom out matters

Most bad decisions I have seen did not start with bad implementation. They started with a blurry problem statement.

That is why the first section begins with the problem rather than the solution.

The questions there force a few useful distinctions:

- who is actually blocked?
- what is happening now?
- what should happen instead?
- where does this live in the system?
- what is at stake if we get it wrong?

Without that, teams often end up solving an adjacent problem. Or they solve the problem locally while making the system harder to own later.

The "stakes" line is especially helpful. It forces you to stop treating every issue like an isolated technical puzzle. Some problems are local annoyances. Some are blockers for a release. Some are security risks. Some are trust risks.

Those are not the same thing.

## Why I separate trade-offs from risks

The solution section is where this template becomes really useful.

The most important distinction in it is between trade-offs and risks.

A trade-off is the cost you knowingly accept if you choose a path.

A risk is what could go wrong if the path fails.

People mix these together all the time. When that happens, every option starts sounding equally bad or equally vague.

Separating them changes the conversation.

For example:

- "This path takes longer" is a trade-off.
- "This path could hide a silent billing regression" is a risk.

Those are not comparable in the same way. One is a known cost. The other is a failure mode.

Once you write them separately, a better question appears:

Which trade-off am I most willing to accept?

Which risk am I least willing to face?

That usually reveals the real decision.

## A real example: transplant or step-by-step upgrade?

When I was preparing our Timesheet app for open source, the biggest choice was whether to do a long, step-by-step Rails upgrade or transplant the code into a fresh Rails 8 app.

The app was eight major Rails versions behind. It handled hours, billing, reports, users, and permissions. The original developers were no longer here.

The blurry version of the decision was:

"Should I do the safe path or the bold path?"

That framing was not helpful.

The template made it sharper.

The step-by-step path looked like this:

- How: upgrade one major version at a time on the real codebase
- Assumes: enough trustworthy coverage to catch regressions at each hop
- Trade-off: many sequential upgrades, carrying old baggage forward
- Risk: prolonged breakage with low familiarity and still a messy end state

The transplant path looked like this:

- How: stand up a clean Rails 8 baseline and move the code into it
- Assumes: the app is conventional enough to transplant and we can rebuild familiarity
- Trade-off: a bigger verification leap up front
- Risk: silent regressions in hours, money, dates, or SQL behavior

Once written that way, the decision became clearer.

The trade-off I was most willing to accept was the bigger leap up front.

The risk I was least willing to face was silent time or billing corruption.

That led to the actual answer:

choose the transplant, but design the verification around the silent-risk part.

That is a much stronger decision than "transplant feels cleaner."

## A smaller example: keep it or delete it?

The same method helps on much smaller and uglier problems too.

One of the Rails 8 blockers in this app was an old initializer that patched PostgreSQL type handling. It used removed APIs and broke boot immediately.

The tempting move was to delete it.

The template helped me ask a better question:

What behavior is this preserving?

Once I looked properly, the answer was not "nothing." It was preserving two pieces of behavior the app still relied on:

- reading Postgres `interval` values as the old string form the app's `Interval` class expected
- handling a custom `cron_spec` type used by scheduling fields

That changed the decision.

The question was no longer "can I remove weird legacy code?" It was "do I understand the behavior well enough to know it is safe to remove?"

I did not, so I ported it carefully instead.

That is a small example, but it shows the point of the method well. A lot of engineering mistakes come from solving the wrong question cleanly.

## The zoom in is where planning gets honest

After the first zoom out, the next step is to zoom in and make the work concrete.

This is where you write:

- what comes first
- what comes next
- what comes after that
- what "done" means

The important part is not producing a perfect plan. It is producing a plan that is specific enough to expose hand-waving.

This stage often reveals that a decision was not really made yet. If the steps do not cohere, or if "done" is still fuzzy, you probably need to go back to the earlier section.

I like this stage because it converts opinions into sequence. A lot of ideas sound good until you try to say what the first three steps are.

## The final zoom out is the most underrated part

The pre-mortem may be the most valuable line in the whole template.

Six months from now, this failed.

Why?

That question forces you to stop telling yourself the flattering story where the plan works exactly as intended.

In the Rails 8 upgrade, the most likely failure was not "the app will not boot." That kind of failure is loud and fixable.

The more dangerous failure was:

the app appears to work, but some hours or money result is now quietly wrong because runtime behavior changed under the hood.

Once I saw that clearly, it changed what I did next. I cared more about verification than I would have if I had stayed in build mode. The pre-mortem turned a vague discomfort into an explicit engineering concern.

That is what the final zoom out is for.

## Why this works well with AI

This pattern works well without AI, but I think it works especially well with AI because AI tends to be strongest at the wrong time.

It is very strong at generating options, code, and explanations.

It is much weaker at owning stakes, choosing acceptable risks, and deciding what failure is tolerable in your context.

So I like using AI inside this method, but not above it.

AI can help me:

- articulate the problem more clearly
- surface options I might miss
- pressure-test assumptions
- suggest likely failure modes
- outline candidate plans

But I still need to decide:

- which trade-off I am willing to accept
- which risk I refuse to normalize
- what "done" actually means

That is why I think of this as a control method as much as a design method. It keeps AI useful without letting it quietly become the decision-maker.

## What this method prevents

Used consistently, this little template helps prevent a few common engineering failures:

- false binaries
- premature implementation
- hidden assumptions
- blurry success criteria
- treating every downside as if it were the same kind of downside
- confusing visible progress with real risk reduction

It also helps make conversations better. When two people disagree, the disagreement is usually not "option A versus option B." It is more often one of these:

- they define the problem differently
- they assume different things must be true
- they accept different trade-offs
- they fear different risks

The template makes that visible.

## What I would keep doing

I do not use this because it is elegant. I use it because it keeps me from rushing into the wrong kind of clarity.

If I were doing it again, I would keep the same habit:

- zoom out before building
- zoom in before committing
- zoom out again before trusting myself

The point is not to move slowly.

The point is to get clear enough that speed becomes safer.

And if AI is part of the work, that matters even more. AI makes exploration cheaper. That is good. But cheaper exploration only helps if you still have a reliable way to decide what is worth building and what is too risky to hand-wave past.
