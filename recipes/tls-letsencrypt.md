---
num: "03"
title: "TLS Certificates with Let's Encrypt"
desc: "Automatic, auto-renewing HTTPS with cert-manager and Cloudflare DNS-01, including real certificates for VPN-only internal sites."
tags: ["TLS", "cert-manager", "Let's Encrypt", "DNS-01"]
color: success
pubDate: 2026-05-17
---

## Context

Every service in this homelab serves HTTPS, including the ones only you can reach. Certificates expire every 90 days, and a homelab operator who has to remember to renew certificates by hand will, eventually, forget. So we don't renew by hand. We install [cert-manager](https://cert-manager.io/), point it at Let's Encrypt, and never think about it again.

The interesting decision here is *how* cert-manager proves to Let's Encrypt that you own the domain.

## Why DNS-01

Let's Encrypt offers two ways to prove domain ownership:

- **HTTP-01** — Let's Encrypt connects to `http://yoursite/.well-known/...` and checks for a token. This requires your site to be publicly reachable on port 80. It also can't issue wildcard certificates.
- **DNS-01** — cert-manager creates a temporary `TXT` record in your DNS, Let's Encrypt checks it, done. No inbound connection to your homelab at all.

DNS-01 wins for a homelab, for two reasons that matter a lot:

1. **It issues wildcard certificates.** One cert for `*.otterpond.dev` covers every service you'll ever add. No per-site issuance.
2. **It works for sites that aren't publicly reachable.** This is the big one. Your private, VPN-only admin dashboards still get real, browser-trusted certificates — because the proof happens in DNS, not over an inbound HTTP connection. More on that below.

The cost is that cert-manager needs API access to edit your DNS. We use Cloudflare for DNS, and we'll give cert-manager a token scoped to *only* DNS edits.

## Step 1 — Install cert-manager

cert-manager installs via Helm:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# Wait for all three pods to be Running
kubectl -n cert-manager get pods
```

You're looking for `cert-manager`, `cert-manager-cainjector`, and `cert-manager-webhook`. If the webhook isn't ready, certificate requests will fail with confusing errors — wait for it.

## Step 2 — Create the Cloudflare API token

In the Cloudflare dashboard → **My Profile → API Tokens → Create Token**, use the **Edit zone DNS** template:

- **Permissions:** `Zone → DNS → Edit` and `Zone → Zone → Read`
- **Zone Resources:** Include → Specific zone → `otterpond.dev`

That's least privilege — the token can edit DNS records and nothing else. If it leaks, the blast radius is "someone can mess with your DNS," not "someone owns your Cloudflare account."

Store the token in your vault (the [secrets recipe](/homelab/1password-secrets)) and sync it into the cluster as a secret. The secret must live in the `cert-manager` namespace:

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token="$CF_TOKEN"
```

> The [secrets recipe](/homelab/1password-secrets) makes this part of a repeatable script, so you're not running `kubectl create secret` by hand and pasting tokens into your shell. For now, the manual command shows the shape.

## Step 3 — Create the ClusterIssuers

A `ClusterIssuer` tells cert-manager *which* CA to use and *how* to solve challenges. Create **two** — production and staging.

Why two? Let's Encrypt's production endpoint has [rate limits](https://letsencrypt.org/docs/rate-limits/) — issue too many certs for one domain in a week and you're locked out for a week. When you're first wiring this up and things are failing, you *will* burn through issuance attempts. So you test against the staging CA (high limits, but untrusted certs) and only switch to production once it works.

`letsencrypt-cf` (production):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cf
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@otterpond.dev
    privateKeySecretRef:
      name: letsencrypt-cf-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

`letsencrypt-cf-staging` is identical except two lines:

```yaml
metadata:
  name: letsencrypt-cf-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
```

Apply both and confirm they're ready:

```bash
kubectl apply -f letsencrypt-cf.yaml
kubectl apply -f letsencrypt-cf-staging.yaml
kubectl get clusterissuer        # READY should be True
```

## Step 4 — Issue the wildcard certificate

One certificate covers the whole domain:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-otterpond
  namespace: apps
spec:
  secretName: wildcard-otterpond-tls
  issuerRef:
    name: letsencrypt-cf
    kind: ClusterIssuer
  dnsNames:
    - otterpond.dev
    - "*.otterpond.dev"
```

Apply it and watch:

```bash
kubectl apply -f wildcard-otterpond.yaml
kubectl -n apps get certificate wildcard-otterpond -w   # wait for READY=True
```

Issuance takes a minute or two — cert-manager has to write the DNS record, wait for it to propagate, and let Let's Encrypt verify it. When it's done, you have a TLS secret `wildcard-otterpond-tls` that any ingress in that namespace can use.

> **Test with the staging issuer first.** Set `issuerRef.name` to `letsencrypt-cf-staging`, confirm the certificate reaches `READY=True`, then switch to `letsencrypt-cf` and delete the staging secret so it re-issues from production. Staging certs are real certificates that browsers don't trust — fine for proving the plumbing works, not fine for actual use.

## The nested-subdomain trap

This one catches everyone exactly once. A wildcard certificate covers **one level** of subdomain and no more:

| Hostname | Covered by `*.otterpond.dev`? |
|----------|-------------------------------|
| `otterpond.dev` | Yes (listed explicitly) |
| `api.otterpond.dev` | Yes |
| `logs.otterpond.dev` | Yes |
| `api.internal.otterpond.dev` | **No** |

`*.otterpond.dev` does **not** match `api.internal.otterpond.dev` — that's two levels deep. If you start nesting subdomains, you need either an additional wildcard (`*.internal.otterpond.dev`) added to the certificate's `dnsNames`, or a dedicated certificate for that host. Decide your naming scheme early and stay flat if you can. `api.otterpond.dev` is easier to live with than `api.apps.internal.otterpond.dev`, and it's also free of this trap.

## Internal-only sites still get real certificates

This is the part that surprises people, so it gets its own section.

Tailscale puts your admin surfaces — dashboards, the logging UI, anything you don't want on the public internet — on a private network. They have no public DNS pointing at a public IP, no Cloudflare Tunnel route, no inbound exposure whatsoever. The only way to reach them is to be on your tailnet.

You might assume those sites are stuck with self-signed certificates and a lifetime of browser warnings. They are not.

Because cert-manager proves ownership via **DNS-01**, it never needs to *reach* the site to get it a certificate. It only needs to write a `TXT` record. So a purely internal service gets a real, browser-trusted, auto-renewing Let's Encrypt certificate exactly like a public one. The wildcard `*.otterpond.dev` already covers it.

The mental model that makes this click:

- **DNS** decides what a hostname resolves to (and that's where Tailscale's MagicDNS quietly points internal names at tailnet addresses).
- **The certificate** only proves you control the *name*.
- **Network reachability** is a completely separate concern, handled by Tailscale.

So `logs.otterpond.dev` can resolve only on your tailnet, be reachable only by you — and still show a clean padlock with no warnings. Real TLS is not a thing you trade away for privacy. The [Tailscale recipe](/homelab/tailscale) wires up the access side; the certificate side is already done, right here.

## Using the certificate

Reference the TLS secret from any ingress in the `apps` namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: apps
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - app.otterpond.dev
      secretName: wildcard-otterpond-tls
  rules:
    - host: app.otterpond.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

> Public sites fronted by a Cloudflare Tunnel get TLS *for free* at Cloudflare's edge and don't strictly need a cluster-side certificate. The wildcard is what covers the internal sites and any host you serve with direct TLS. Issue it regardless — it's one certificate and it covers everything.

## Renewal

There is no renewal step. cert-manager renews automatically, 30 days before expiry. That's the entire point of this recipe. The only thing you might ever do is watch it happen:

```bash
kubectl -n apps get certificate wildcard-otterpond -w
```

## When it breaks

A certificate stuck on `READY=False` is the usual failure. cert-manager models issuance as a chain of resources, and the trick is to walk that chain from the top until you find the one that's unhappy:

```
Certificate → CertificateRequest → Order → Challenge
```

Walk it like this:

```bash
# 1. The Certificate — the top-level intent.
kubectl -n apps describe certificate wildcard-otterpond

# 2. The CertificateRequest — one per issuance attempt.
kubectl -n apps get certificaterequest
kubectl -n apps describe certificaterequest <name>

# 3. The Order — the ACME order with Let's Encrypt.
kubectl -n apps get order
kubectl -n apps describe order <name>

# 4. The Challenge — where DNS-01 actually happens.
kubectl -n apps get challenge
kubectl -n apps describe challenge <name>

# 5. cert-manager's own logs, if the chain didn't make it obvious.
kubectl -n cert-manager logs -l app=cert-manager --tail=200
```

The `describe` on the failing resource almost always names the real problem. Common ones:

**Challenge stuck "pending", reason `DNS01`.** cert-manager can't write or verify the `TXT` record. Check the API token:

```bash
kubectl -n cert-manager get secret cloudflare-api-token         # exists?
dig _acme-challenge.otterpond.dev TXT                            # record present?
```

If the secret is missing or the token lacks `DNS:Edit` on the zone, fix that and the challenge retries on its own.

**`Order` failing with a rate-limit error.** You've hit Let's Encrypt's production limit — too many issuance attempts in a week. This is exactly what the staging issuer is for. Switch the certificate's `issuerRef` to `letsencrypt-cf-staging`, get the plumbing working there, and only move back to production once it's clean. The production limit resets after a week; there is no way to speed it up.

**Certificate never appears at all.** Check the cert-manager webhook is actually running (`kubectl -n cert-manager get pods`). A dead webhook silently swallows new `Certificate` resources.

### Inspecting what you got

To see a certificate's real expiry and which names it covers:

```bash
kubectl -n apps get secret wildcard-otterpond-tls \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -noout -subject -dates -ext subjectAltName
```

If `subjectAltName` doesn't list the host you're trying to serve, you've hit the nested-subdomain trap above — go add the name to `dnsNames`.
