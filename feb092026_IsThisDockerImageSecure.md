---

# 🛡️ Is This Docker Image Secure?

## What Started With Pi-hole Ended in Building My Own Vulnerability Management Pipeline

This whole thing started with a simple question ...

> “Is this Docker image secure? `ekofr/pihole-exporter`”

I was just trying to monitor Pi-hole metrics

Instead… I ended up building a full vulnerability management workflow using:

* 🐳 Docker
* 🔍 Trivy
* 📦 JSON reports
* 🛡️ DefectDojo

This post documents what I learned.

---

# 🔎 Step 1 — Don’t Trust, Inspect

First thing I did:

```bash
docker inspect ekofr/pihole-exporter
```

What I noticed:

* Image size: ~3.6MB (tiny)
* Very few layers
* No explicit `USER` → probably running as root
* Using `:latest` ⚠️

### Lesson #1:

Never blindly trust `latest`.

Even in a home lab.

Better:

```yaml
image: ekofr/pihole-exporter@sha256:<digest>
```

Now the image is immutable. No surprises if someone pushes a new `latest`.

---

# 🔍 Step 2 — Scan It (But Do It Safely)

Instead of mounting `docker.sock` (which is basically root access to your host), I exported the image as a tar and scanned it offline.

### Export image

```bash
docker save ekofr/pihole-exporter:latest -o /tmp/pihole-exporter.tar
```

### Scan with Trivy (Dockerized)

```bash
docker run --rm \
  -v /tmp:/work \
  -v ~/.cache/trivy:/root/.cache/ \
  aquasec/trivy:latest image \
  --severity HIGH,CRITICAL \
  --input /work/pihole-exporter.tar
```

Why this method?

| Method            | Risk              |
| ----------------- | ----------------- |
| Mount docker.sock | 🔴 High privilege |
| Scan exported tar | 🟢 Much safer     |

This way Trivy can't control my Docker daemon.

---

# 📦 Step 3 — Export Vulnerabilities to JSON

I wanted structured data.

```bash
docker run --rm \
  -v /tmp:/work \
  -v ~/.cache/trivy:/root/.cache/ \
  aquasec/trivy:latest image \
  -f json \
  -o /work/pihole-exporter-report.json \
  --input /work/pihole-exporter.tar
```

Now I had a machine-readable vulnerability report.

This includes:

* CVE
* Package
* Installed version
* Fixed version
* Severity
* Description

Now we're getting serious.

---

# 🏗️ Step 4 — Deploy DefectDojo in Docker

If I'm scanning containers… I want lifecycle tracking.

So I deployed DefectDojo:

```bash
git clone https://github.com/DefectDojo/django-DefectDojo.git
cd django-DefectDojo
docker compose up -d
```

Changed port via `.env`:

```bash
DD_PORT=8091
DD_TLS_PORT=9443
```

Restarted:

```bash
docker compose down
docker compose up -d
```

Now I had a vulnerability management platform running locally.

That escalated quickly 😄

---

# 🔐 Step 5 — API Token

Inside DefectDojo:

* Click username
* Profile
* Generate API v2 key

Export it:

```bash
export DD_TOKEN="your_token_here"
```

---

# 📁 Step 6 — Create Product + Engagement

Structure matters.

I created:

**Product:** Monitoring Stack
**Engagement:** Container Scan – Feb 2026

The engagement ID comes from the URL:

```
/engagement/3/overview
```

ID = 3

---

# 🚀 Step 7 — Import Trivy Scan via API

```bash
curl -X POST "http://localhost:8091/api/v2/import-scan/" \
  -H "Authorization: Token $DD_TOKEN" \
  -F "scan_type=Trivy Scan" \
  -F "file=@/tmp/pihole-exporter-report.json" \
  -F "engagement=3" \
  -F "active=true" \
  -F "verified=true"
```

Boom 💥

Findings showed up inside DefectDojo.

---

# 🧠 Big Realization

Most of the findings were Debian OS package CVEs.

But here's the important thing:

> Vulnerable package ≠ exploitable vulnerability

Questions to ask:

* Is the service exposed?
* Is there a real attack path?
* Is it running as root?
* Can an attacker reach it?

Vulnerability management is not panic management.

It’s risk management.

---

# 📊 What I Actually Built

Without planning to, I built:

Container → Trivy → JSON → DefectDojo → Governance

That’s basically enterprise vulnerability management in a home lab.

---

# 🎯 What I Learned

* Don’t trust `latest`
* Always inspect images
* Prefer offline scanning when possible
* Avoid mounting docker.sock unless necessary
* Centralize vulnerability tracking
* Context matters more than CVE count

---

# 🔮 What’s Next?

Possible improvements:

* Nightly automated scans
* SLA tracking
* Deduplication
* Alerting on new Critical
* Grafana dashboard for open CVEs

---

# 🏁 Final Thoughts

I started by asking:

> “Is this Docker image secure?”

And ended up building a mini security program.

Security isn’t just tools.

It’s architecture, workflow, and context.

And honestly… this is the fun part 😄

---

If this helps you in your own lab, feel free to fork or adapt.

Keep building.
