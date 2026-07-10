---
author: Edgar Mlowe
title: "AI as Teammate, Not Autopilot: Upgrading a Rails 5 App to Rails 8"
description: "A practical look at using AI for a risky Rails 8 upgrade without giving up engineering ownership, verification, or reviewability."
featured:
  endpoint: true
  image_url: /blog/2026/06/ai-as-teammate-not-autopilot-upgrading-rails-5-to-rails-8/cover.webp
date: 2026-06-23
tags:
- artificial-intelligence
- ruby
- rails
- open-source
- troubleshooting
---

![An editorial illustration of an engineer crossing from a tangled legacy software landscape toward a cleaner modern system, with AI present as a small guide rather than the driver.](/blog/2026/06/ai-as-teammate-not-autopilot-upgrading-rails-5-to-rails-8/cover.webp)

<!-- Illustration created with AI direction by Edgar Mlowe, 2026. -->

When we started getting our internal Timesheet app ready to go open source, one problem dominated the work: the backend was still on Ruby 2.4 and Rails 5.

The app still worked. That was not the issue. The issue was that both versions had been end-of-life for years. It tracks hours, billing, reports, users, roles, and permissions. Quiet mistakes matter more than loud crashes in software like that.

I used AI heavily during the upgrade. But the useful part was not "AI wrote the code." The useful part was simpler: AI made exploration cheap. I still had to own the risky decisions, the risky edits, and the proof that the app still behaved correctly.

## The rule that kept this safe

My rule from the start was simple:

> AI helps with the exploring and the first draft. I own every change.

That rule shaped everything else.

I did not want AI deciding what was safe to merge. I wanted it helping me understand old code, explain removed Rails behavior, suggest likely fixes, and argue trade-offs. If a change touched hours, money, reports, or anything else that could fail quietly, I needed to understand it well enough to explain it in plain English before I trusted it.

That is where I think a lot of AI conversations go wrong. The real question is not whether the tool can generate code. It clearly can. The real question is who owns correctness when the code matters.

## Why this upgrade was a good test

The app was about eight major Rails versions behind. The original developers were no longer here. The codebase was not tiny either: around 595 source files and roughly 130,000 lines of code.

So the job was not just "get it to boot on Rails 8." It was:

- move off unsupported Ruby and Rails versions
- preserve behavior in code that handles hours and money
- keep the diff reviewable
- learn enough of the system that we can maintain it in the open later

That last part mattered. Open-sourcing a project you do not understand is not much better than keeping it on obsolete dependencies.

## Where AI actually helped

I found AI genuinely useful in four places.

### Understanding code I did not write

Nobody who wrote the original system was still around. Before I touched a file, I often needed a quick, grounded explanation of what it did, what depended on it, and what would be risky to change.

AI was good at helping me form that first mental model faster. Not perfectly, and never without checking, but faster.

### Translating old Rails and Ruby behavior

A lot of the work was translation work. Older Rails and Ruby code had to be understood on its own terms first.

Examples like these came up repeatedly:

- `alias_method_chain`
- `to_s(:db)`
- older serialization patterns
- removed config APIs
- libraries whose defaults or method names had changed

This is a strong use case for AI. It can often tell you what the older code used to mean, when Rails changed that behavior, and what the modern replacement is likely to be.

### Separating upgrade regressions from old bugs

One of the most useful questions AI helped me move through was:

Is this broken because of my upgrade, or was it already broken before I touched it?

During manual testing I found several failures that would have been easy to fix inside the upgrade branch. Instead, I checked the same paths on the old Rails 5 version. In four cases, they failed there too. They were real bugs, but they were not upgrade regressions.

That mattered. I filed them separately and kept them out of the upgrade branch so the diff stayed about the upgrade.

### Suggesting a first draft of a fix

AI was often good at the first draft:

- here is probably what changed
- here is likely why it broke
- here is the smallest modern rewrite that keeps the same intent

But that is where its job stopped. A suggested fix is not a verified fix.

## I did not let AI choose the strategy

The biggest decision was whether to do a long, step-by-step Rails upgrade or transplant the code into a fresh Rails 8 app.

I did not want AI choosing that for me. I used it to help compare the options, but I made the decision with a simple rhythm: zoom out to define the real problem and compare options, zoom in to plan the work, then zoom out again to do a pre-mortem. AI helped me explore the options. It did not choose the path.

For this app, I chose a transplant.

Being eight major versions behind changes the math. A step-by-step path means carrying old baggage through many sequential upgrades. A transplant means a bigger leap up front, but into a cleaner modern baseline.

The obvious risk was silent regressions in hours, money, date behavior, or SQL ordering. That was the risk I cared about most. So instead of pretending it was small, I designed the checking around it.

I also ran a throwaway spike first: build a fresh Rails 8 app, move the old code in, see what breaks, and treat that result as a risk map rather than final work. AI made that exploration cheap. The real upgrade still had to be done carefully by hand.

A good example was an old initializer that registered custom PostgreSQL types. It looked like exactly the kind of strange legacy patch you want to delete during a modernization pass. But zooming out first changed the question from "can I remove this?" to "what behavior is this preserving?" The answer was: interval parsing the app still relied on, and a custom `cron_spec` type used by scheduling fields. That led me to port it carefully instead of deleting it blindly.

## A few small fixes that show the real work

A lot of the upgrade was not glamorous. It was small, exact changes that preserved intent while updating the mechanism.

One example was the development logger:

```ruby
- config.logger = Logger.new(path)
+ config.logger = ActiveSupport::Logger.new(path)
```

That one-word change mattered because a plain `Logger` no longer had a method a session-store dependency called on every logged-in request. The result was that every logged-in request returned a 500. No backend spec caught it. Running the real app did.

Another example was date formatting:

```ruby
- start_range = start_range.to_s(:db)
+ start_range = start_range.to_fs(:db)
```

That rename broke the calendar paths.

The pattern behind both fixes was the same:

keep the same behavior, change how it is written.

The quieter class of bug was even more interesting. In the hours code, a plain `sum` used to start from the first element. On modern Ruby and Rails it starts from `0`, which meant time math became `0 + Interval` and blew up until I made the starting value explicit with `sum(Interval.new)`.

That same pattern showed up elsewhere too:

- interval summation needed the right zero value instead of starting from `0`
- SQL array comparisons needed explicit `::int[]` casts
- a raw SQL `order(...)` needed `Arel.sql`
- a custom PostgreSQL type registration had to move from an old monkey-patch style to a modern `prepend` + `super` approach

None of those are exciting alone. Together, they are what a real upgrade looks like.

## The working rules mattered as much as the tool

The most useful thing I wrote down during this project was not code. It was the working rules.

The important ones were simple:

- small steps, then stop and check
- explain what you are doing before doing it
- ask before risky actions
- when I want to own a change, give me the diff and let me apply it myself

I also kept a few plain files in the repo to survive context resets: one for general rules, one for how we were working, and one for the current resume point. Starting a fresh session could be as simple as "read the notes and continue."

That sounds small, but it made the AI collaboration more consistent and made the work easier to pick up again later.

## How I checked the result

I did not trust "the tests passed" as a complete answer.

The backend suite was useful, and in the end I had 145 green specs, including a few new guard tests around upgrade-sensitive behavior. The frontend end-to-end suite also ended 18 out of 18 green.

But that was not enough by itself, because some of the biggest risks were exactly the kind of things an existing suite might miss.

So I used a written manual checklist on top of the automated tests. That checklist ended up with 74 browser checks. More importantly, I wrote it so either a person or an AI could run it without changing the standard. The operator could change. The checklist, the expected result, and the evidence rule did not.

Each workflow had a visible expected result. Each run used a throwaway database with known logins. Each step recorded what was actually seen, with screenshots as evidence. Any 500 or console error failed the test.

That mattered for another reason too: AI is too willing to say a test passed. Requiring visible evidence and explicit expectations made those runs much more trustworthy. It also made the checklist portable. A human tester and an AI browser run could both be judged against the same artifact.

That checklist probably deserves its own write-up. The important point here is that it turned manual testing from "click around and see" into a reusable test method.

This process paid off. It found real regressions that the spec suite had not exercised, including login failures, a broken calendar path, and crashing totals. It also made the testing itself better, because the manual run did not just find bugs. It told me exactly which missing specs to add. Some of the new regression tests in the upgraded suite exist only because the checklist caught those failures first.

## What I would do again

The main lesson for me was not "AI made me faster." That is true, but not very interesting.

The more useful lesson is that AI was best at cheap exploration:

- exploring unfamiliar code
- exploring framework changes
- exploring likely causes
- exploring candidate fixes
- exploring upgrade paths with a throwaway spike

The final answer still needed human ownership:

- judgment about what belonged in the upgrade and what did not
- ownership of the risky edits
- evidence that the app still behaved correctly
- a reviewable git history

That approach got me to a Rails 8 app with 145 green specs, 18 green end-to-end tests, a clean 74-check manual pass on the important workflows, and 36 small commits that a reviewer can actually read.

If I were doing another risky legacy upgrade with AI, I would keep the same rule:

Use AI for the exploring and the first draft. Keep the human responsible for the change.
