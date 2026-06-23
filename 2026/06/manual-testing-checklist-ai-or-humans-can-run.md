---
author: Edgar Mlowe
title: "A Manual Testing Checklist That AI or Humans Can Run"
description: "A practical way to turn manual testing into a repeatable method, where the checklist and evidence standard matter more than whether a human or an AI runs it."
date: 2026-06-24
tags:
- artificial-intelligence
- tools
- troubleshooting
---

In [a recent post](/blog/2026/06/ai-as-teammate-not-autopilot-upgrading-rails-5-to-rails-8/), I wrote about upgrading our internal Timesheet app from Rails 5 to Rails 8 and using AI as a teammate rather than an autopilot.

One part of that story deserves its own write-up: the manual testing checklist.

I do not mean "click around and see what happens." I mean a written test method with explicit expectations, evidence rules, and pass/fail standards. A method that can be run by a human or by an AI browser agent without changing what "done" means.

That distinction matters.

## The problem with most manual testing

Most manual testing is too dependent on the operator.

One person tests carefully. Another person tests quickly. One person notices a console error. Another only looks at the page. One person writes down what failed. Another says, "looks good to me."

That kind of testing can still be useful, especially when the application is not well-covered by automated tests. But it is hard to compare runs, hard to review later, and hard to reuse. It also breaks down quickly when you try to involve AI, because AI has a bad habit of telling you everything passed unless you pin it down hard.

What I wanted instead was simple:

> The operator can change. The testing standard should not.

Once I saw the problem that way, the checklist became less about "manual testing" and more about making the test method portable.

## What I wanted the checklist to do

I built the checklist during a risky Rails 8 upgrade on a system that handles hours, billing, reports, and permissions. Some parts of the app had automated coverage. Some important paths did not.

I needed something that would:

- verify visible workflows end to end
- work on a throwaway database
- produce evidence I could review later
- fail loud on server errors and browser errors
- be clear enough that either a human or an AI could run it

That last point was important. I was not trying to create a checklist "for AI." I was trying to create a checklist whose quality did not depend on who happened to be running it.

## The minimum structure that made it work

The checklist did not need to be fancy. It just needed to be disciplined.

Each test case needed a few things:

- a clear starting state
- the exact action to take
- a visible expected result
- a place to record what was actually seen
- a screenshot or similar evidence artifact
- an explicit fail rule

A simple example looks like this:

```text
Test: Login works for seeded admin user

Start state:
- Logged out
- Fresh browser session
- Throwaway test database

Steps:
1. Open the login page
2. Enter the seeded admin email and password
3. Submit the form

Expected:
- Redirect to dashboard
- Dashboard heading visible
- No server 500
- No red browser console error

Record:
- Actual URL:
- Actual visible text:
- Screenshot path:
- Pass / fail:
- Notes:
```

That is already much better than "test login."

The key is that the expected result is visible and concrete. Not "seems fine." Not "works." Something a second person can review later and say yes or no.

## Why the evidence rule matters

The evidence rule turned out to be the most important part.

If the operator is a person, evidence prevents memory from becoming the source of truth.

If the operator is an AI, evidence prevents bluffing from becoming the source of truth.

For the Rails 8 upgrade, every checklist run recorded what was actually seen and included screenshots. Any 500 response was a failure, even if the page mostly rendered. Any obvious red console error was a failure, even if the visible UI looked close enough.

That mattered because some failures are surprisingly easy to miss.

In my case, one of the regressions was a one-word logger mismatch that caused every logged-in request to return a 500. Another was a removed Rails date-formatting API that broke calendar endpoints. Another was time-summing behavior that changed under the hood and crashed totals. Some of these were caught by the automated suite. Some were only caught by running the real workflows and insisting on evidence.

Without an evidence rule, those findings are easier to wave away. With an evidence rule, they become reviewable artifacts.

## The checklist has to control the environment too

A good checklist does not only define steps. It defines the test environment.

For this kind of work, the checklist ran on a throwaway database with known logins and known data. It always started from a clean logged-out state. The point was to make each run comparable to the last one.

That solves a few common problems:

- "It worked on my machine" becomes less likely when the data is controlled.
- AI does less wandering if the expected state is explicit.
- You spend less time arguing about whether a failure was caused by stale data.

This is one reason I do not think of the checklist as informal testing. It is closer to a manual harness for the parts of the system that do not yet have enough automation.

## How a human runs it

For a human tester, the checklist is mostly about discipline.

The human still provides judgment. A human may notice that a page technically loads but the wrong data is shown, or that the flow feels broken in a way a narrow expected result did not fully describe.

But the checklist prevents that judgment from becoming vague. The human still has to say what step failed, what was seen, and what evidence backs it up.

This does two useful things:

First, it makes review much easier. You can look at a failure report and understand what happened without repeating the whole run immediately.

Second, it makes the checklist improve over time. Every ambiguous test teaches you where the expected result was underspecified. Every repeated failure teaches you which manual checks deserve to become automated later.

## How an AI runs it

For an AI browser run, the checklist is even more valuable.

Left alone, an AI will often optimize for completion. It reaches the end of a flow and reports success too easily. That is not dishonesty in the human sense. It is a failure mode of the medium. If the prompt is loose, the model fills gaps with optimism.

The checklist counters that by making the AI produce evidence instead of opinions.

The AI does not get to say "calendar works." It has to say what URL it reached, what text it saw, whether an error appeared, and what screenshot supports the claim.

That does not make AI manual testing perfect. It still needs human review, especially for important failures or subtle behavioral questions. But it makes AI much more useful because now it is executing a standard rather than improvising one.

## The operator changes. The method does not.

This was the key design point for me.

If the checklist is good, then the difference between a human run and an AI run becomes smaller than people expect. The operator changes, but:

- the steps stay the same
- the expected results stay the same
- the environment stays the same
- the evidence rule stays the same
- the fail rule stays the same

That is what makes the method portable.

And once the method is portable, you can use people where judgment matters most and AI where repetition or breadth helps most, without lowering the bar.

## What it caught in the Rails 8 upgrade

In the Timesheet upgrade, the checklist paid for itself.

It found real regressions that the existing automated suite had not covered well enough. Those included:

- logged-in requests failing because of a logger incompatibility
- broken calendar paths because of removed Rails formatting APIs
- totals and report paths crashing because semantics had changed under the hood

Just as important, the checklist improved the automated tests afterward.

Some of the Rails 8 regression specs I added later exist only because the manual run told me exactly which paths had slipped through. The manual run did not just find bugs. It identified missing tests.

That is another reason I like this method. Manual testing is often treated as disposable. I think that is a mistake. The best manual findings should feed the suite.

## What this method is good for

This kind of checklist works especially well when:

- you are upgrading a legacy system
- the automated suite is incomplete
- the workflows have visible outcomes
- the application state can be controlled
- you need reviewable evidence

It is also a good fit when you want to use AI in testing without giving it too much freedom to define success for itself.

## What it is not good for

This is not a replacement for unit tests, request specs, end-to-end automation, or property-style verification.

It is also not ideal for problems where correctness is mostly invisible. If the risk is "the total is numerically wrong but still looks plausible," you need stronger checks than screenshots. In my Rails 8 work, that was exactly why I stayed cautious around hours and money logic. A checklist helps a lot, but not every truth is visible in the browser.

So I think of the manual checklist as a bridge:

- better than ad hoc clicking
- weaker than full automation in the long run
- extremely useful while the stronger automation does not exist yet

## What I would keep doing

If I were setting this up again, I would keep the same principles:

- make the expected result visible and concrete
- control the environment
- require evidence
- make every error a real failure
- design the checklist so the operator can change without changing the standard

That last part is the real idea.

The checklist is not good because it is manual.

It is good because it makes the testing method explicit enough that a human or an AI can run it and still be judged against the same standard.

That is what made it useful in the Rails 8 upgrade, and that is why I expect to keep using it well beyond that one project.
