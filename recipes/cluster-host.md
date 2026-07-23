---
num: "02"
title: "Provisioning the Cluster Host"
desc: "Turning a bare Mac Mini into a Kubernetes cluster host with Homebrew, Colima, and k3d — one that survives reboots and power blips."
tags: ["Kubernetes", "Colima", "k3d", "macOS"]
color: teal
pubDate: 2026-05-17
---

## Context

Everything in this homelab runs on one machine. This recipe turns a bare Mac Mini into a Kubernetes cluster host that survives reboots, power blips, and your own future meddling.

The goal is a host that is *boring*. Its entire job is to run a container runtime and a single-node cluster. It should not be a development machine, it should not have Opinions, and you should be able to rebuild it from this recipe in under half an hour. That last property — disposability — is the whole point.

## Why a Mac Mini, and why Colima

A Mac Mini is a genuinely good homelab server: low idle power, silent, fast ARM cores, and it just sits there. The catch is that macOS doesn't run Linux containers natively — it needs a Linux VM underneath.

You have two realistic options for that VM:

- **Docker Desktop** — works, but it's a heavy GUI app that wants to update itself, show you things, and license itself. It is built for a developer's laptop, not a headless server.
- **Colima** — a small, CLI-only tool that manages a Lima VM and exposes a standard Docker socket. No GUI, no login, no telemetry. It is built for exactly this.

Use Colima. On a server you never look at, the last thing you want is a desktop app waiting for a click.

On top of Colima we run **k3d**, which runs **K3s** (a small, fully-conformant Kubernetes) inside Docker containers. So the stack is: macOS → Colima (Linux VM) → Docker → k3d → K3s. That's more layers than a cloud setup, but each layer is replaceable and each one is one command to rebuild.

> **Worth watching:** Apple's own [`container`](https://github.com/apple/container) project is a native, Swift-based tool for running Linux containers on Apple silicon, using lightweight per-container VMs on the Virtualization framework. It's young, but it's a plausible future replacement for the Colima layer — a more native, better-integrated base for exactly this kind of setup. Not adopting it today; keeping an eye on it. When it's ready, the layer below k3d is the one piece of this stack that would swap out, and the rest of the recipe wouldn't much care.

## Step 1 — The toolchain (Homebrew)

Homebrew is the package layer the entire homelab sits on. Install it first.

Get it from the official site — **[brew.sh](https://brew.sh)** — and follow the instructions there. It is deliberately *not* reproduced here as a `curl … | bash` one-liner. Piping a script straight from a URL into a shell means running whatever that URL serves *at the moment you run it*, unread — and install scripts, the repos that host them, and package tooling in general have been hijacked before. Go to the source, see what you're about to run, then run it. This is the one install the whole homelab's integrity rests on; it's worth thirty seconds of caution.

With Homebrew installed, pull the homelab toolchain:

```bash
brew install colima k3d kubectl helm k9s cloudflared 1password-cli awscli postgresql gh
```

`k9s` is a terminal UI for Kubernetes. You will live in it. It's not optional in any way that matters.

### The keg-only gotcha

Some formulae are **keg-only** — Homebrew installs them but deliberately does *not* put them on your `PATH`, usually to avoid clashing with a macOS-bundled version. `postgresql@<version>` is the one that will bite you, because the client tools (`pg_dump`, `pg_restore`, `psql`) silently aren't found, or — worse — an *older* macOS-bundled `psql` is found instead and you get cryptic version-mismatch errors later (the [PostgreSQL recipe](/homelab/postgres-and-backups) covers exactly that failure).

When `brew install` finishes, it prints the fix. Don't skim past it. It looks like:

```bash
echo 'export PATH="/opt/homebrew/opt/postgresql@18/bin:$PATH"' >> ~/.zshrc
```

Run that, open a new shell, confirm with `which pg_dump`. If `which` points at `/usr/bin` or finds nothing, the PATH entry didn't take.

### brew services

Homebrew can also run things as background services (`brew services start <formula>`). We'll use this for Colima autostart in Step 5. Worth knowing it exists now — it's the boring answer to "how do I make this start on boot," and boring is what we want.

## Step 2 — Start Colima

```bash
colima start --cpu 4 --memory 8 --disk 100
```

Size to your hardware — leave macOS a couple of cores and a few GB. The VM config lives at `~/.colima/default/colima.yaml` if you need to hand-tune it later; edit that file and `colima restart` to apply.

Verify Docker works through Colima:

```bash
docker info | grep -i 'server version'
```

### The storage-path gotcha

Colima only mounts *some* host directories into the Linux VM — by default, your home directory. A path like `/data/...` or `/Users/Shared/...` may not exist inside the VM at all, which means a container bind-mounting it gets an empty directory or fails outright.

This matters the moment you put real data on a host path — a PostgreSQL data volume is the obvious case. The rule:

> **Bind-mount host data under your home directory** (e.g. `~/homelab/postgres`). It's the path Colima reliably shares into the VM.

Learn this here so you don't learn it by losing a database later. You can check what's actually visible inside the VM with `colima ssh -- mount`.

## Step 3 — Create the cluster

k3d wraps cluster creation into one command. Run it on the host:

```bash
k3d cluster create homelab \
  --api-port 6550 \
  -p "80:80@loadbalancer" \
  -p "443:443@loadbalancer" \
  --servers 1 \
  --agents 1 \
  --volume k3d-storage:/var/lib/rancher/k3s/storage@all \
  --k3s-arg "--tls-san=localhost@server:*" \
  --k3s-arg "--tls-san=$(hostname -s)@server:*" \
  --k3s-arg "--tls-san=homelab-server.your-tailnet.ts.net@server:*"
```

What the non-obvious flags do:

- `--api-port 6550` — the Kubernetes API listens here. CI will reach it over Tailscale.
- `-p "80:80@loadbalancer"` / `443` — maps the host's web ports to the cluster's load balancer, so ingress works.
- `--volume k3d-storage:...@all` — a named Docker volume for persistent storage. **Without this, every cluster recreate wipes your persistent volumes.** This is the line that makes the cluster disposable but the *data* survivable.
- `--tls-san=...` — extra hostnames the API server's TLS certificate is valid for. You *must* include the Tailscale hostname here, or CI's `kubectl` calls fail certificate validation. Add every name you'll reach the API by.

Then export the kubeconfig:

```bash
k3d kubeconfig write homelab --output ~/homelab.yaml
```

> One edit: the generated kubeconfig points the API server at `localhost`. For any client that isn't the host itself — your laptop, CI — swap `localhost` for the Tailscale hostname before using it. The [CI/CD recipe](/homelab/github-actions-deploys) covers this.

Confirm the cluster is alive:

```bash
kubectl get nodes        # both should be Ready
kubectl get pods -A      # Traefik, CoreDNS, etc. should be Running
```

K3s ships with Traefik as the ingress controller and a local-path storage provisioner. You don't install those — they're already there.

## Step 4 — Make macOS behave like a server

A Mac Mini ships configured to be a desktop. Three of those defaults will hurt you.

**Stop it sleeping.** A sleeping server is an offline server:

```bash
sudo pmset -a sleep 0 displaysleep 0 disksleep 0
sudo pmset -a autorestart 1   # power back on automatically after a power loss
sudo pmset -a womp 1          # wake on network access
```

**Turn off FileVault.** FileVault encrypts the disk, and an encrypted disk has to be unlocked with a password at boot — *before* macOS finishes starting, before auto-login, before Colima, before anything. On a laptop that is exactly what you want. On a headless box in a closet, it means every reboot stops dead at the unlock screen until someone walks over and types a password — which defeats the point of a server that is supposed to recover on its own. Turn it off in **System Settings → Privacy & Security → FileVault**. macOS also disables auto-login while FileVault is on, so this isn't really optional here — it's a prerequisite for the next setting.

**Enable auto-login.** After a reboot — and there *will* be reboots — the machine must reach a logged-in user session on its own, because that's where Colima and your background services run. Set it in **System Settings → Users & Groups → Automatically log in as**. Yes, this trades a little physical security for a lot of uptime; for a box locked in your house, that's the right trade.

## Step 5 — Make Colima survive a reboot

This is the step that looks trivial and isn't. If Colima doesn't come back after a reboot, *nothing* comes back — no Docker, no cluster, no sites.

The tempting approach is a hand-rolled `launchd` agent: write a `.plist`, `launchctl bootstrap` it, done. In practice that path is a maze — `launchctl bootstrap` and `kickstart` throw errors like `Domain does not support specified action`, the plist needs `plutil -lint` to find the typo, and you'll burn an evening on it.

Skip the maze. Homebrew already solved this:

```bash
brew services start colima
```

Confirm it registered:

```bash
brew services list   # colima should show "started"
```

Now reboot the machine on purpose and confirm the cluster comes back by itself:

```bash
sudo reboot
# ...wait, then SSH back in...
docker info && kubectl get nodes
```

If you skip the deliberate reboot test, you haven't verified disposability — you've *assumed* it. Assume nothing about boot behavior.

## Step 6 — Fix ephemeral port exhaustion (do this now)

This one is included as a setup step on purpose, because debugging it cold is genuinely confusing — connections start failing, but only *sometimes*, and only after the box has been up a while.

**The problem.** Every TCP connection borrows a number from a pool of *ephemeral ports*. When a connection closes it sits in `TIME_WAIT` for a minute or two before its port is reusable. macOS defaults that pool to roughly `49152–65535` — about 16,000 ports.

A homelab host opens a *lot* of short-lived connections, and the volume isn't entirely under your control. The real-world trigger behind this recipe was **aggressive web crawlers** — bots indexing the public sites far harder than any human would. Every request arrives down the tunnel and fans out into a short-lived local connection (`cloudflared` → cluster, cluster → app, app → database). A crawler swarm turns that into thousands of connections a minute, all cycling into `TIME_WAIT` faster than the pool drains. The pool empties, and new connections — including ones with nothing to do with the crawler — start failing with "can't assign requested address." The whole box goes flaky because something on the internet decided to index you. That's what makes it confusing to debug cold: the symptom shows up far from the cause.

**Diagnosing it.** Count the `TIME_WAIT` entries:

```bash
netstat -an | grep TIME_WAIT | wc -l
```

If that number is in the tens of thousands, you've found it. (A real incident behind this recipe clocked 34,000+.) To see *which* process is the offender:

```bash
sudo lsof -iTCP -sTCP:TIME_WAIT | awk '{print $1}' | sort | uniq -c | sort -nr | head
```

**The fix.** Widen the pool by lowering its starting port:

```bash
sysctl net.inet.ip.portrange.first net.inet.ip.portrange.last   # check current
sudo sysctl -w net.inet.ip.portrange.first=10000                # apply now
```

That takes the pool from ~16k ports to ~55k — enough headroom that `TIME_WAIT` churn drains comfortably.

**Make it stick.** `sysctl -w` is lost on reboot, and macOS doesn't reliably honor `/etc/sysctl.conf` for these keys. Use a `launchd` daemon that re-applies it at every boot. Create `/Library/LaunchDaemons/dev.otterpond.portrange.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>            <string>dev.otterpond.portrange</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/sbin/sysctl</string>
    <string>-w</string>
    <string>net.inet.ip.portrange.first=10000</string>
  </array>
  <key>RunAtLoad</key>        <true/>
</dict>
</plist>
```

Load it:

```bash
sudo launchctl load /Library/LaunchDaemons/dev.otterpond.portrange.plist
```

Now the wider pool is the permanent default, and this incident never happens to you.

## When it breaks

Single-node clusters fail in a small number of predictable ways. Here's the ladder.

### First, try a restart — not a rebuild

A clean stop/start preserves all cluster state (pods, secrets, deployments) and is far faster than reprovisioning. It also refreshes the VM's networking, which fixes most "it was fine yesterday" problems:

```bash
k3d cluster stop homelab
colima stop
colima start
k3d cluster start homelab
kubectl get nodes        # wait for Ready
```

If `k3d cluster start` fails outright, *then* reprovision (see below).

### Symptom: pods can't reach the database; image pulls time out

Almost always a **stale `host.k3d.internal`**. k3d writes the Docker host's IP into `/etc/hosts` inside the cluster nodes when the cluster is created. If Colima's VM gets a new DHCP lease — after a restart, a sleep/wake, or a macOS update — that IP goes stale, and anything inside the cluster trying to reach a service on the host (like PostgreSQL) times out.

Check whether the IPs still match:

```bash
docker exec k3d-homelab-server-0 cat /etc/hosts | grep host.k3d
colima ssh -- ip -4 addr show eth0 | grep inet
```

If they differ, repoint the cluster nodes at the real VM IP:

```bash
VM_IP=$(colima ssh -- hostname -I | awk '{print $1}')

for node in k3d-homelab-server-0 k3d-homelab-agent-0; do
  docker exec "$node" sh -c \
    "cp /etc/hosts /tmp/hosts && \
     sed 's/[0-9.]*[[:space:]]*host.k3d.internal/'\"$VM_IP\"'\thost.k3d.internal/' /tmp/hosts > /etc/hosts"
done

# Restart anything that cached the old IP in a connection pool
kubectl -n apps delete pods -l app=<your-app>
```

(`/etc/hosts` inside the node is a Docker mount, so you can't `sed -i` it in place — copy, edit, overwrite.)

### Symptom: cluster networking is generally broken

When you're not sure *what's* wrong with cluster networking, check it from inside a node:

```bash
docker exec k3d-homelab-server-0 ip route | grep default     # default route exists?
docker exec k3d-homelab-server-0 ping -c2 <colima-vm-ip>     # host reachable?
ping -c2 k3d-homelab-server-0                                 # node name resolves?
```

If the default route or host gateway is missing, that's a Colima VM networking problem — go back to the restart sequence above; restarting Colima rebuilds the VM's networking.

### Symptom: a local dev server won't start, "port already in use"

A previous process is still holding the port. Find it and kill it:

```bash
lsof -i :4321        # whatever port is stuck
kill <pid>
```

Not exciting, but it's the answer 100% of the time.

### Full reprovision

If a restart won't bring the cluster back, rebuild it. Because everything is disposable, this is a chore, not a crisis:

```bash
k3d cluster delete homelab
# ...then re-run Step 3 of this recipe.
```

After the cluster is back, re-apply TLS and logging, then redeploy your apps. Persistent data survives in the `k3d-storage` volume and in your off-site backups — which is exactly why those exist.
