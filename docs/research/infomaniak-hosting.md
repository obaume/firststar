# What Infomaniak offers for Django + React

Research for issue #4. Question: which Infomaniak products fit a Django API plus a Vite React app? It covers compute, managed Postgres, object storage for headshots and Share Images, background jobs and scheduled tasks, email sending, Swiss data residency, and rough pilot costs.

- **Prices read:** 2026-10-11. All prices are in CHF, **excluding Swiss VAT** ("Prices do not include VAT" appears on every product page).
- **Monthly figures:** Hourly rates are converted at 730 h/month, the same convention Infomaniak uses in its own comparison ([FAQ 2626](https://www.infomaniak.com/en/support/faq/2626/public-cloud-comparison-of-offers)).
- **Where the cloud prices come from:** The Public Cloud, Database Service, Kubernetes and Jelastic price tables are drawn client-side. I read them from the JSON files behind infomaniak.com's own pricing pages, listed under [Sources](#sources). For example, the flavour `a4_ram8_disk50_perf1` = CHF 0.02204/h → CHF 16.09/month matches the "CHF 16.10 / month" shown in text on the [Public Cloud prices page](https://www.infomaniak.com/en/hosting/public-cloud/prices).

## TL;DR

- **Compute.** Django needs root access. Infomaniak's own Django guide applies to **VPS Cloud / VPS Lite** only. It says "shared hosting and Managed Cloud Server are not supported" ([FAQ 516](https://www.infomaniak.com/en/support/faq/516/install-django)). Shared web hosting can't run Python at all ([FAQ 1307](https://www.infomaniak.com/en/support/faq/1307/use-a-perl-or-python-script)). Realistic options:
  - **VPS Lite:** cheapest, about CHF 7/month.
  - **Public Cloud (OpenStack) instance:** about CHF 8–16/month, billed hourly, CHF 300 trial credit.
  - **Jelastic Cloud (PaaS):** roughly CHF 30–45/month for a small setup.
  - **Managed Kubernetes:** possible but overkill for a pilot.
- **Managed Postgres: not available today.** The Database Service lists **MySQL only**. PostgreSQL (and Redis) are marked "Available soon". Postgres must be self-run, either in a container or VM on VPS or Public Cloud, or as a Jelastic PostgreSQL node.
- **Object storage:** Public Cloud Object Storage (OpenStack Swift with an S3-compatible API) costs about **CHF 0.011/GiB/month**. 10 TB/month of egress is included. Data stays in Switzerland. It is S3-*compatible*, not S3, so check the quirks listed below.
- **Background jobs / scheduling:** There is no managed queue or scheduler product. Run Celery, RQ or django-q2 plus cron or systemd timers yourself. Jelastic has per-container cron. Kubernetes has CronJobs.
- **Email:** There is **no transactional-email product**. Mail Service (from CHF 2.29/month) gives authenticated SMTP, capped at **200 (Starter) / 500 (Premium) recipients per rolling 24 h**. That is fine for admin and password mails, not for mailing every fan.
- **Swiss residency:** Every product looked at states it is hosted in Infomaniak's own Swiss data centers.
- **Pilot cost:** about **CHF 12–20/month** excl. VAT for a single VM + object storage + Mail Service. About **CHF 35–50/month** if you want PaaS (Jelastic) or an SLA-backed VPS Cloud.

## Load assumption

Hundreds to low thousands of fans vote within a few minutes after a game ends. 2,000 votes in 5 minutes is about 7 requests/s on average. Assume spikes of a few tens of requests/s, plus page and asset loads. The React SPA is static, so serve it from nginx or object storage. My estimate, not a vendor claim: one 2 vCPU / 4 GB VM running gunicorn and Postgres comfortably handles this pilot load. Size up for the first real game and measure.

## Compute options

| Product | What it is | Python/Django | Price (excl. VAT) | Notes |
|---|---|---|---|---|
| **Web hosting** (shared) | Shared PHP hosting | **No.** "running Perl and Python scripts is not supported" ([FAQ 1307](https://www.infomaniak.com/en/support/faq/1307/use-a-perl-or-python-script)) | from CHF 10.91/mo ([page](https://www.infomaniak.com/en/hosting/web-hosting)) | Out. |
| **Managed Cloud Server** | Managed server with Apache/PHP and ~40 add-on technologies (Node.js, Redis, …) | Django guide says Managed Cloud Server is **not supported** ([FAQ 516](https://www.infomaniak.com/en/support/faq/516/install-django)). Apps can listen on ports 4000–4009 behind HAProxy ([FAQ 2171](https://www.infomaniak.com/en/support/faq/2171/redirect-web-traffic-to-a-specific-port)). | from CHF 38.41/mo ([page](https://www.infomaniak.com/en/hosting/managed-cloud-server)) | PHP-oriented. Poor fit. |
| **VPS Lite** | Unmanaged Linux VM, root access | Yes (Django guide applies) | 1 vCPU/2 GB/20 GB **CHF 2.70**; 2/2/40 **5.40**; 2/4/60 **7.20**; 4/4/80 **9.00**; 4/8/160 **18.00** per month, **with annual commitment** ([page](https://www.infomaniak.com/en/hosting/vps-lite)) | 500 Mbit/s. **No availability guarantee, no snapshots.** Resources can only be increased. Outbound port 25 blocked (SMTP submission on 587/465 is unaffected). Unmanaged: no OS/app support. |
| **VPS Cloud** | Unmanaged VM on the same OpenStack infrastructure | Yes | from **CHF 38.41/mo** (from 4 vCPU / 12 GB / 250 GB NVMe, up to 32 vCPU / 128 GB) ([prices](https://www.infomaniak.com/en/hosting/vps-cloud/prices)) | **99.99 % availability** commitment, 1 snapshot included, scale up/down, monthly/quarterly/annual billing. |
| **Public Cloud** (OpenStack) | Self-service IaaS: instances, block storage, object storage, LBs, DNS, K8s, DBaaS | Yes (any Linux image) | Hourly. E.g. `a1_ram2_disk20_perf1` CHF 0.0064/h (**≈4.67/mo**); `a2_ram4_disk20_perf1` 0.01043/h (**≈7.61/mo**); `a2_ram4_disk50_perf1` 0.01399/h (**≈10.21/mo**); `a4_ram8_disk50_perf1` 0.02204/h (**≈16.09/mo**). Floating IPv4 CHF 0.00457/h (**≈3.34/mo**). Load balancer 0.0137/h (**≈10.00/mo**). Block storage (perf1) CHF 0.00012/GiB/h (**≈0.088/GiB/mo**). | **CHF 300 free credit for 3 months** (needs credit card + phone). Pay monthly in arrears, card only. Bandwidth free (object-storage egress above 10 TB billed). Stopped instances aren't billed except storage, GPU and public IPv4 ([prices page FAQ](https://www.infomaniak.com/en/hosting/public-cloud/prices)). Terraform/Ansible supported. |
| **Jelastic Cloud** (PaaS, Virtuozzo) | Containers (Docker, Python, Node.js…), databases incl. **PostgreSQL**, load balancers, autoscaling | Yes: "Java, PHP, Ruby, Node.js, Python, and Go", Apache Python app server, or Docker images ([page](https://www.infomaniak.com/en/hosting/dedicated-and-cloud-servers/jelastic-cloud)) | Per hour per **cloudlet** (1 cloudlet = 128 MB RAM + 400 MHz CPU), tiered by count: 1–3 cloudlets CHF 0.003256/h each (≈2.38/mo); 4–8: 0.002541; 9–12: 0.002167; 13–16: 0.002024; 17–64: 0.001881 (≈1.37/mo); … 257+: 0.001386. Public IP CHF 0.00539/h (≈3.93/mo). Storage CHF 0.00011/GB/h beyond 20 GB. Page example: 1 container, 1 cloudlet, 20 GB, IP = **CHF 6.31/mo**. | 14-day free trial. Prepaid credit. Reserved vs dynamic (burst) cloudlets. "Exclusively hosted in Infomaniak's data centers in Switzerland". 99.99 % availability claim. 2.8 GB external traffic per hour included. |
| **Managed Kubernetes** (Public Cloud) | Managed control plane, workers are Public Cloud instances | Yes (containers) | Control plane: **Shared = free** (no SLA, up to 10 nodes); Dedicated 4 CHF 0.04/h (≈29.20/mo, 99.9 % SLA); Dedicated 8: 0.08/h; Dedicated 16: 0.16/h ([page](https://www.infomaniak.com/en/hosting/public-cloud/kubernetes)). Plus worker instances, LB, volumes at Public Cloud rates. | Cilium, choice of ingress, node-pool autoscaling. Overkill for one pilot club. Realistic floor ≈ 2 workers + LB ≈ CHF 25–35/mo. |

### Notes on Jelastic pricing

- The tier boundaries and rates above come from the JSON behind the Jelastic configurator on infomaniak.com. The configurator applies the tier to the total cloudlet count of the configuration.
- **Not verified:** The page does not publish the price of *dynamic* cloudlets separately from reserved ones. It also doesn't publish how tiers apply across several environments. Use the 14-day trial to see real billing.
- Rough pilot estimate: Django app (~8 cloudlets ≈ 1 GB), PostgreSQL (~8 cloudlets), and a worker (~4 cloudlets). That is ≈20 cloudlets × CHF 0.001881 × 730 ≈ CHF 27, plus IP CHF 3.93 and some storage, so **≈ CHF 30–45/month** with burst headroom.

## Managed Postgres

[Database Service](https://www.infomaniak.com/en/hosting/public-cloud/database) (Public Cloud DBaaS):

- **MySQL** is the only engine on sale. MariaDB, **PostgreSQL**, OpenSearch and **Redis** are all labelled "Available soon". The underlying price data only contains `"type":"mysql"` plans.
- MySQL plans (for reference):
  - Essential (1 node, no SLA) `essential-db-4`: 1 vCPU / 4 GB / 80 GB, CHF 0.02965/h ≈ **21.64/mo**
  - Business (2 nodes, multi-AZ, 99.9 % SLA) `business-db-4`: CHF 0.05622/h ≈ **41.04/mo**
  - Advanced (3 nodes): CHF 0.08278/h ≈ **60.43/mo**
  - Backups from CHF 0.000015/GB/h.
  - Features: automated backups, point-in-time recovery, TLS, Terraform/API.

Postgres options today:

1. **Self-run Postgres** on the same VM as Django (pilot) or on its own Public Cloud instance with a block-storage volume. Add your own backups, for example `pg_dump` / WAL-G to Object Storage.
2. **Jelastic PostgreSQL node.** Jelastic lists "Postgre" among supported databases ([page](https://www.infomaniak.com/en/hosting/dedicated-and-cloud-servers/jelastic-cloud)). It is provisioned for you, but you still own tuning and backups.
3. Wait for DBaaS PostgreSQL. There is **no published date**, so this is unverifiable.

Django runs on MySQL too. Switching DB engine just to get a managed DB is an option, but that is a decision for the stack ticket, not this research.

## Object storage (headshots, Share Images, logos)

- **Product:** Public Cloud **Object Storage**, OpenStack Swift with an S3-compatible API ([docs](https://docs.infomaniak.cloud/object_storage/), [S3 docs](https://docs.infomaniak.cloud/object_storage/s3/)).
- **Price:** storage CHF 0.000015/GiB/h ≈ **CHF 0.011/GiB/month** (≈ CHF 11/TiB/month). Egress is free up to **10 TB per month per organisation**, then CHF 0.009/GiB ([prices page](https://www.infomaniak.com/en/hosting/public-cloud/prices)).
- **Pilot scale:** a few GB of headshots and generated PNGs cost well under CHF 1/month.
- **Endpoints:** `https://s3.pub1.infomaniak.cloud/` and `https://s3.pub2.infomaniak.cloud/`. Use region `us-east-1` (a compatibility value only). "all data is located in our data centers in Switzerland" ([S3 docs](https://docs.infomaniak.cloud/object_storage/s3/)).
- **Quirks to plan for** (from the S3 docs page):
  - "Swift is only S3-compatible. Some features are not available."
  - Use path-style addressing (`forcePathStyle=true`). In django-storages that means `addressing_style = "path"`.
  - AWS CLI ≥ 1.37 forces `aws-chunked` content encoding, "which is not supported by Swift". **Inference, not verified:** recent boto3 versions have the same default-checksum behaviour. With django-storages/boto3, expect to set `request_checksum_calculation="when_required"` (env `AWS_REQUEST_CHECKSUM_CALCULATION=when_required`). Test uploads early.
  - Multipart overwrite leaves old segments in a `+segments` bucket that need manual cleanup.
  - Bucket `CreationDate` is always `2009-02-03T16:45:09.000Z`.
- **Public read vs presigned URLs:** Public access is possible through Swift container ACLs ([docs](https://docs.infomaniak.cloud/object_storage/)). **Not verified:** the S3 docs page does not say whether S3 presigned URLs are supported. Headshots and Share Images are meant to be public, so a public-read container is enough. Check presigned URLs if private uploads are needed.
- **Other products:** Swiss Backup and kDrive are backup and file-sync products, not app object storage. Out of scope.

## Background jobs and scheduled tasks

Needed: vote auto-open/close and Share Image generation. Infomaniak has **no managed queue, worker or scheduler service** in any product I checked. Managed Redis is "Available soon" on DBaaS.

- **VPS / Public Cloud:** run them yourself. Django + Celery/RQ + Redis, or a DB-backed queue (django-q2, Procrastinate on Postgres) to avoid Redis. Trigger scheduled vote open/close with Celery beat, cron or a systemd timer. Docker Compose on one VM is enough for the pilot.
- **Jelastic:** per-node cron. The platform docs say: "The platform gives you a possibility to run your programs at specified time with a help of cron" ([Virtuozzo docs](https://www.virtuozzo.com/application-platform-docs/cron-job/)). Workers are extra containers.
- **Kubernetes:** standard Deployments for workers and `CronJob` for schedules.
- **Managed Cloud Server:** has cron, but Django isn't supported there (see above).

Design note: time-based open/close is more robust if it is stored as `opens_at` / `closes_at` and checked at request time. The scheduler then only handles side effects such as image generation and notifications. That lowers the dependency on the job runner being up at the exact second.

## Email sending

- **No transactional-email or API product.** Apps send through authenticated SMTP on `mail.infomaniak.com` (587 STARTTLS / 465 TLS) using a Mail Service or kSuite mailbox.
- **Mail Service:** from **CHF 2.29/month** ([prices](https://www.infomaniak.com/en/ksuite/service-mail/prices)).
- **Sending limits** per address, rolling 24 h, **counted per recipient** ([FAQ 2065](https://www.infomaniak.com/en/support/faq/2065/understanding-the-limits-on-outgoing-emails-per-24-hours), updated 09/14/2026):
  - Mail Service Starter: **200**
  - Mail Service Premium: **500**
  - kSuite Standard/Business/Enterprise: 500
  - Web hosting unauthenticated PHP `mail()`: 50
- VPS Cloud, Managed Cloud Server and web hosting include "2500 emails sent / month free" for the **Newsletter** tool. That is for marketing campaigns, not transactional mail.
- Running your own mail server on VPS Lite isn't possible (outbound port 25 blocked). VPS Cloud opens port 25 "upon justified request". Self-hosted mail is not advisable anyway.
- **Implication:** Infomaniak SMTP covers staff and admin mail (invites, password resets, login links for club admins). If Firststar later emails fans (vote reminders, result notifications), 500/day is too low. Use a Swiss/EU transactional provider at that point. That choice is outside this ticket.

## Swiss data residency

- **VPS Cloud:** "TIER III+ and located in Switzerland … ISO 27001, ISO 9001 certified" ([prices](https://www.infomaniak.com/en/hosting/vps-cloud/prices)).
- **VPS Lite:** "Hosted in Switzerland" ([page](https://www.infomaniak.com/en/hosting/vps-lite)).
- **Jelastic:** "exclusively hosted in Infomaniak's data centers in Switzerland. Your data remains under Swiss jurisdiction, in full compliance with the FADP and GDPR" ([page](https://www.infomaniak.com/en/hosting/dedicated-and-cloud-servers/jelastic-cloud)).
- **Public Cloud / DBaaS / Kubernetes:** "Managed exclusively in Switzerland". The pages also say "Infomaniak operates its own data centers in Switzerland" ([DBaaS](https://www.infomaniak.com/en/hosting/public-cloud/database), [K8s](https://www.infomaniak.com/en/hosting/public-cloud/kubernetes)).
- **Object Storage:** "all data is located in our data centers in Switzerland" ([S3 docs](https://docs.infomaniak.cloud/object_storage/s3/)).
- **Not covered here:** I did not review the DPA/contract terms or sub-processor list. Do that before handling personal data from fans. The infomaniak.com footer links a "Trust center" and "Legal documents".

## Pilot cost scenarios (CHF/month, excl. VAT)

| Scenario | Components | ≈ CHF/mo |
|---|---|---|
| **A. Cheapest: VPS Lite** | VPS Lite 2 vCPU/4 GB/60 GB (7.20, annual) + Object Storage (<1) + Mail Service (2.29) | **≈ 10–11** |
| **B. Recommended for pilot: Public Cloud single VM** | `a2_ram4_disk50_perf1` (10.21) + floating IPv4 (3.34) + 20 GiB volume for Postgres data (≈1.75) + Object Storage (<1, also used for DB backups) + Mail Service (2.29) | **≈ 18** (covered by the CHF 300 trial credit for 3 months) |
| **C. PaaS: Jelastic** | ~20 cloudlets (app + Postgres + worker) + IP + Object Storage + Mail Service | **≈ 35–50** |
| **D. SLA-backed VM: VPS Cloud** | VPS Cloud 4 vCPU/12 GB (38.41) + Object Storage + Mail Service | **≈ 42** |
| **E. Managed K8s** | Shared control plane (0) + 2× `a2_ram4_disk20_perf1` (15.22) + LB (10.00) + volumes + self-run Postgres + Object Storage + Mail | **≈ 30–40**, much more ops work |

Why B over A for the pilot:

- Hourly billing with no annual lock-in.
- Snapshots and backups exist; VPS Lite has none.
- Block volumes survive instance rebuilds.
- Same platform as Object Storage, the future DBaaS PostgreSQL and Kubernetes.
- The CHF 300 trial covers the whole pilot season start.

A stays a valid low-cost fallback. Jelastic (C) is the pick if the team would rather not administer a Linux VM. It is still not truly "managed Postgres".

## Unverified / open points

- **DBaaS PostgreSQL:** "Available soon", with no date published.
- **Jelastic:** dynamic-cloudlet price and multi-environment tiering are not on the page.
- **Object Storage:** support for S3 presigned URLs and bucket policies is not documented on the S3 page. Check against the linked OpenStack S3/Swift compatibility matrix or by testing.
- **boto3 checksum / `aws-chunked` issue** with django-storages: inferred from the AWS CLI warning, not tested.
- **Price-data dates:** the Public Cloud, Jelastic, DBaaS and Kubernetes prices come from the JSON behind infomaniak.com's pricing pages. Their build date is unknown. They were read on 2026-10-11 and cross-checked against one visible on-page figure (CHF 16.10).
- **Sending limits:** older Infomaniak FAQs and terms quoted different numbers (e.g. 100 or 1,440/day for some Mail Service tiers). The figures above come from FAQ 2065 (updated 09/14/2026), which is the most recent.

## Sources

All read 2026-10-11.

- Product pages (infomaniak.com):
  - [VPS Lite](https://www.infomaniak.com/en/hosting/vps-lite)
  - [VPS Cloud](https://www.infomaniak.com/en/hosting/vps-cloud) and its [prices](https://www.infomaniak.com/en/hosting/vps-cloud/prices)
  - [Managed Cloud Server](https://www.infomaniak.com/en/hosting/managed-cloud-server)
  - [Web hosting](https://www.infomaniak.com/en/hosting/web-hosting)
  - [Jelastic Cloud](https://www.infomaniak.com/en/hosting/dedicated-and-cloud-servers/jelastic-cloud)
  - [Public Cloud](https://www.infomaniak.com/en/hosting/public-cloud) and its [prices](https://www.infomaniak.com/en/hosting/public-cloud/prices)
  - [Kubernetes Service](https://www.infomaniak.com/en/hosting/public-cloud/kubernetes)
  - [Database Service](https://www.infomaniak.com/en/hosting/public-cloud/database)
  - [Mail Service prices](https://www.infomaniak.com/en/ksuite/service-mail/prices)
- Price data behind the pricing pages (Gatsby static-query JSON on infomaniak.com):
  - Public Cloud: https://www.infomaniak.com/g/page-data/sq/d/3538198066.json
  - DBaaS: https://www.infomaniak.com/g/page-data/sq/d/461866520.json
  - Kubernetes: https://www.infomaniak.com/g/page-data/sq/d/55375645.json
  - Jelastic: https://www.infomaniak.com/g/page-data/sq/d/3331974153.json
- Infomaniak FAQ:
  - [516 Install Django](https://www.infomaniak.com/en/support/faq/516/install-django)
  - [1307 Perl/Python scripts](https://www.infomaniak.com/en/support/faq/1307/use-a-perl-or-python-script)
  - [2171 Ports 4000–4009 / HAProxy](https://www.infomaniak.com/en/support/faq/2171/redirect-web-traffic-to-a-specific-port)
  - [2065 Outgoing email limits](https://www.infomaniak.com/en/support/faq/2065/understanding-the-limits-on-outgoing-emails-per-24-hours)
  - [2626 Public Cloud comparison](https://www.infomaniak.com/en/support/faq/2626/public-cloud-comparison-of-offers) (prices dated 2021, used only to cross-check)
- Infomaniak Public Cloud docs:
  - [Object Storage](https://docs.infomaniak.cloud/object_storage/)
  - [S3 compatibility](https://docs.infomaniak.cloud/object_storage/s3/)
  - [Billing/metering](https://docs.infomaniak.cloud/metering/billing/)
- Jelastic/Virtuozzo platform docs: [Setting Up Cronjob](https://www.virtuozzo.com/application-platform-docs/cron-job/)
