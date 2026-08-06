---
num: "07"
title: "CI/CD with GitHub Actions"
desc: "Push-to-deploy into a private cluster — runner options, ephemeral Tailscale, and a staging environment that runs on real data."
tags: ["CI/CD", "GitHub Actions", "Kubernetes"]
color: purple
pubDate: 2026-05-17
---

## Context

Deploying by hand — SSH in, `git pull`, rebuild, restart — works exactly until the day you misremember a step at 11pm. A pipeline removes the memory from the equation: you push a commit, and a known-good sequence builds an image and rolls it out the same way every time.

The wrinkle for a homelab is that the cluster isn't on the public internet. GitHub's runners live in a datacenter; your cluster lives in your house behind no open ports. This recipe bridges that gap with an ephemeral Tailscale connection, and along the way builds a staging environment that runs on *real* data.

## The pipeline shape

Two stages, deliberately separate (this is the build/release/run split from the [twelve-factor recipe](/homelab/twelve-factor)):

1. **Build & publish** — on every push, build a container image, tag it with the commit SHA, push it to a registry (GitHub Container Registry — GHCR — is free and right there).
2. **Deploy** — take a published image and roll it out to the cluster.

Keeping them separate means a deploy is just "point the cluster at an image that already exists and was already tested." You can redeploy or roll back without rebuilding, and a deploy never depends on a build succeeding *right now*.

The build stage is unremarkable — `docker build`, `docker push`, standard GitHub Actions. The interesting part is the deploy stage reaching a cluster it can't see.

## Where the runners run

GitHub Actions runs jobs on **GitHub-hosted runners** (fresh VMs in GitHub's datacenter) or on **self-hosted runners** (compute you provide and register). For a homelab there are really three options, and they trade off differently.

**1. A runner installed natively on the host.** The obvious first idea: register a self-hosted runner directly on the Mac Mini. It's always on, it's already on the cluster's network. I tried this, and it doesn't hold up — the runner shares the **same Docker/Colima instance as production**, with no isolation between the two. CI is a messy, resource-hungry neighbor: it builds images, pulls layers, spins up throwaway containers, chews memory and disk. Run that on the same daemon hosting your live cluster and a heavy pipeline run contends with production for the same memory and disk — your live site gets slow at exactly the moment a build is hammering the box. This is the one approach to actively rule out: the problem isn't *self-hosting*, it's *self-hosting with no isolation*.

**2. GitHub-hosted runners.** Each job gets a clean, disposable VM in GitHub's datacenter — strong isolation, zero runner maintenance, nothing running on your hardware. It runs on GitHub's metered minutes, but for a homelab that bill is small: a private repo gets 2,000 minutes a month on the free plan, and the Pro plan — $4/month — raises that to 3,000. A homelab's deploy cadence doesn't come close to burning either. (Public repositories get unmetered minutes entirely.) The real catch is that the runner sits *outside* your network, so it needs a way in to reach the private cluster — the ephemeral-Tailscale approach in the next section. It's the least to operate, and the worked example below uses it.

**3. Self-hosted runners via the Actions Runner Controller (ARC).** [ARC](https://docs.github.com/en/actions/tutorials/use-actions-runner-controller/get-started) runs self-hosted runners *as Kubernetes pods*. Each job gets a fresh runner pod, scheduled on the cluster and destroyed when the job ends — genuine, controller-managed ephemerality, with isolation at the pod boundary rather than none at all. CI compute stays on your own hardware (no metered minutes), and because the runner is already *inside* the cluster, it can reach the API directly — no Tailscale hop needed. The tradeoff is operational: you're now running ARC itself, a controller plus its scaling configuration, which is more to install, understand, and keep alive.

| Approach | Isolation | Ephemerality | Cluster access | Cost |
|----------|-----------|--------------|----------------|------|
| Native on host | None — shares prod's runtime | None | Trivial (but not worth it) | Free — but ruled out |
| GitHub-hosted | Strong (fresh VM) | Per-job | Needs a way in (Tailscale) | Metered minutes — 2k/mo free, 3k on Pro ($4/mo) |
| ARC on the cluster | Pod-level | Per-job, controller-managed | Direct (already in-cluster) | Your hardware + operating ARC |

Between GitHub-hosted and ARC, it's a real decision with no default-right answer: GitHub-hosted is less to operate; ARC keeps the compute on your own metal and hands you a runner lifecycle you control end to end. The worked example below uses GitHub-hosted runners because it's the shorter path to a green pipeline — not because it's the better pick. If you'd rather own the whole loop, ARC is a natural homelab fit, with setup and scaling worth a recipe of their own.

## Reaching the cluster: ephemeral Tailscale

This section is the GitHub-hosted path — a runner outside your network that needs a way in. If you went with ARC, the runner pods are already inside the cluster and can reach the API directly; skip ahead to the deploy step.

The deploy job joins your tailnet for the duration of the run, using the OAuth client from the [Tailscale recipe](/homelab/tailscale). It joins as an ephemeral, `tag:ci`-tagged device that disappears the moment the job ends.

Three GitHub Actions secrets make this work:

| Secret | What it is |
|--------|-----------|
| `TS_OAUTH_CLIENT_ID` | Tailscale OAuth client ID |
| `TS_OAUTH_CLIENT_SECRET` | Tailscale OAuth client secret |
| `KUBE_CONFIG` | The cluster's kubeconfig (see the note below) |

Set them with the GitHub CLI (which reads the value from a prompt or file, *not* from your shell history):

```bash
gh secret set TS_OAUTH_CLIENT_ID
gh secret set TS_OAUTH_CLIENT_SECRET
gh secret set KUBE_CONFIG < homelab-kubeconfig.yaml
```

> **The kubeconfig gotcha.** The kubeconfig that k3d generates points the API server at `localhost` — correct on the Mac Mini, useless from a GitHub runner. Before storing it as `KUBE_CONFIG`, edit the `server:` line to the Mac Mini's **Tailscale hostname** (`https://homelab-server:6550`). This is also why the cluster was created with that hostname in its `--tls-san` list — so the API server's certificate is valid for the name the runner will actually use. Miss either half and the runner connects but fails TLS verification.

The deploy job:

```yaml
deploy:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Join Tailscale
      uses: tailscale/github-action@v3
      with:
        oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
        oauth-secret: ${{ secrets.TS_OAUTH_CLIENT_SECRET }}
        tags: tag:ci

    - name: Configure kubectl
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBE_CONFIG }}" > ~/.kube/config

    - name: Deploy
      run: |
        kubectl set image deployment/my-app \
          my-app=ghcr.io/you/my-app:sha-${GITHUB_SHA::7} -n apps
        kubectl rollout status deployment/my-app -n apps --timeout=180s
```

The runner joins the tailnet, can now reach `homelab-server:6550`, points the deployment at the freshly built image, and waits for the rollout to finish so a failed deploy fails the workflow. When the job ends, the ephemeral device is gone — nothing lingers on your tailnet.

For anything beyond a single deployment, manage manifests with **Kustomize** and apply an overlay (`kubectl apply -k overlays/production`) instead of imperative `set image` commands. Overlays are what make the staging environment below cheap.

## Staging deploys, with real data

A staging environment is only worth having if it tells you the truth, and it only tells you the truth if it looks like production. Same cluster, same images, same manifests — and, critically, *real data*. Staging that runs on three hand-typed test rows will happily pass a deploy that explodes on production's actual data.

### Same everything, different overlay

Run staging as a second Kustomize overlay — its own namespace (`staging`), its own hostnames, its own secrets, but the *same base manifests and the same images* as production. The deploy workflow takes an `environment` input and picks the overlay:

```yaml
- name: Deploy
  run: kubectl apply -k overlays/${{ inputs.environment }}
```

Staging differs from production in *where it runs* and *what data it holds* — never in *what's running*. The moment staging drifts to a different image or a different manifest, it stops being a useful rehearsal.

### Cloning production data into staging

Give staging real data by cloning production's database into it. Run this as a Kubernetes `Job` in the staging namespace as part of the staging deploy: it `pg_dump`s production and restores into the staging database.

One important refinement — **skip the data in large, static reference tables.** Many databases have a few tables that are huge but rarely change (imported datasets, lookup tables). Copying their *schema* matters; copying millions of rows of their *data* just makes every clone slow. Exclude the data, keep the structure:

```bash
pg_dump "$PROD_URL" \
  --no-owner --no-privileges \
  --exclude-table-data='"large_reference_table"' \
  --exclude-table-data='"another_static_table"' \
  -Fc -f /tmp/dump

pg_restore --no-owner --no-privileges -d "$STAGING_URL" /tmp/dump
```

`--no-owner` and `--no-privileges` drop ownership and grant statements so the restore doesn't fail when the source and destination use different database roles. `pg_restore` will print warnings about that — expected, not a problem.

### The same clone, locally

The exact same idea works for pulling production data onto your laptop for development — connect to the production database over Tailscale, dump, restore into a local PostgreSQL container. Wrap it in a script that:

1. Fetches production credentials from the vault (the [secrets recipe](/homelab/1password-secrets)).
2. Backs up your *current* local database first, so a bad sync is reversible.
3. Dumps production (with the `--exclude-table-data` exclusions above).
4. Drops, recreates, and restores the local database.
5. Cleans up the dump file.

Same dump, same exclusions, different destination. Production data on your laptop, one command, and a rollback if you don't like the result.

## When it breaks

### Symptom: deploy job can't reach the cluster

The runner joined Tailscale but `kubectl` times out or fails TLS. Two usual causes:

- **The kubeconfig still says `localhost`.** A runner pointed at `localhost` is talking to itself. The `server:` line must be the Tailscale hostname. Re-check the stored `KUBE_CONFIG`.
- **TLS hostname mismatch** — the runner reaches the API but rejects its certificate. The cluster's API server cert must be valid for the hostname in the kubeconfig; that name has to be in the cluster's `--tls-san` list. If it isn't, recreate the cluster with it included.

If the runner never joined the tailnet at all, that's an OAuth/ACL problem — the [Tailscale recipe](/homelab/tailscale)'s troubleshooting covers it.

### Symptom: deploy succeeds but the rollout never completes

`kubectl rollout status` is timing out — the new pods aren't going healthy. The deploy did its job; the *app* is unhappy. Look at the pods directly:

```bash
kubectl -n apps get pods
kubectl -n apps describe pod <pod>
kubectl -n apps logs <pod>
```

Common culprits: the image tag doesn't exist in the registry (build stage failed or the SHA is wrong), an image pull secret is missing, or the app is crashing on a bad config or a missing secret.

### Symptom: staging clone job fails

- **`pg_restore` exits non-zero with warnings** about roles or ownership — that's expected with `--no-owner`/`--no-privileges`. Check whether the data actually landed before treating it as a failure.
- **Can't connect to the production database** — the clone job (or local script) can't reach production over Tailscale. Confirm tailnet connectivity and that the database host is reachable.
- **The clone is painfully slow** — you're copying data from a large reference table. Add it to the `--exclude-table-data` list.
