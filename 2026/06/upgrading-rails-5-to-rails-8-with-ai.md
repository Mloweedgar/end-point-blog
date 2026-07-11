---
author: Edgar Mlowe
title: "Upgrading Rails 5 to Rails 8 with AI: What Broke After 145 Specs Went Green"
description: "A first-hand AI-assisted Rails 5.0 to Rails 8.1 fresh-app transplant: what broke after 145 specs passed, why the tests missed it, and how we verified the application's highest-risk workflows."
featured:
  endpoint: true
  image_url: /blog/2026/06/upgrading-rails-5-to-rails-8-with-ai/cover.webp
date: 2026-06-23
tags:
- artificial-intelligence
- ruby
- rails
- open-source
- troubleshooting
---

![An editorial illustration of an engineer crossing from a tangled legacy software landscape toward a cleaner modern system, with AI present as a small guide rather than the driver.](/blog/2026/06/upgrading-rails-5-to-rails-8-with-ai/cover.webp)

<!-- Illustration created with AI direction by Edgar Mlowe, 2026. -->

We moved our internal Timesheet app, a legacy Rails application, from Ruby 2.4.10 and Rails 5.0.1 to Ruby 3.3.6 and Rails 8.1.3, with AI assisting the work throughout. We did it as a fresh-app transplant: build a clean Rails 8 app, move the old code into it, and get it running. The backend spec suite went green, 145 examples. The frontend end-to-end suite passed, 18 of 18. `bin/rails zeitwerk:check` eager-loaded the configured application code without complaint.

The app was still broken. Every request behind login returned a 500. The calendar failed. Monthly totals crashed. None of the green tests noticed. Running the real app in a browser did.

This post is about that gap. What broke in a direct Rails 5 to Rails 8 transplant, why a green suite missed it, and how we verified the application's highest-risk workflows after the migration. The app tracks hours, billing, reports, users, roles, and permissions, so quiet wrong answers matter more than loud crashes.

## Why a fresh-app transplant instead of an incremental upgrade

The standard advice is to upgrade one version at a time. The official Rails guides recommend it, and so does most consulting experience. We did not do that here, and it was a project-specific call, not a claim that transplants are safer in general.

The app was three major Rails versions behind, with many release series in between, and both Ruby 2.4.10 and Rails 5.0.1 had been end-of-life for years. The original developers were gone. A step-by-step path meant carrying old configuration and dead idioms through every intermediate upgrade. A transplant meant one larger jump into a clean Rails 8 baseline, at the cost of hitting every incompatibility at once.

We ran a throwaway spike first: fresh Rails 8 app, move the code in, see what breaks, then throw it away. That gave us a risk map before any real work started. The risk we cared about was silent behavior change in the areas that matter for a timesheet: hours, billing, reports, permissions, dates, and raw SQL.

A rough rule for choosing. A transplant is worth considering when the app is far behind, the intermediate versions hold no value to you, and you can afford to verify behavior end to end. Incremental is the right default when you are one or two versions behind, need to keep shipping, or cannot fully re-verify the app afterward.

## What actually broke

Here are the migration regressions, and why the automated suite did not catch them.

| Symptom | Root cause | Why the tests missed it | How it was found |
| --- | --- | --- | --- |
| Every logged-in request returned 500 | Dev config used a plain `Logger`. `activerecord-session_store` calls `#silence` on the Active Record logger on every session lookup, and a plain `Logger` does not provide it. `ActiveSupport::Logger` does. | The fault was in development-environment config. Specs run in the test env with a different logger. | Logging in to the running app |
| Calendar pages 500'd | `to_s(:db)` was renamed to `to_fs(:db)` | The calendar date helper had no spec of its own | Browser workflow check |
| Monthly and billing totals crashed | The totals code sums interval values with `Enumerable#sum`, and on the upgraded stack the sum began with integer `0`, producing `0 + Interval` and a TypeError. Fixed by supplying an explicit interval identity to `sum`. | The code that calculates the totals had no request spec | Browser check of report totals |
| Two reports 500'd | In the upgraded stack, these queries' array parameters were typed as `text[]` while the stored columns were `integer[]`, and PostgreSQL has no overlap operator for that pair. Fixed by casting the parameters explicitly with `::int[]`. | One query had a spec and failed there. The second was in a report path with none. | Existing suite for the first, browser check for the second |
| The project search 500'd | Rails blocks raw SQL fragments in `order`. The results were ranked with a raw `ts_rank(...)` expression, now wrapped in `Arel.sql`. | No spec exercised that ordering | Browser search check |
| Three React admin pages rendered a raw object string, weeks later | `yajl-ruby` had quietly patched `JSON.dump`. Removing the gem broke `render json: JSON.dump(relation)`. Fixed with `render json: relation`. | The one spec that hit these endpoints checked the status code, not the body. A 200 with a garbage string still passed. | The pages broke after the gem was removed |

One note on that raw-SQL fix. `Arel.sql` does not sanitize anything. It marks a fragment as trusted. It was safe here only because the search phrase is reduced to word characters before it reaches the query.

The login failure is the one to sit with. The fix was one word:

```ruby
- config.active_record.logger = Logger.new(File.join(Rails.root, 'log', 'activerecord.log'))
+ config.active_record.logger = ActiveSupport::Logger.new(File.join(Rails.root, 'log', 'activerecord.log'))
```

The plain `Logger` configured here did not provide the logger-silencing interface that `activerecord-session_store` expects when it looks up a session. `ActiveSupport::Logger` did. Every logged-in request 500'd until that line changed. No backend spec caught it, because the fault lived in development-environment config that the test suite never loads. Running the app caught it in seconds.

## What each layer of testing actually proved

The checks were not one thing. Each layer proved something different, and it helps to keep them apart.

**`zeitwerk:check`** eager-loads the application's configured code paths. It surfaces naming and load-time problems at startup instead of on the first request that reaches them. A clean run proves the configured code can load. It does not prove any workflow behaves correctly. Ours passed while the app was still broken behind login. Loading is the floor, not the finish line.

**Backend specs.** 145 RSpec examples, green, including three new guard tests written during the upgrade. They covered the model and request behavior that already had specs. They did not cover the paths that never had specs, which is exactly where the runtime 500s lived: the dev logger, the calendar, the PDF builder, and billing.

**End-to-end tests.** 18 Playwright tests against the running stack, green. Useful, but they exercise a fixed set of flows, not every important screen.

**Manual browser checklist.** 74 checks, run top to bottom against a disposable seeded database. This is the layer that found the login 500, the broken calendar, and the crashing totals. It is a written method, not exploratory clicking. Two rules make it trustworthy: every check states a visible expected result, and every result needs evidence, meaning what was seen plus a screenshot. Any 500 or console error is an automatic fail. The operator can be a person or an AI driving the browser. The expected result and the evidence rule do not change.

A few checks, in the shape they are written:

- **Valid login**: log in with valid credentials. Expect: lands on the calendar, nav shows the logged-in email. Evidence: screenshot.
- **Permission wall**: as a non-admin, open an admin URL directly. Expect: access refused, the admin page does not render. Evidence: screenshot.
- **Report totals**: on a generated report, compare the row values to the grand total. Expect: the rows sum to the shown total. Evidence: screenshot.
- **Error roundup**: across the whole run, watch for any 5xx response or red console error. Expect: none, and list every one seen.

The checklist lives in the repository as a plain Markdown file, so a human run and an AI browser run are judged against the same artifact. Several of the new backend guard tests exist only because a checklist run found the failure first.

## Regression, or already broken?

During manual testing I found failures that would have been easy to just fix on the branch. Before fixing, I checked the same path on the tagged Rails 5 version. In four cases it failed there too. Those were real bugs, but they were not upgrade regressions. I kept them off the upgrade branch and filed them on their own, so the diff stayed about the upgrade and nothing else.

That distinction, upgrade regression versus pre-existing bug versus unsupported assumption versus missing coverage, is what kept the branch reviewable. A reviewer can read the 29 commits and see only upgrade work.

## Where AI helped, and where it stopped

I used AI heavily, but the useful part was not that it wrote code. It was that it made exploring unfamiliar, obsolete code cheap. The pattern was always the same. It proposed, I verified, and nothing merged without evidence.

- **Removed idioms.** One initializer extended Rails' PostgreSQL type map for the application's custom scheduling type using `alias_method_chain`, an old Rails extension pattern that no longer exists. AI explained the old idiom and drafted the replacement, a module hooked in with `prepend` and `super`. I kept the change only after confirming the scheduling fields still loaded, saved, and returned correctly.
- **Old behavior.** It explained why `to_s(:db)`, `sum` without a starting value, and untyped array binds behaved differently on the upgraded stack. I confirmed each against the running app, not against the explanation.
- **Strategy.** I used it to compare transplant versus incremental and to run a pre-mortem before committing. I made the decision.

I also kept a few plain files in the repo: the working rules, how we were collaborating, and the current resume point. A fresh session could start with "read the notes and continue," which kept the work easy to pick back up.

The one time I trusted an assumption instead of testing it, it cost me. Early on I kept an old JSON library, `yajl-ruby`, because I assumed the API endpoints needed it. They did not. It was quietly patching `JSON.dump` so several admin endpoints serialized their query results as real arrays, and the suite stayed green the whole time. Weeks later I removed the gem, and three React admin pages started rendering a raw object string instead of a list. The fix was to stop leaning on the patched `JSON.dump` and let Rails encode the response with `render json: relation`. The lesson matched everything else here. An untested assumption is not a safe one, however green the suite looks.

## A method for risky Rails migrations

If I did another AI-assisted Rails upgrade like this, I would keep the same short method:

- Reproduce a failure on the old version before calling it a regression.
- Treat load checks, unit tests, end-to-end tests, and browser checks as four separate kinds of evidence, not one.
- Turn every manually found regression into an automated test.
- Require visible evidence for browser checks: an expected result and a screenshot, not "looks fine."
- Keep the commit history small and readable, so a reviewer can follow the upgrade one change at a time.

AI made the exploration cheap. The evidence is what made the result trustworthy.
