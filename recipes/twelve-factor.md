---
num: "01"
title: "Twelve-Factor on a Homelab Budget"
desc: "Which of the twelve factors hold, bend, or get ignored when you are the entire ops team for a single-node homelab."
tags: ["Twelve-Factor", "Architecture", "Philosophy"]
color: purple
pubDate: 2026-05-17
---

## Context

The [Twelve-Factor App](https://12factor.net/) is the closest thing the industry has to a shared definition of "built right." It was written for SaaS running on cloud platforms with effectively unlimited resources and a team to operate them.

A homelab has one node, one operator, and a hardware budget that already feels indulgent. So the honest question isn't *"do we follow twelve-factor?"* — it's *"which factors actually earn their keep when you're the whole ops team?"*

The answer, mostly: more than you'd expect. Twelve-factor isn't really about scale. It's about *not being surprised by your own system.* That matters more in a homelab, not less — because when it breaks, there's no one else to page.

This recipe is the lens. The rest of the collection is the execution.

## The scorecard

| Factor | Verdict | Where it shows up |
|--------|---------|-------------------|
| 1. Codebase | **Holds** | [The CI/CD recipe](/homelab/github-actions-deploys) |
| 2. Dependencies | **Holds** | [The cluster-host recipe](/homelab/cluster-host) |
| 3. Config | **Holds** | [The secrets recipe](/homelab/1password-secrets) |
| 4. Backing services | **Holds** | [The PostgreSQL recipe](/homelab/postgres-and-backups) |
| 5. Build, release, run | **Holds** | [The CI/CD recipe](/homelab/github-actions-deploys) |
| 6. Stateless processes | **Holds** | [The PostgreSQL recipe](/homelab/postgres-and-backups) |
| 7. Port binding | **Holds** | [Cluster-host](/homelab/cluster-host) & [Cloudflare Tunnels](/homelab/cloudflare-tunnels) recipes |
| 8. Concurrency | **Ignored** (mostly) | — |
| 9. Disposability | **Holds — load-bearing** | [The cluster-host recipe](/homelab/cluster-host) |
| 10. Dev/prod parity | **Bends** | [The CI/CD recipe](/homelab/github-actions-deploys) |
| 11. Logs as streams | **Holds** | [The logging recipe](/homelab/openobserve-logging) |
| 12. Admin processes | **Holds** | [The PostgreSQL recipe](/homelab/postgres-and-backups) |

## The factors that hold

**Codebase, build/release/run.** One repo, one history, deploys built from commits. This is free — Git and a CI pipeline give it to you whether you ask or not. The discipline worth keeping: a deploy is a *build artifact plus config*, never "I SSH'd in and edited a file." The moment you hand-edit a running container, you've lost the ability to reproduce it. A CI pipeline keeps the artifact a tagged image and the config separate.

**Dependencies.** Declare them, don't assume them. On a homelab this mostly means: the host's job is to run a container runtime and nothing else. Your app's dependencies live *in the image*, not in whatever you happened to `brew install` last year. Keep the host boring on purpose.

**Config in the environment.** The single highest-leverage factor for a solo operator. Credentials never live in the image and never live in Git — they're injected at runtime. This is what lets you publish your manifests publicly without a second thought. Make a vault the source of truth and sync it into the cluster.

**Backing services as attached resources.** Your database is a thing your app *connects to*, identified by a connection string, not a thing your app *is*. Sounds obvious until you notice how easy it is to assume "localhost." Treat it as attached and you can move it, restore it, or point staging at a clone without touching app code.

Twelve-factor is deliberately silent on *who operates* that backing service — the same app should work against a local container or a managed cloud database and not be able to tell the difference. The homelab runs its own PostgreSQL. That's a fine answer to a question twelve-factor doesn't ask — but it sends an operational bill to your desk: the durability a managed database would have handled is now *your* job. State is allowed to exist; it is not allowed to exist in only one place. That's why off-site, tested backups are non-negotiable rather than a nice-to-have.

**Stateless processes.** Your app processes are stateless and share-nothing — no sticky sessions, nothing cached in memory that a later request depends on. Persistent data goes to the database, which is *exactly* what this factor prescribes. Running that database yourself doesn't bend this: twelve-factor governs your app *processes*, and it openly expects a stateful backing service to exist. Whose machine that service runs on is simply not a question this factor asks.

**Port binding.** Your app speaks HTTP on a port. It does not embed a web server's worth of opinions about TLS, hostnames, or routing — something in front does that. In this stack, Traefik and Cloudflare handle the front door. The app just binds a port and gets out of the way.

**Logs as event streams.** Write to stdout. Don't manage log files, don't rotate anything, don't ship anything yourself. Something else collects the stream and makes it searchable. The app's only job is to *talk*.

**Admin processes.** Migrations, backfills, one-off scripts — these run as discrete jobs against the same image and config as the app, not as code paths bolted into startup. A migration is an event with a beginning and an end, and you want to be able to point at the one that broke.

**Disposability.** This one is load-bearing, so it gets its own section.

## Disposability is the whole game

Twelve-factor says processes should start fast and shut down gracefully. In a homelab, disposability isn't a nice-to-have — it's the property that lets you sleep.

You have one node. It *will* go down: a macOS update, a power blip the UPS didn't fully cover, a kernel panic, a cat. The question is never "will it fail" — it's "how annoyed are you when it does."

If every piece of your system is disposable — the cluster can be recreated from a script, the pods are stateless, the data is backed up off-site, the config is in a vault — then a dead node is a *30-minute chore*, not a *disaster*. You rebuild and move on.

If any piece is a precious snowflake — a hand-configured VM, a database with no backups, a secret that exists only in one container's environment — then a dead node is the worst night of your month.

So these recipes are deliberately biased toward disposability. The cluster is built from a script you can re-run. Secrets live in a vault, not in pods. Backups live off the box entirely. None of this is for scale. It's so that "the Mac Mini died" is a sentence you can say calmly.

## The factor that bends

**Dev/prod parity (10).** True parity means dev, staging, and prod are near-identical. On a homelab, "dev" is a laptop and "prod" is a Mac Mini — they are not identical and pretending otherwise is a lie. What you *can* keep honest is **staging**: same cluster, same images, same manifests, real data cloned from prod. Staging is where parity is worth paying for; your laptop is not.

## The factor we ignore

**Concurrency (8)** — scaling out via the process model. You have one node. You will scale *up* (give a pod more memory) long before you scale *out*, and you will mostly not scale at all, because your traffic is "me, my family, and a few friends." Designing for horizontal scale here is solving a problem you don't have with complexity you'll definitely feel. If the homelab ever outgrows the homelab, that's a *wonderful* problem, and you can revisit this then. Until then: one replica, sized sensibly, is the right answer.

## The meta-rule: don't run infrastructure to support your infrastructure

The strongest pull in a homelab is toward *more*. Self-host the secret manager. Self-host the CI runners. Self-host the DNS, the registry, the monitoring of the monitoring. Each one is a fun weekend. Collectively they are a second full-time job, and they all fail at 3am.

So this stack leans on managed edges on purpose:

- **Cloudflare** is the front door — DNS, the tunnel, off-site backup storage.
- **GitHub** runs CI and hosts container images.
- **1Password** is the vault.
- **Tailscale** is the private network.

These are the load-bearing walls. Could you self-host all of them? Absolutely. Should you? Only if running that thing is the *point* — if it's the project you actually want. Otherwise every hour spent babysitting a self-hosted dependency is an hour not spent on the thing you built the homelab *for*.

A homelab should be mostly your stuff, standing on a few boring foundations you don't have to think about. That's the budget. Spend it on what you came here to build.
