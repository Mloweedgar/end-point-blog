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

When we started getting our internal Timesheet app ready to go open source, one problem dominated the work: the backend was still on Ruby 2.4.10 and Rails 5.0.1.

The app still worked. That was not the issue. The issue was that both versions had been end-of-life for years. It tracks hours, billing, reports, users, roles, and permissions. Quiet mistakes matter more than loud crashes in software like that.

I used AI heavily during the upgrade. But the useful part was not "AI wrote the code." The useful part was simpler: AI made exploration cheap. I still had to own the risky decisions, the risky edits, and the proof that the app still behaved correctly. The real question is never whether the tool can generate code — it clearly can — but who owns correctness when the code matters.

## The rule that kept this safe

My rule from the start was simple:

> AI helps with the exploring and the first draft. I own every change.

I did not want AI deciding what was safe to merge. I wanted it helping me understand old code, explain removed Rails behavior, suggest likely fixes, and argue trade-offs. If a change touched hours, money, reports, or anything else that could fail quietly, I needed to understand it well enough to explain it in plain English before I trusted it.

## Why this upgrade was a good test

The app was about eight major Rails versions behind: Ruby 2.4.10 and Rails 5.0.1, both years past end-of-life, up to Ruby 3.3.6 and Rails 8.1.3. The original developers were no longer here, and the codebase was not small.

So the job was not just "get it to boot on Rails 8." It was:

- move off unsupported Ruby and Rails versions
- preserve behavior in code that handles hours and money
- keep the diff reviewable
- learn enough of the system that we can maintain it in the open later

That last part mattered. Open-sourcing a project you do not understand is not much better than keeping it on obsolete dependencies.

## Where AI actually helped

I found AI genuinely useful in four places.

### Understanding code I did not write

Before I touched a file, I often needed a quick, grounded explanation of what it did, what depended on it, and what would be risky to change.

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

I did not want AI choosing that for me. I used it to compare the options, but I made the decision myself, with a simple rhythm: zoom out to define the real problem and compare options, zoom in to plan the work, then zoom out again for a pre-mortem.

For this app, I chose a transplant.

Being eight major versions behind changes the math. A step-by-step path means carrying old baggage through many sequential upgrades. A transplant means a bigger leap up front, but into a cleaner modern baseline.

The obvious risk was silent regressions in hours, money, date behavior, or SQL ordering. That was the risk I cared about most. So instead of pretending it was small, I designed the checking around it.

I also ran a throwaway spike first: build a fresh Rails 8 app, move the old code in, see what breaks, and treat that result as a risk map rather than final work. AI made that exploration cheap. The real upgrade still had to be done carefully by hand.

But "see what breaks" needs a definition. Mine was one command: `bin/rails zeitwerk:check`. It forces every file in the app to load at once, so a renamed class or a removed method fails at startup instead of hiding until some request hits it.

The principle behind it:

boot all at once, verify a piece at a time.

You cannot run half the app on Rails 5 and half on Rails 8. Either the whole thing starts up or it does not. All the small, area-by-area checking only counts once it does.

Same catch as everywhere else: a clean start proves the code loads, not that it works. The check passed and the suite was green while the app was still broken behind login. Loading is the floor, not the finish line.

A good example was an old initializer that registered custom PostgreSQL types. It looked like exactly the kind of strange legacy patch you want to delete during a modernization pass. But zooming out first changed the question from "can I remove this?" to "what behavior is this preserving?" The answer was: interval parsing the app still relied on, and a custom `cron_spec` type used by scheduling fields. That led me to port it carefully instead of deleting it blindly.

## A few small fixes that show the real work

A lot of the upgrade was not glamorous. It was small, exact changes that preserved intent while updating the mechanism.

One example was the development logger:

```ruby
- config.logger = Logger.new(path)
+ config.logger = ActiveSupport::Logger.new(path)
```

That one-word change mattered because a plain `Logger` no longer has the `#silence` method that `activerecord-session_store` calls on every session lookup; on Rails 8 only `ActiveSupport::Logger` still defines it. The result was that every logged-in request returned a 500. No backend spec caught it. Running the real app did.

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

- **SQL array comparisons needed explicit `::int[]` casts.** Rails 8 sends `?` array binds as `text[]`, so a query like `array[?] && role_ids` became `text[] && integer[]` — an operator Postgres does not have, which meant a 500. This one bit twice, in two different reports.
- **A raw SQL `order(...)` needed `Arel.sql`.** Rails blocks unsafe raw SQL in `order`, so `order(ts_rank(...))` had to be wrapped in `Arel.sql` (safe here because the search phrase is reduced to word characters first).

The custom PostgreSQL type I mentioned earlier is a good example of the mechanical side of the same work. Porting it meant swapping a Rails idiom that no longer exists for the modern one: `alias_method_chain` for `prepend` + `super`.

```ruby
# Rails 5: alias_method_chain, removed in modern Rails
ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.class_eval do
  def initialize_type_map_with_postgres_oids(mapping)
    initialize_type_map_without_postgres_oids(mapping)
    # ...register the cron_spec OID...
  end
  alias_method_chain :initialize_type_map, :postgres_oids
end
```

```ruby
# Rails 8: prepend + super
module CustomPostgresTypes
  def initialize_type_map(mapping = type_map)
    super
    oid = select_value("SELECT oid FROM pg_type WHERE typname = 'cron_spec'")
    # ...register the cron_spec OID...
  end
end
ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.prepend(CustomPostgresTypes)
```

None of those are exciting alone. Together, they are what a real upgrade looks like.

## The working rules mattered as much as the tool

The most useful thing I wrote down was not code. It was the working rules: small steps then stop and check, explain before doing, ask before risky actions, and when I wanted to own a change, hand me the diff to apply myself. I kept a few plain files in the repo — general rules, how we were working, and the current resume point — so a fresh session could start with "read the notes and continue." Small, but it kept the collaboration consistent and easy to pick back up.

## How I checked the result

I did not trust "the tests passed" as a complete answer.

The backend suite was useful, and in the end I had 145 green specs, including a few new guard tests around upgrade-sensitive behavior. The frontend end-to-end suite also ended 18 out of 18 green.

But that was not enough by itself, because some of the biggest risks were exactly the kind of things an existing suite might miss.

So I used a written manual checklist on top of the automated tests. That checklist ended up with 74 browser checks. More importantly, I wrote it so either a person or an AI could run it without changing the standard. The operator could change. The checklist, the expected result, and the evidence rule did not.

Each workflow had a visible expected result. Each run used a throwaway database with known logins. Each step recorded what was actually seen, with screenshots as evidence. Any 500 or console error failed the test.

That mattered for another reason too: AI is too willing to say a test passed. Requiring visible evidence and explicit expectations made those runs much more trustworthy. It also made the checklist portable. A human tester and an AI browser run could both be judged against the same artifact.

That checklist probably deserves its own write-up. The important point here is that it turned manual testing from "click around and see" into a reusable test method.

This process paid off. It found real regressions that the spec suite had not exercised, including login failures, a broken calendar path, and crashing totals. It also made the testing itself better, because the manual run did not just find bugs. It told me exactly which missing specs to add. Some of the new regression tests in the upgraded suite exist only because the checklist caught those failures first.

One mistake is worth admitting, because it shows how easily this still bites. Early on I kept an old JSON library, `yajl-ruby`, because I assumed the API endpoints needed it. They did not. It was quietly patching `JSON.dump` so a few admin endpoints serialized their query results as real arrays, and the suite stayed green the entire time. The assumption only broke weeks later, when I removed the gem and three React admin pages started rendering a raw object string instead of a list. The fix was to stop relying on the patched `JSON.dump` and let Rails encode the response directly. The lesson was the same as everywhere else: an untested assumption is not a safe one, however green the suite looks.

## What I would do again

The main lesson was not "AI made me faster." That is true, but not very interesting.

The more useful lesson is where the line fell. AI was best at cheap exploration — unfamiliar code, framework changes, likely causes, candidate fixes, upgrade paths I could throw away. The final answer still needed a human — judgment about what belonged in the upgrade, ownership of the risky edits, evidence that the app still behaved correctly, and a git history a reviewer can read.

That approach got me to a Rails 8 app with 145 green specs, 18 green end-to-end tests, a clean 74-check manual pass on the important workflows, and 29 small commits that a reviewer can actually read.

If I were doing another risky legacy upgrade with AI, I would keep the same rule:

Use AI for the exploring and the first draft. Keep the human responsible for the change.
