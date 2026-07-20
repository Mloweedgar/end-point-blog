---
author: Edgar Mlowe
title: "Give Every Task Its Own Environment with Git Worktree and Docker Compose"
description: "How a pile of post-upgrade user feedback led me to a simple pattern: one git worktree and one Docker Compose project per task, each with its own containers, ports, and database."
featured:
  endpoint: true
  image_url: /blog/2026/07/one-environment-per-task/cover.webp
date: 2026-07-20
tags:
- git
- docker
- containers
- tips
---

![Diagram titled "Give every task its own environment". One box labeled "one repo, one .git" connects to three folders made with git worktree: timesheet-25 on branch fix-25 at localhost:3100, timesheet-31 on branch fix-31 at localhost:3200, and the main timesheet checkout on develop at localhost:3000. Each folder has its own web, api, and db containers and its own database. A caption reads: separate containers, separate databases, switching tasks is just cd.](/blog/2026/07/one-environment-per-task/cover.webp)

<!-- Diagram created by Edgar Mlowe, 2026. -->

Not long ago we [upgraded our internal Timesheet app from Rails 5 to Rails 8](/blog/2026/06/upgrading-rails-5-to-rails-8-with-ai/). Once the new version was in front of real users, feedback started arriving. We went through all of it, confirmed what was real, and logged each item as an issue in our tracker. Soon I had a list: a login quirk, a report that didn't look right, a handful of small UI problems. Mostly independent fixes, with no reason to do them one after another.

I use AI assistants a lot, and Claude will happily fan out agents to work on several issues at once. But that alone wasn't the control I wanted. I want to review every diff before it goes anywhere. For some issues I want two different solutions so I can compare them. And I want to open the app and click through each fix by hand, at my own pace.

The real blocker wasn't the agents. It was my development setup: one checkout, one database, one port 3000. Every switch between issues meant stash, checkout, wrong schema, reset, re-seed, and about 20 minutes gone, most of it waiting and fixing state rather than thinking.

The fix I settled on is simple: give every task its own environment. Its own copy of the code, its own containers, and its own database. It's built from two features that have been around for years, git worktree and the Docker Compose project name. Nothing to install.

### The whole trick

```bash
git worktree add ../timesheet-25 -b fix-25 develop
cd ../timesheet-25
echo 'COMPOSE_PROJECT_NAME=timesheet-25' >> .env
docker compose up -d
```

That's all of it. You now have a second copy of the app, running next to the first one, with its own database.

It works because of how the two tools behave.

`git worktree` gives you a full checkout of the repo in another folder, sharing the same `.git` directory. No second clone, no stash. Each worktree can be on its own branch, and switching tasks is just `cd`.

Docker Compose namespaces everything by project name. Same `compose.yml`, different `COMPOSE_PROJECT_NAME`, and you get separate containers and separate named volumes. Separate volumes means each task has its own Postgres data. Migrations you run on one task never touch another.

The one thing to take care of is ports. Two stacks can't both publish port 3000, so each worktree needs its own. If your `compose.yml` reads ports from `.env`, like this:

```yaml
ports:
  - "127.0.0.1:${WEB_PORT:-3000}:3000"
```

then the `.env` in each worktree just sets different ones:

```bash
COMPOSE_PROJECT_NAME=timesheet-25
WEB_PORT=3100
API_PORT=3101
DB_PORT=55433
```

### How the feedback actually got fixed

Day to day it looked like this. Issue 25 ran on localhost:3100, issue 31 ran on localhost:3200, and my main checkout kept the normal ports. Each fix was a browser tab.

<!-- TODO(Edgar): screenshot here. Two browser tabs side by side, one on :3100 and one on :3200, showing different data. Save it as two-tabs.webp in the post folder and reference it like the cover image. This is the proof the post is lived, not imagined. -->

For the issues I handed to an agent, the agent got its own worktree. It could edit code, run migrations, and run tests in there, with no way to touch my code or my database. I kept working on the trickiest issue myself, next door. When an agent finished, I reviewed the diff, then opened that task's port and clicked through the fix by hand. A running copy of the app is a much better review tool than a diff alone.

For one stubborn issue I tried two solutions at once: two worktrees, one approach in each, both running side by side. I compared them, kept the better one, and deleted the other. Before this, trying two full implementations of the same thing felt too expensive. Now it's two folders.

<!-- TODO(Edgar): one sentence here about the real issue you solved twice. Which issue was it, and what were the two approaches? A specific, slightly messy detail here is worth more than any polish. -->

The same trick works for reviewing a colleague's merge request: check out their branch in a worktree, start its stack, and review by clicking through the running app instead of only reading the diff.

To pause a task, `docker compose stop` in its folder, and the data waits. When a fix was merged, `docker compose down -v` and `git worktree remove` cleaned it up completely.

None of this needs AI, though. If you never touch an assistant, it's still the fastest way I know to hold three tasks at once.

### The honest parts

Each new environment builds images and seeds its own database the first time, so the first `docker compose up` in a fresh worktree takes a few minutes. Docker's layer cache keeps later builds fast, and you can point all worktrees at one shared image tag if you want more. Disk usage grows with each Postgres volume, so remove environments when you're done. And if your compose file hardcodes ports, make them overridable first, which is a small, worthwhile change on its own.

I automated the boring parts, picking free ports, writing the `.env`, cleanup, into a small bash script. But the script isn't the point. The four commands above are the point, and they work in any project with a compose file. Next time a pile of small issues lands on you, give each one its own environment and see how it feels.
