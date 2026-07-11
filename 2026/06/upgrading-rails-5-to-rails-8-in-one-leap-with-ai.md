---
author: Edgar Mlowe
title: "Upgrading Rails 5 to Rails 8 in One Leap with AI: What Broke After the Suite Went Green"
description: "A first-hand Rails 5.0 to 8.1 upgrade done as a fresh-app transplant: what broke once the specs were green, why the tests missed it, and how we checked the app really still worked."
featured:
  endpoint: true
  image_url: /blog/2026/06/upgrading-rails-5-to-rails-8-in-one-leap-with-ai/cover.webp
date: 2026-06-23
tags:
- artificial-intelligence
- ruby
- rails
- open-source
- troubleshooting
---

![An editorial illustration of an engineer crossing from a tangled legacy software landscape toward a cleaner modern system, with AI present as a small guide rather than the driver.](/blog/2026/06/upgrading-rails-5-to-rails-8-in-one-leap-with-ai/cover.webp)

<!-- Illustration created with AI direction by Edgar Mlowe, 2026. -->

We moved our internal Timesheet app from Ruby 2.4.10 and Rails 5.0.1 to Ruby 3.3.6 and Rails 8.1.3. We did it as a fresh-app transplant: build a clean Rails 8 app, move the old code into it, and get it running. The backend spec suite went green, 145 examples. The frontend end-to-end suite passed, 18 of 18. `bin/rails zeitwerk:check` loaded the whole app without complaint.

The app was still broken. Every request behind login returned a 500. The calendar failed. Monthly totals crashed. None of the green tests noticed. Running the real app in a browser did.

This post is about that gap. What broke in a direct Rails 5-to-8 transplant, why a green suite missed it, and how we established that the app actually still behaved the same. It tracks hours, billing, reports, users, roles, and permissions, so quiet wrong answers matter more than loud crashes.

## Why a transplant instead of an incremental upgrade

The standard advice is to upgrade one version at a time. The official Rails guides recommend it, and so does most consulting experience. We did not do that here, and it was a project-specific call, not a claim that transplants are safer in general.

The app was three major Rails versions behind, with many release series in between, and both Ruby 2.4.10 and Rails 5.0.1 had been end-of-life for years. The original developers were gone. A step-by-step path meant carrying old configuration and dead idioms through every intermediate upgrade. A transplant meant one larger jump into a clean Rails 8 baseline, at the cost of hitting every incompatibility at once.

We ran a throwaway spike first: fresh Rails 8 app, move the code in, see what breaks, then throw it away. That gave us a risk map before any real work started. The risk we cared about was silent behavior change in the areas that matter for a timesheet: hours, billing, reports, permissions, dates, and raw SQL.

A rough rule for choosing. A transplant is worth considering when the app is far behind, the intermediate versions hold no value to you, and you can afford to verify behavior end to end. Incremental is the right default when you are one or two versions behind, need to keep shipping, or cannot fully re-verify the app afterward.

## What actually broke

Here is what broke, and why the automated suite did not catch it.

| Symptom | Root cause | Why the tests missed it | How it was found | Permanent guard |
| --- | --- | --- | --- | --- |
| Every logged-in request returned 500 | Dev config used a plain `Logger`. `activerecord-session_store` calls `#silence` on the AR logger on every session lookup, and a plain `Logger` does not provide it. `ActiveSupport::Logger` does. | The fault was in development-environment config. Specs run in the test env with a different logger. | Logging in to the running app | None automated. Checklist LOGIN tests re-check it. |
| Calendar pages 500'd | `to_s(:db)` was renamed to `to_fs(:db)` | `WorkDay.db_days` had no spec | Browser workflow check | `previous_next returns 200` guards a sibling `to_fs` caller. The calendar caller still has none. |
| Monthly and billing totals crashed | `Enumerable#sum` now starts from `0`, so `0 + Interval` raised. Fixed with an explicit identity, `sum(Interval.new)`. | The totals code had no request spec | Browser check of report totals | None automated. Checklist REP-14 re-checks that rows sum to the total. |
| Two reports 500'd | Rails 8 sent the `?` array bind as `text[]`, so `array[?] && role_ids` became `text[] && integer[]`, an operator Postgres does not define. Fixed with `::int[]`. | One query had a spec and failed there. The second was in a report with none. | Existing suite for the first, browser check for the second | Existing suite covers the role-id query. The report query has none. |
| Chart search 500'd | Rails blocks raw SQL fragments in `order`. Wrapped the known-safe expression in `Arel.sql`. | No spec exercised that ordering | Browser search of charts | None automated |
| Three React admin pages rendered a raw object string, weeks later | `yajl-ruby` had quietly patched `JSON.dump`. Removing the gem broke `render json: JSON.dump(relation)`. Fixed with `render json: relation`. | Those admin endpoints had no specs, and the patch kept them green | The pages broke after the gem was removed | None as a unit spec. The 18/18 e2e run confirmed the fix. |

One note on that raw-SQL fix. `Arel.sql` does not sanitize anything. It marks a fragment as trusted. It was safe here only because the search phrase is reduced to word characters before it reaches the query.

The login failure is the one to sit with. The fix was one word:

```ruby
- config.active_record.logger = Logger.new(File.join(Rails.root, 'log', 'activerecord.log'))
+ config.active_record.logger = ActiveSupport::Logger.new(File.join(Rails.root, 'log', 'activerecord.log'))
```

A plain `Logger` no longer carries the `#silence` method that `activerecord-session_store` calls on every session lookup. `ActiveSupport::Logger` does. Every logged-in request 500'd until that line changed. No backend spec caught it, because the fault lived in development-environment config that the test suite never loads. Running the app caught it in seconds.

## What each layer of testing actually proved

The checks were not one thing. Each layer proved something different, and it helps to keep them apart.

**`zeitwerk:check`** eager-loads the application's configured code paths. It surfaces naming and load-time problems at startup instead of on the first request that reaches them. A clean run proves the configured code can load. It does not prove any workflow behaves correctly. Ours passed while the app was still broken behind login. Loading is the floor, not the finish line.

**Backend specs.** 145 RSpec examples, green, including three new guard tests written during the upgrade. They covered the model and request behavior that already had specs. They did not cover the paths that never had specs, which is exactly where the runtime 500s lived: the dev logger, the calendar, the PDF builder, and billing.

**End-to-end tests.** 18 Playwright tests against the running stack, green. Useful, but they exercise a fixed set of flows, not every important screen.

**Manual browser checklist.** 74 checks, run top to bottom against a disposable seeded database. This is the layer that found the login 500, the broken calendar, and the crashing totals. It is a written method, not exploratory clicking. Two rules make it trustworthy: every check states a visible expected result, and every result needs evidence, meaning what was seen plus a screenshot. Any 500 or console error is an automatic fail. The operator can be a person or an AI driving the browser. The expected result and the evidence rule do not change.

A few checks, in the shape they are written:

- **LOGIN-01**: log in with valid credentials. Expect: lands on the calendar, nav shows the logged-in email. Evidence: screenshot.
- **AUTH-01**: as a non-admin, open an admin URL directly. Expect: access refused, the roles editor does not render. Evidence: screenshot.
- **REP-14**: on a generated report, compare the row values to the grand total. Expect: the rows sum to the shown total. Evidence: screenshot.
- **XC-02**: across the whole run, watch for any 5xx response or red console error. Expect: none, and list every one seen.

The checklist ships in the repo at `testing/checklist.md`, so a human run and an AI browser run are judged against the same artifact. Several of the new backend guard tests exist only because a checklist run found the failure first.

## Regression, or already broken?

During manual testing I found failures that would have been easy to just fix on the branch. Before fixing, I checked the same path on the tagged Rails 5 version. In four cases it failed there too. Those were real bugs, but they were not upgrade regressions. I kept them off the upgrade branch and filed them on their own, so the diff stayed about the upgrade and nothing else.

That distinction, upgrade regression versus pre-existing bug versus unsupported assumption versus missing coverage, is what kept the branch reviewable. A reviewer can read the 29 commits and see only upgrade work.

## Where AI helped, and where it stopped

I used AI heavily, but the useful part was not that it wrote code. It was that it made exploring unfamiliar, obsolete code cheap. The pattern was always the same. It proposed, I verified, and nothing merged without evidence.

- **Removed idioms.** The app registered a custom PostgreSQL `cron_spec` type through `alias_method_chain`, which no longer exists. AI explained the old idiom and drafted the modern form. I confirmed the type still registered before keeping it.

```ruby
# Rails 8: prepend + super. The Rails 5 version did the same job with alias_method_chain, now removed.
module CronSpecTypeRegistration
  def initialize_type_map(mapping = type_map)
    super
    oid = select_value("SELECT oid FROM pg_type WHERE typname = 'cron_spec'")
    if oid
      mapping.register_type(
        oid.to_i,
        ActiveRecord::ConnectionAdapters::PostgreSQL::OID::SpecializedString.new(:cron_spec)
      )
    end
  end
end
ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.prepend(CronSpecTypeRegistration)
```

- **Old behavior.** It explained why `to_s(:db)`, `sum` without a starting value, and untyped array binds behave differently now. I confirmed each against the running app, not against the explanation.
- **Strategy.** I used it to compare transplant versus incremental and to run a pre-mortem before committing. I made the decision.

I also kept a few plain files in the repo: the working rules, how we were collaborating, and the current resume point. A fresh session could start with "read the notes and continue," which kept the work easy to pick back up.

The one time I trusted an assumption instead of testing it, it cost me. Early on I kept an old JSON library, `yajl-ruby`, because I assumed the API endpoints needed it. They did not. It was quietly patching `JSON.dump` so several admin endpoints serialized their query results as real arrays, and the suite stayed green the whole time. Weeks later I removed the gem, and three React admin pages started rendering a raw object string instead of a list. The fix was to stop leaning on the patched `JSON.dump` and let Rails encode the response with `render json: relation`. The lesson matched everything else here. An untested assumption is not a safe one, however green the suite looks.

## A method for risky migrations

If I did another upgrade like this, I would keep the same short method:

- Reproduce a failure on the old version before calling it a regression.
- Treat load checks, unit tests, end-to-end tests, and browser checks as four separate kinds of evidence, not one.
- Turn every manually found regression into an automated guard where the path can hold one.
- Require visible evidence for browser checks: an expected result and a screenshot, not "looks fine."
- Keep the commit history small and readable, so a reviewer can follow the upgrade one change at a time.

AI made the exploration cheap. The evidence is what made the result trustworthy.
