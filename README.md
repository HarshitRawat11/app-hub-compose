# compose/ — app-hub, always on

**Status: WRITTEN, NEVER DEPLOYED.** Nothing here has run on a host.

The second deployment target. The same three services that run on EKS, running
on one machine that stays up — because a bookmark page does not justify
$150–200/month, and the EKS path exists to teach Kubernetes rather than to host
a dashboard.

| Target | Purpose | Lifecycle |
|---|---|---|
| `infra/` + `manifests/` | Learning Terraform, EKS, observability, CI/CD | destroyed every session |
| **`compose/`** (here) | **The dashboard actually used** | **always on** |

**These must not converge.** The EKS path is never simplified for cost; this
path never acquires Kubernetes. What they share is **the image** — same
registry, same tag, same bytes.

---

## Why the laptop first, and not Oracle

Oracle Cloud Always Free is the better *machine* — 2 OCPU and 12 GB against a
spare laptop. It is not the better *first step*:

- **Its documented idle policy targets exactly this workload.** Oracle may
  reclaim an Always Free instance whose 95th-percentile CPU, network and memory
  all sit under 20% over 7 days. A bookmark dashboard will sit near 1%. The
  widely-repeated claim that upgrading to Pay As You Go exempts you is **not in
  the documentation** — do not rely on it without checking.
- **Signup and ARM capacity are lotteries you do not control**, and can take
  weeks.
- **The laptop is x86**, so there is no `arm64` rebuild to do first.

Every file here is identical for both. Starting on hardware you own proves the
Compose shape, the config split, the data decision and the tunnel — then Oracle
is a redeploy, not a rethink.

---

## Before you start

**1. Get this repo onto the host.** It lives at
`github.com/HarshitRawat11/app-hub-compose` — public, like the rest of app-hub.
Nothing secret is committed here: `.env` is gitignored and `.env.example` holds
only placeholders.

```bash
git clone git@github.com:HarshitRawat11/app-hub-compose.git && cd app-hub-compose
```

The host needs **git, docker and the Compose plugin, and nothing else.** The
services arrive as images from ECR, not as source — there is no build step
here and there should never be one, because the whole point is that the host
runs the same bytes the cluster runs.

*(Git identity in this repo is set **locally**, not globally — the development
machine is work-managed and personal commits must not carry the work identity.
That matters when committing from the dev box, not on the host.)*

**2. Create the IAM user.** This is the one genuinely new AWS object, and it is
the security decision worth getting right.

Policy: exactly the four DynamoDB actions the repository layer performs, on the
one table, plus read-only ECR pull:

| Action | Resource |
|---|---|
| `dynamodb:GetItem`, `PutItem`, `DeleteItem`, `Scan` | the `app-hub-links` table ARN only |
| `ecr:GetAuthorizationToken` | `*` — account-level by the shape of the API |
| `ecr:BatchGetImage`, `GetDownloadUrlForLayer`, `BatchCheckLayerAvailability` | the three app-hub repository ARNs |

**The policy is written and ready to paste: [`iam-policy.json`](iam-policy.json).**
IAM console → Users → *Create user* (no console access) → *Attach policies
directly* → **Create policy** → **JSON** tab → paste the file → attach it to the
new user → then *Security credentials* → *Create access key* → **Other**.

**Every ARN in it was read from AWS, not constructed** — `describe-table` and
`describe-repositories` — because `infra/irsa.tf` already carries the reason:
an ARN built from account and region is *"silently wrong the day anything
moves"*, and ECR moved once already (2026-09-18).

**It validates clean against AWS IAM Access Analyzer**, with zero findings —
and the check was proved capable of failing first, by running a deliberately
bad policy through it and watching it return `INVALID_ACTION` and two security
warnings. A validator that has never been seen to fail proves nothing:

```bash
wsl -e bash -lc "cd /mnt/c/Users/harshit.rawat/Documents/Projects/app-hub/compose && aws accessanalyzer validate-policy --policy-document file://iam-policy.json --policy-type IDENTITY_POLICY --region ap-south-1 --query 'findings[].issueCode' --output table"
```

**The four DynamoDB actions are the same four `infra/irsa.tf` grants the pod** —
not a superset. If the EKS deployment can do it, this host can; if it cannot,
neither can this. Divergence there would mean the two deployments behave
differently against the same table, which is worse than either being wrong.

**Create the access key by hand, not in Terraform.** `aws_iam_access_key` puts
the secret into Terraform state — a long-lived credential in S3, readable by
anything that can read state. The user and policy are fine in Terraform; the
key is not.

**3. Set up Tailscale.** Five things, and the fifth is the one that bites six
months from now.

**Why Tailscale and not Cloudflare Tunnel** — a named Cloudflare Tunnel's public
hostname must live on a **domain in your Cloudflare account**, and this project
owns none. Cloudflare has no free equivalent of the `*.pages.dev` name that
Pages hands out; the free option is a Quick Tunnel, whose URL changes on every
restart. Tailscale gives a stable HTTPS hostname on the free plan with no domain
purchase.

**(a) Create the account** — free Personal plan at `login.tailscale.com`. Yours
to do; I do not create accounts.

**(b) Enable HTTPS certificates.** Admin console → **DNS** → *Enable HTTPS*.
Without it there is no certificate and Funnel cannot serve TLS at all.

**(c) Enable Funnel**, which is **off by default for a tailnet**. The complete
policy file is written and ready to paste:
[`tailscale-acl.hujson`](tailscale-acl.hujson). Admin console → **Access
controls** → replace the document → **Save**.

> **It is the WHOLE file, not a fragment, and that is deliberate.** That screen
> edits one document for the entire tailnet. Pasting only a `nodeAttrs` block
> replaces everything else, including the `acls` rule that lets your own devices
> reach each other — you would cut yourself off from n8n while fixing Funnel.

**It scopes Funnel to `tag:app-hub`, not to `autogroup:member`** — a deliberate
change from what this README said before, for two reasons:

- **`autogroup:member` lets every device you own publish to the public
  internet**, laptop included. One mistyped `tailscale funnel` command then
  exposes something you did not mean to expose. The tag confines the capability
  to this one host.
- **Tagged devices do not have key expiry.** `learn/34` records *"node keys
  expire at 180 days"* as a thing that will bite later: an untagged node drops
  off the tailnet roughly six months in and the public URL stops working with
  no change on your side. For a host whose entire job is to stay up, that is
  the more valuable half. Confirm it on the **Machines** page — a tagged node
  shows no expiry date.

**The cost of tagging, stated so it does not surprise you:** the auth key must
be generated **with that tag**, or the node comes up untagged, Funnel stays
refused, and the symptom is a public URL that never answers. See `.env.example`.

**Two different things control public exposure, and it is worth keeping them
straight.** This file decides whether the node *may* use Funnel at all;
`tailscale-serve.json` decides *which ports* are actually published. Funnel
fails closed if either is wrong, and the failure looks like a network problem
both times.

Skip this and the container starts, logs in, reports healthy, and is simply not
reachable from outside — a failure that reads as a networking problem and is a
permissions one.

**(d) Generate an auth key.** Settings → **Keys** → *Generate auth key*.
**Reusable**, **not ephemeral**: an ephemeral node is deleted when it goes
offline, and the hostname is what the URL is built from. Into `.env` as
`TS_AUTHKEY`.

**(e) Once it has joined, turn OFF key expiry for this node.** Machines → the
`app-hub` node → **Disable key expiry**. Node keys expire after 180 days by
default; when one does, the node drops off the tailnet and the dashboard goes
dark for no visible reason, half a year from now, long after anyone would
connect it to this step. An always-on host wants a non-expiring node key.

Your URL is then **`https://app-hub.<your-tailnet>.ts.net`**. The exact tailnet
name is in the admin console header, or:

```bash
docker compose exec tailscale tailscale status
```

---

### Funnel is public, and the URL is not a secret

`AllowFunnel` in `tailscale-serve.json` is what exposes this to the open
internet. **Anyone with the URL can load the dashboard — there is no login.**

And the URL is discoverable: Funnel serves a real Let's Encrypt certificate, and
every issued certificate is published to **Certificate Transparency logs**,
which are public and searchable. Treat the hostname as known, not hidden.

That matters here specifically, because **the gateway dashboard does not filter
on the `public` flag.** That flag governs `site/projects.json` — the Cloudflare
Pages site — and nothing else. This dashboard renders the DynamoDB catalogue
directly, which currently includes entries marked private: `Notes`,
`Acharya Amit Puri`, and a `localhost` URL.

**To make it tailnet-only instead**, which is arguably what a personal dashboard
wants, set `"AllowFunnel"` to `false` in `tailscale-serve.json`. It stays on the
same stable HTTPS hostname, still free, still no domain — but only devices
signed into your tailnet (your phone, your laptop) can reach it. The public face
of this project is already `site/` on Cloudflare Pages; **this host does not
also have to be the portfolio piece.**

**4. Fill in `.env`:**

```bash
cp .env.example .env
```

---

## Deploy

**The registry is durable as of 2026-09-18.** ECR used to live in the ephemeral Terraform stack, so the nightly `make down` deleted the repositories along with the cluster — which would have broken this host every night, with an error that reads like the login expiry below and is really a missing repository. The repositories are now in `infra/persistent/`, `prevent_destroy` is set, and `make down` no longer empties them.

ECR authentication expires **every 12 hours**, so this is not one-time:

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 314146298861.dkr.ecr.ap-south-1.amazonaws.com
```

```bash
docker compose pull && docker compose up -d
```

```bash
docker compose ps
```

All four `running`, the two with healthchecks `healthy`.

---

## Verify — and check the right things

**Locally first**, before trusting the tunnel:

```bash
docker compose exec gateway python -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8001/links',timeout=5).read()[:200])"
```

**Then confirm the same links appear as on EKS** — that is what the shared table
buys, and it is the claim worth testing rather than assuming:

```bash
aws dynamodb scan --table-name app-hub-links --region ap-south-1 --select COUNT --query Count --output text
```

**Then the public URL**, from a network that is not your home one — a phone on
mobile data is the honest test. A phone on your home wifi proves nothing:

```
https://app-hub.<your-tailnet>.ts.net
```

If it does not answer, check Funnel is actually enabled before suspecting
anything else — it is off by default per tailnet, and that is the usual cause:

```bash
docker compose exec tailscale tailscale funnel status
```

**Confirm nothing is listening on the host**, which is the point of the tunnel:

```bash
docker compose ps --format '{{.Service}}  {{.Ports}}'
```

Every line should show **no published port**. A `0.0.0.0:xxxx->` anywhere means
a `ports:` block crept in and the host is exposed directly.

---

## What differs from EKS, and what must not

**Must not differ:** the image. Same registry, same tag, same digest.

**Must differ — and this is the one that bites:**

```
EKS       LINKS_SERVICE_URL=http://links-service:80     Service maps 80 -> 8000
Compose   LINKS_SERVICE_URL=http://links-service:8000   no Service; direct to the container
```

The **same variable needs a different value per target**. That is the third
appearance of this bug — the 8000-vs-80 mix-up (2026-09-10) and `AGGREGATOR_URL`
missing entirely (`D-21`, 2026-09-13). `AGGREGATOR_URL` is `8002` in *both*,
because aggregator's Service maps 8002 → 8002. **Do not make them match out of
symmetry.**

**Must differ — credentials.** IRSA does not exist here. See above.

**Must differ — the edge.** ALB + Ingress there; **Tailscale Funnel** here. Both
are outbound-only in spirit: nothing listens on the host either way.

---

## Data: why the same table

A local store would mean **two divergent catalogues of the thing you use every
day** — add a link here, it is missing next time the cluster comes up. For a
learning cluster that is tolerable; for the dashboard you actually use it is
the product failing at its only job.

The cost is honest: **a long-lived AWS key on a host you patch yourself**,
where EKS had none. Contained by scoping the user to four actions on one table.

DynamoDB on-demand at this volume is **≈ $0**, and the `$3/day` budget guardrail
already watches it.

**This direction is also the cheaper one to reverse.** Shared now, adding a
local read cache later, is additive. Local now, merging later, means
reconciling two datasets with conflicting ids.

---

## n8n lives here now (`N-06`, re-targeted 2026-09-20)

`N-06` originally said *"move n8n onto the EKS cluster"*, decided 2026-08-30.
This host is the better answer, and the reasoning is worth keeping:

- EKS would need a **Postgres surviving the nightly destroy** — an RDS billing
  permanently, which breaks the `$0` resting state `FINISH-LINE.md`'s `O10`
  now makes a criterion.
- EKS would put the **cost watchdog on the cluster it watches**. Every
  `make down` would destroy the thing whose job is to say the cluster is up.
- EKS would need `N8N_ENCRYPTION_KEY` as a Kubernetes Secret. **Here that
  problem does not exist**: the key lives inside the `n8n_data` volume and is
  never typed, printed or passed anywhere.

### The admin UI is tailnet-only, and that is deliberate

| route | goes to | reachable by |
|---|---|---|
| `:443` | `gateway:8001` | **anyone** — Funnel, public |
| `:8443` | `n8n:5678` | **your tailnet only** — Serve, not in `AllowFunnel` |

n8n's UI holds every credential on the instance. It has no business on the
public internet, and the dashboard has no business being private. One file
expresses both.

**This is also a security improvement over the container it replaces**, which
published `5678` on `0.0.0.0` — reachable from anything on the local network.

### Migrating the existing instance — read before running

> **The `n8n_data` volume is the entire instance: workflows, credentials, and
> the encryption key that makes the credentials readable.** `docker-compose.yml`
> declares it `external: true` so Compose ATTACHES it rather than creating one.
> If that ever became a non-external volume, n8n would start blank, generate a
> fresh key, and **every stored credential would be permanently undecryptable.**

**1. Confirm the volume exists before touching anything.**

```bash
docker volume ls | grep n8n_data
```

**2. Stop the standalone container — do NOT remove the volume.**

```bash
docker stop n8n && docker rm n8n
```

`docker rm` removes the *container*. The named volume survives; that is the
whole point of it being named. Never pass `-v`.

**3. Bring it up under Compose.**

```bash
docker compose up -d n8n
```

**4. Prove the credentials still decrypt — do not just check it starts.**

A blank n8n also starts, and looks healthy. Open the UI and confirm a workflow
that uses a credential (`eks-cost-watchdog` uses SMTP) still shows it bound
rather than missing. **A workflow listing its credential as not-found is the
symptom of a lost encryption key**, and it is not recoverable afterwards.

```bash
docker compose exec n8n ls /home/node/.n8n
```

`config`, `database.sqlite` and `nodes/` should be present — the same files the
standalone container had.

---

## Keeping it running — the annoying parts

- **ECR login expires every 12 hours.** Any `docker compose pull` after that
  fails with an auth error that reads like a permissions problem.
- **OS patching is yours**, on a machine that is always on and reachable.
- **Nothing tells you a base image has a CVE.**
- **The laptop must stay awake** — and this project has just spent two days
  learning what sleep does to background services (`D-24`). Disable sleep on
  the host, or you will rediscover it.
- **Something must watch the watcher.** If this host goes down, nothing
  currently says so — and it cannot be this host that tells you.
- **Tailscale is free but is a dependency you do not control**, and the free
  Personal plan's terms are theirs to change.
- **The node key expires after 180 days unless you disable expiry**, and when
  it does the dashboard goes dark with no local cause. See step 3(e).
- **`docker compose down -v` changes your URL.** It destroys the state volume,
  the node re-registers, and because the old `app-hub` node still exists
  Tailscale appends a suffix — `app-hub-1`. Every bookmark breaks silently.
  Plain `docker compose down` is safe.
- **Funnel is public and its hostname is in Certificate Transparency logs.**
  See the note in step 3.

---

## Moving to Oracle later

Nothing here changes except the image architecture. Build a **multi-arch
manifest** rather than separate `-arm64` tags:

```bash
docker buildx build --platform linux/amd64,linux/arm64 --push -t <repo>:<sha> .
```

One tag serves both; EKS transparently selects `amd64`. **Decide this before
the first ARM push** — separate tags means two tag schemes forever, and
retagging retrospectively is miserable.

Note `make down`'s ECR emptying already loops until the repository reports zero,
because buildkit pushes an index plus children. Multi-arch means more children
per tag; the loop already handles it, the pass count just grows.
