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

**1. Create the GitHub repo.** `HarshitRawat11/app-hub-compose`, then:

```bash
wsl -e bash -lc "cd /mnt/c/Users/harshit.rawat/Documents/Projects/app-hub/compose && git remote add origin git@github.com:HarshitRawat11/app-hub-compose.git"
```

Git identity is already set **locally** in this repo — this is a work-managed
laptop and personal commits must not carry the work identity.

**2. Create the IAM user.** This is the one genuinely new AWS object, and it is
the security decision worth getting right.

Policy: exactly the four DynamoDB actions the repository layer performs, on the
one table, plus read-only ECR pull:

| Action | Resource |
|---|---|
| `dynamodb:GetItem`, `PutItem`, `DeleteItem`, `Scan` | the `app-hub-links` table ARN only |
| `ecr:GetAuthorizationToken` | `*` — account-level by the shape of the API |
| `ecr:BatchGetImage`, `GetDownloadUrlForLayer`, `BatchCheckLayerAvailability` | the three app-hub repository ARNs |

**Create the access key by hand, not in Terraform.** `aws_iam_access_key` puts
the secret into Terraform state — a long-lived credential in S3, readable by
anything that can read state. The user and policy are fine in Terraform; the
key is not.

**3. Create the Cloudflare Tunnel.** Zero Trust dashboard → Networks → Tunnels.
Copy the token into `.env`. Route the hostname to **`http://gateway:8001`** —
cloudflared shares the Compose network, so it addresses gateway by service name.

**4. Fill in `.env`:**

```bash
cp .env.example .env
```

---

## Deploy

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

**Then the tunnel**, from a network that is not your home one — a phone on
mobile data is the honest test:

```
https://<your-tunnel-hostname>/
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

**Must differ — the edge.** ALB + Ingress there; Cloudflare Tunnel here.

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
- **Cloudflare Tunnel is free but is a dependency you do not control.**

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
