---
num: "05"
title: "Public Ingress with Cloudflare Tunnels"
desc: "Serving public sites from home with no port forwarding and no exposed IP, plus how to add a new site to the tunnel."
tags: ["Cloudflare", "Tunnels", "Ingress"]
color: warn
pubDate: 2026-05-17
---

## Context

You have services on a Mac Mini in your house, and you want some of them reachable from the public internet. The traditional answer — port-forward 80 and 443 on your router to the server — works, and it's also a small pile of problems: you're advertising your home IP, you've punched holes in your firewall, your IP changes so you need dynamic DNS, and none of it works at all behind the CGNAT of a cellular backup link.

A **Cloudflare Tunnel** sidesteps every one of those. It's the front door for everything public in this homelab, and it's the reason the network can fail over to a cheap cellular connection without anyone noticing.

## How a tunnel works

The trick is direction. A tunnel doesn't wait for the internet to connect *in* — it connects *out*.

A small daemon called `cloudflared` runs on the Mac Mini and opens an **outbound** connection up to Cloudflare's edge network. It stays connected. When someone requests `api.otterpond.dev`:

1. DNS resolves the hostname to Cloudflare (not to your house).
2. The request hits Cloudflare's edge.
3. Cloudflare sends it *down the tunnel* that `cloudflared` already has open.
4. `cloudflared` hands it to a local service on the Mac Mini.
5. The response goes back the same way.

What this buys you:

- **No inbound ports.** Your firewall stays closed. There is nothing to port-forward and nothing to scan.
- **Your home IP is never exposed.** The public only ever sees Cloudflare.
- **No dynamic DNS.** Hostnames point at Cloudflare permanently. Your home IP can change — or fail over to cellular — and nothing needs reconfiguring, because nothing was ever pointed at it.
- **TLS for free.** TLS terminates at Cloudflare's edge. Public hostnames get valid certificates without a cluster-side certificate at all.
- **DDoS absorption and caching.** You get Cloudflare's edge in front of a home server, which is a frankly absurd amount of capacity for a homelab.

The one thing to internalize: a tunnel is an *outbound* connection. That single fact is why the homelab survives an ISP failover untouched — the [network-topology recipe](/homelab/network-topology) walks through exactly how.

## Step 1 — Install and authenticate

`cloudflared` installs via Homebrew (it's in the toolchain list in the [cluster-host recipe](/homelab/cluster-host)):

```bash
brew install cloudflared
cloudflared tunnel login
```

`login` opens a browser, you pick the zone (`otterpond.dev`), and Cloudflare drops a certificate into `~/.cloudflared/`. That cert authorizes this machine to create and run tunnels for that zone.

## Step 2 — Create the tunnel

```bash
cloudflared tunnel create homelab
```

This creates the tunnel and writes a **credentials file** — `~/.cloudflared/<tunnel-id>.json`. That file is the tunnel's identity. Treat it like a private key: it does not go in Git, and it does not leave the host.

Confirm it exists:

```bash
cloudflared tunnel list
```

## Step 3 — Write the config

The tunnel config is an *ingress map*: hostname in, local service out. Create `~/.cloudflared/config.yml`:

```yaml
tunnel: <tunnel-id>
credentials-file: /Users/server/.cloudflared/<tunnel-id>.json

ingress:
  # Each public hostname routes to the cluster's web port.
  - hostname: otterpond.dev
    service: http://127.0.0.1:80
  - hostname: www.otterpond.dev
    service: http://127.0.0.1:80
  - hostname: api.otterpond.dev
    service: http://127.0.0.1:80

  # Required: a catch-all must be last.
  - service: http_status:404
```

Two things worth understanding:

- **Everything routes to `http://127.0.0.1:80`.** That's the cluster's load balancer on the host's port 80 (mapped there when the cluster was created). Traefik inside the cluster looks at the `Host` header and routes to the right service. So `cloudflared` doesn't need to know about individual apps — it just hands all web traffic to the cluster, and the cluster sorts it out. The tunnel config changes rarely; your Kubernetes ingress changes often.
- **The catch-all rule is mandatory and must be last.** Anything not matching a hostname above gets a clean 404 instead of an error. `cloudflared` refuses to start without it.

## Step 4 — Route DNS

The tunnel exists, but Cloudflare's DNS doesn't point at it yet. One command per hostname:

```bash
cloudflared tunnel route dns homelab otterpond.dev
cloudflared tunnel route dns homelab www.otterpond.dev
cloudflared tunnel route dns homelab api.otterpond.dev
```

Each creates a `CNAME` in Cloudflare pointing the hostname at the tunnel. Check what's routed:

```bash
cloudflared tunnel route dns list
```

## Step 5 — Run it as a service

`cloudflared` must run all the time and restart on boot — if it's down, every public site is down. Install it as a system service:

```bash
sudo cloudflared service install
```

On macOS this registers a `launchd` daemon (`/Library/LaunchDaemons/com.cloudflare.cloudflared.plist`) that starts the tunnel at boot and keeps it running. Confirm:

```bash
cloudflared tunnel info homelab    # should show an active connection
```

Then load the public hostname in a browser. You should reach your service, with a valid certificate, served from a Mac Mini that has no open inbound ports.

## Adding a new site

Once the tunnel is up, putting a new site online is a three-part move. Say you're adding `blog.otterpond.dev`.

**1. Tell the cluster about it.** Deploy the app and create a Kubernetes ingress for `blog.otterpond.dev` (the [TLS recipe](/homelab/tls-letsencrypt) shows the ingress shape). This is where the real routing config lives.

**2. Add the hostname to the tunnel.** Add one block to `~/.cloudflared/config.yml`, above the catch-all:

```yaml
  - hostname: blog.otterpond.dev
    service: http://127.0.0.1:80
```

Restart the service so it reloads the config:

```bash
sudo launchctl kickstart -k system/com.cloudflare.cloudflared
```

**3. Route DNS:**

```bash
cloudflared tunnel route dns homelab blog.otterpond.dev
```

That's it. New site, publicly reachable, TLS handled, no firewall change. The pattern is always the same: **cluster ingress → tunnel config → DNS route.** After you've done it twice it's muscle memory.

> If a hostname serves the *same* cluster, step 2 is the only part that's easy to forget — and the failure mode is a confusing 404 from the tunnel's catch-all rule rather than from your app. When a new site 404s, check the tunnel config first.

## When it breaks

### Symptom: a public site returns 502 Bad Gateway

The tunnel is up but it can't reach the local service. `cloudflared` is connected to Cloudflare fine; the problem is between `cloudflared` and the cluster. Check, in order:

```bash
docker ps | grep k3d                       # is the cluster running?
curl -I -H "Host: api.otterpond.dev" http://127.0.0.1:80   # does the cluster answer locally?
```

If the local `curl` fails, this isn't a tunnel problem — it's a cluster or ingress problem. Fix that and the 502 clears.

### Symptom: a new site returns 404

Almost always step 2 above — the hostname is missing from `~/.cloudflared/config.yml`, so the catch-all rule is answering. Add the block, kickstart the service, done. If the config *does* have it, check that the DNS route was created (`cloudflared tunnel route dns list`).

### Symptom: everything public is down

Check whether `cloudflared` itself is connected:

```bash
cloudflared tunnel info homelab
sudo launchctl print system/com.cloudflare.cloudflared | grep -i state
```

If the tunnel has no active connections, the daemon died or can't reach Cloudflare. Restart it:

```bash
sudo launchctl kickstart -k system/com.cloudflare.cloudflared
```

If it still won't connect, the host has lost outbound internet entirely — that's a network problem, not a tunnel problem. (After an ISP failover, give it the reconnect window before assuming anything is wrong; the [network-topology recipe](/homelab/network-topology) explains the timing.)

### Symptom: config changes aren't taking effect

`cloudflared` reads its config at startup. Editing `config.yml` does nothing until the service restarts. Always `launchctl kickstart` after a config edit. If you're not sure the config is even valid, dry-run it:

```bash
cloudflared tunnel ingress validate
cloudflared tunnel ingress rule https://blog.otterpond.dev   # which rule matches this URL?
```
