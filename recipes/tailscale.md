---
num: "06"
title: "Private Access with Tailscale"
desc: "A private mesh network for SSH and admin UIs, with a Tailscale OAuth client set up the right way for ephemeral CI."
tags: ["Tailscale", "VPN", "WireGuard", "SSH"]
color: teal
pubDate: 2026-05-17
---

## Context

Some things should be on the public internet. Most things should not. The cluster's API, your SSH access, admin dashboards, the logging UI — none of that belongs behind a public hostname, no matter how good the password is. The attack surface you don't expose is the one you never have to defend.

Tailscale is the private side of the homelab. It's a mesh VPN that makes your devices — laptop, phone, the Mac Mini, even ephemeral CI runners — behave as if they're on one small, private network, no matter where they physically are. If a service is on the tailnet and nowhere else, then "who can reach it" has exactly one answer: devices you've personally approved.

## How it works, briefly

Tailscale builds a [WireGuard](https://www.wireguard.com/) mesh between your devices. Each device gets a stable address in the `100.x.x.x` range that follows it everywhere. Connections are end-to-end encrypted and, wherever possible, peer-to-peer — Tailscale's coordination server handles introductions, then gets out of the way.

Two features carry most of the weight:

- **MagicDNS** — devices are reachable by name (`homelab-server`) instead of by `100.x` address. You will use this constantly.
- **Tailscale SSH** — SSH access governed by tailnet policy instead of by managing `authorized_keys` files on every host.

The mental split for the whole homelab: **Cloudflare Tunnel is the public door, Tailscale is the private one.** A service is reachable by the public, or reachable by your tailnet, and you decide which on purpose — never by accident.

## Step 1 — Stand up the tailnet

Create a Tailscale account — that *is* your tailnet. Then install the client on each device you want on it:

```bash
# On the Mac Mini
brew install tailscale
sudo tailscale up
```

`tailscale up` prints a URL; open it, authenticate, and the device joins. Do the same on your laptop and phone (the app handles it there). Confirm the tailnet sees everyone:

```bash
tailscale status
```

Every device shows its `100.x` address and its MagicDNS name. From your laptop you can now reach the Mac Mini as `homelab-server` from anywhere in the world, and nothing about it is public.

## Step 2 — SSH over the tailnet

Enable Tailscale SSH on the server:

```bash
sudo tailscale up --ssh
```

Now SSH is gated by tailnet identity, not by key files:

```bash
ssh you@homelab-server
```

This is how you administer the box day to day. The Mac Mini's actual SSH port is never exposed to the internet, never port-forwarded, never in a `cloudflared` ingress rule. The only way in is to be on the tailnet, authenticated as you.

## Step 3 — Internal-only sites

This is where Tailscale and the rest of the stack meet.

The [TLS recipe](/homelab/tls-letsencrypt) established that an internal service can hold a real, browser-trusted certificate — DNS-01 issuance never needs to *reach* the site. Tailscale provides the other half: making the hostname resolve, but only for you.

For a service like `logs.otterpond.dev` that should be tailnet-only:

- **Do not** add a Cloudflare Tunnel route for it. No public ingress, no public path.
- **Do** point its DNS record at the Mac Mini's tailnet address (`100.x.x.x`). That address only routes for devices on your tailnet. To anyone else, the hostname resolves to an address they simply cannot reach.
- The wildcard certificate already covers it, so the browser shows a clean padlock.

The result: `logs.otterpond.dev` is a normal-looking HTTPS URL that works perfectly on your laptop and phone, and is invisible — not just locked, *invisible* — to everyone else. No warnings, no self-signed certificate, no compromise. That's the combination worth remembering: **DNS-01 gives internal sites real TLS; Tailscale gives them private reachability.**

## OAuth client setup (for automation)

Everything above is for *you* — interactive devices you log into. CI is different. A GitHub Actions runner needs to reach the cluster's API over the tailnet, but it's not a person, it spins up and dies in minutes, and you can't sit there clicking an auth URL for it.

The right tool is a **Tailscale OAuth client**. It lets automation join the tailnet as a short-lived, tagged, *ephemeral* device that cleans itself up when the job ends. The [CI/CD recipe](/homelab/github-actions-deploys) consumes this; setting it up is a Tailscale concern, so it lives here.

**1. Define a tag for CI devices.** In the admin console → **Access controls**, add a tag and declare who may apply it:

```json
{
  "tagOwners": {
    "tag:ci": ["autogroup:admin"]
  }
}
```

`tag:ci` is the identity every CI runner will join under. `autogroup:admin` as the owner means only admins can mint devices with that tag — a random OAuth client can't.

**2. Grant `tag:ci` access to the cluster.** A CI runner needs to reach the Mac Mini's cluster API. Add a grant for it:

```json
{
  "grants": [
    {
      "src": ["tag:ci"],
      "dst": ["autogroup:admin"],
      "ip":  ["*"]
    }
  ]
}
```

`dst: ["autogroup:admin"]` means "devices owned by an admin user" — in a one-person homelab, that's the Mac Mini plus your own laptop and phone. The Mac Mini is the one that matters here; it's where the cluster API lives. The grant is still bounded where it counts: the *source* is `tag:ci` and nothing else, so the only thing that can open this door is a CI runner you minted, and it can only reach your own devices — not the rest of the tailnet, and nothing off it.

You can scope it tighter — `dst` to a specific host, `ip` to `["tcp:6550"]` instead of `["*"]` — but for a single-operator homelab the broader grant is a fair simplification: it keeps working when you rename a host or add a second machine, and the `tag:ci` source restriction is doing the real security work.

**3. Create the OAuth client.** Admin console → **Settings → OAuth clients → Generate OAuth client**.

Worth understanding what this client does before you fill in the form: it never joins the tailnet itself. On each CI run it mints a fresh, single-use auth key, and the runner joins with *that*. So it needs exactly one capability — writing auth keys.

- **Scope:** under **Auth Keys**, check **Write**. Tailscale ticks **Read** alongside it automatically — expected, leave it. Everything else on the form stays unchecked; the runner mints one key and joins, and that's the whole job.
- **Tag:** the moment you check **Write**, a **Tags** field appears beneath it, marked *required for write scope*. Add `tag:ci`. This isn't a formality — it's the safety rail. Every auth key this client mints, and every device that joins with one, is stamped `tag:ci` and bound by the grant from step 2. An OAuth client that could mint *untagged* keys would be a skeleton key to your whole tailnet, so Tailscale doesn't let you make one.

One scope, one tag — that's the entire client.

Ephemeral cleanup comes free: devices that authenticate through an OAuth client are ephemeral by default, no extra setting. A runner joins for the length of the job and removes itself when the job ends — so you're never left with a graveyard of dead CI devices cluttering your device list.

**4. Save the credentials** — a Client ID and a Client Secret. These go straight into your vault (the [secrets recipe](/homelab/1password-secrets)). They are *not* pasted into a terminal and *not* committed anywhere. The [CI/CD recipe](/homelab/github-actions-deploys) wires them into GitHub Actions.

> Why an OAuth client and not a plain auth key? Auth keys are long-lived static secrets — leak one and someone joins your tailnet until you notice. An OAuth client mints fresh, tagged, ephemeral credentials per use, governed by the ACL above. For anything automated, it's the only sane choice.

## When it breaks

Tailscale is reliable, but it sits underneath everything else, so when it hiccups it's worth fixing fast and decisively.

### Symptom: a device shows offline, or won't connect

The reset that fixes the large majority of Tailscale weirdness — stale routes, a confused client after a network change, a device that *says* it's up but isn't passing traffic:

```bash
tailscale down
tailscale up --reset
```

`--reset` clears accumulated client state and brings the connection up clean. It's the Tailscale equivalent of turning it off and on again, and like that classic, it works far more often than it has any right to. Reach for it first. The [alerts recipe](/homelab/openobserve-alerts) probes `BackendState`, `Self.Online`, and a real peer ping — not just whether `tailscaled` is running — and this reset is the first heal.

### Symptom: you can't reach the Mac Mini by its name

If `homelab-server` won't resolve or won't connect but you genuinely need into the box — keep a fallback ready: **reach it by its LAN IP.** When you're on the same physical network as the Mac Mini, its plain local-network address (e.g. `192.168.x.x`) works regardless of tailnet health:

```bash
ssh you@homelab-server          # the normal path
ssh you@192.168.x.x             # fallback when the tailnet is being difficult
```

This is worth knowing *before* you need it. The failure case is "I'm remoting in to fix the homelab and the tool I use to remote in is the thing that's broken." Having the LAN IP written down — and knowing the tailnet is a convenience layer on top of a network that still works without it — turns a panic into a minor annoyance. Once you're on the box by LAN IP, run the reset above to fix the tailnet itself.

### Symptom: CI can't reach the cluster

If a CI run fails to join the tailnet or can't reach the API, it's almost always one of three things, in this order of likelihood:

1. The OAuth client lost its `tag:ci` tag, so the runner can't tag itself. Check the OAuth client config.
2. The ACL grant doesn't permit `tag:ci` to reach the cluster. Check the `grants` block.
3. The OAuth credentials in GitHub Actions are stale or wrong. Re-sync them from the vault.

A runner that briefly appears in your device list tagged `tag:ci` and then vanishes is *correct* — that's ephemeral cleanup, not a bug.
