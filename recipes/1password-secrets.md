---
num: "08"
title: "1Password as a Secret Vault"
desc: "One source of truth for secrets, synced into Kubernetes on demand — and why you never put a secret on the command line."
tags: ["Secrets", "1Password", "Kubernetes"]
color: pink
pubDate: 2026-05-17
---

## Context

Every service in the homelab needs secrets — database passwords, API tokens, signing keys. Those secrets have to end up as Kubernetes `Secret` objects so pods can use them. The question is *where they live the rest of the time*, and the wrong answers are everywhere:

- **In Git** — even a private repo. One bad push and it's permanent history.
- **Only in the cluster** — then the cluster *is* your backup, and a rebuild loses every secret.
- **In a `.env` file on your laptop** — until the laptop dies, or you need the same secret on another machine.
- **In your head** — for about a week.

This is the "config in the environment" factor from the [twelve-factor recipe](/homelab/twelve-factor), and it has one real requirement: a **single source of truth** for secrets that is not the cluster, not Git, and not a file you have to remember to back up. This homelab uses 1Password for that, with a small script that syncs secrets into Kubernetes on demand.

## Why a password manager as the vault

You almost certainly already have a password manager, it's already encrypted, already backed up, already synced across your devices, and already has a CLI. That's the entire argument. You *could* run HashiCorp Vault. You'd then be operating a secrets service to support the homelab — exactly the "infrastructure to support your infrastructure" trap the [twelve-factor recipe](/homelab/twelve-factor) warns against. A vault you already trust with the rest of your life is a better answer than a vault you have to babysit.

This recipe uses 1Password and its `op` CLI. Any password manager with a scriptable CLI works the same way; the *pattern* is the point.

## The model

```
   1Password vault  ──(op CLI)──▶  sync script  ──(kubectl)──▶  Kubernetes Secrets
   (source of truth)                (on demand)                  (consumed by pods)
```

- **The vault is the source of truth.** Secrets are created and edited there.
- **Kubernetes Secrets are a derived cache.** They're generated *from* the vault and can be regenerated any time. They are disposable, like everything else in the cluster.
- **A script does the sync.** Re-runnable, idempotent, the only thing that ever creates a cluster secret.

The payoff: rebuilding the cluster doesn't mean recovering secrets. You re-run the sync script and the cluster is repopulated from the vault. Secrets stop being a thing you can lose.

## Step 1 — The CLI

```bash
brew install 1password-cli
op signin
```

`op signin` authenticates the session. Test a read:

```bash
op vault list
```

## Step 2 — Lay out the vault

Make a dedicated vault — `Homelab` — so the homelab's machine-readable secrets aren't tangled up with your personal logins. Inside it, one item per logical credential:

| Item | Holds |
|------|-------|
| `Database` | PostgreSQL host, port, database, username, password |
| `JWT Key` | Token-signing key for your apps |
| `Cloudflare API` | DNS-edit token (used by cert-manager) |
| `Cloudflare R2` | Account ID + access keys for off-site backups |
| `Container Registry` | Username + token for pulling private images |

Each item carries named fields, read back by name:

```bash
op item get "Database" --vault Homelab --fields password --reveal
```

`--reveal` prints the secret value rather than a masked placeholder — which is the point inside a script, and something to be deliberate about anywhere else.

## Step 3 — The sync script

The sync script reads items from the vault and creates the matching Kubernetes secrets. The shape, for one secret:

```bash
#!/usr/bin/env bash
set -euo pipefail

VAULT="Homelab"
NAMESPACE="apps"

# Read each field from the vault into a shell variable.
DB_HOST=$(op item get "Database" --vault "$VAULT" --fields server   --reveal)
DB_PORT=$(op item get "Database" --vault "$VAULT" --fields port     --reveal)
DB_NAME=$(op item get "Database" --vault "$VAULT" --fields database --reveal)
DB_USER=$(op item get "Database" --vault "$VAULT" --fields username --reveal)
DB_PASS=$(op item get "Database" --vault "$VAULT" --fields password --reveal)

CONN="Host=${DB_HOST};Port=${DB_PORT};Database=${DB_NAME};Username=${DB_USER};Password=${DB_PASS}"

# Recreate the secret idempotently — delete-then-create so re-runs update it.
kubectl delete secret app-secrets -n "$NAMESPACE" --ignore-not-found
kubectl create secret generic app-secrets -n "$NAMESPACE" \
  --from-literal=database-connection-string="$CONN"
```

The properties that make this work:

- **Idempotent.** Delete-then-create means running it twice is safe, and running it is how you *update* a secret — change the value in 1Password, re-run the script.
- **Re-runnable after a rebuild.** A fresh cluster becomes a fully-secreted cluster with one command.
- **Secrets never touch disk or Git.** They go vault → shell variable → `kubectl`, and vanish when the script exits.

Make the script create vault items that don't exist yet (prompting you for the value) and you have one command that bootstraps the vault *and* the cluster. Worth the extra few lines.

## Never put a secret on the command line

This is the security rule that matters most, and it's the one most easily broken by accident.

**Anything you type as a command-line argument is recorded.** Your shell writes it to history (`~/.zsh_history`), and while it runs it's visible in the process list to anything that can read `ps`. So this:

```bash
some-tool set-token "ghp_REALSECRETVALUE123456"   # DON'T
```

just wrote a live credential into a plaintext file on your disk, where it will sit indefinitely, get synced to backups, and quietly outlive your memory of having typed it. This is not hypothetical — it is the single most common way homelab secrets leak, and it leaks them to *yourself* in a file you forget exists.

The fixes are all easy:

- **Reference a variable, never a literal.** `--from-literal=key="$DB_PASS"` is fine — your history records the text `$DB_PASS`, not its value. `--from-literal=key="ghp_real..."` is not. This is exactly why the sync script reads into variables first.
- **Pipe from `op` directly:**
  ```bash
  gh secret set MY_TOKEN < <(op item get "Some API" --vault Homelab --fields token --reveal)
  ```
- **Read from stdin** when a tool supports it — `op item get ... | tool --token-stdin`.
- **Use `op run`** to inject secrets as environment variables into a command without them ever being typed:
  ```bash
  op run --env-file=.env.tpl -- ./some-script.sh
  ```

And if a secret *does* end up in your history: the secret is now compromised — **rotate it**, don't just delete the history line. A deleted history entry doesn't un-leak a credential that was sitting in a plaintext file. Regenerate the token, update the vault, re-run the sync.

## When it breaks

### Symptom: the script fails with auth errors

The `op` session expired. Sessions are short-lived by design:

```bash
op signin
```

Then re-run the sync. If it's failing inside CI or a non-interactive context, you need a 1Password *service account* token rather than an interactive `op signin` — interactive auth can't work where there's no human.

### Symptom: a pod can't find its secret

The secret in the cluster doesn't match what the pod expects. Check what's actually there:

```bash
kubectl -n apps get secret app-secrets -o jsonpath='{.data}' | jq 'keys'
```

That lists the *keys* without revealing values. A missing or misnamed key means the script and the pod's manifest disagree — line them up. If the key exists but the value looks wrong, the vault item changed and the cluster is stale: re-run the sync.

### Symptom: secrets work, then later don't

You rotated something in 1Password but never re-synced — the cluster is still holding the old value. The vault is the source of truth, but the cluster only learns about a change when the script runs. Rotating a secret is always two steps: **update the vault, re-run the sync.** A pod restart after the sync picks up the new value.
