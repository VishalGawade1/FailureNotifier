# Failure Informer

> A tiny, teachable **Kubernetes controller** that **watches Pods and sends email notifications when they fail**. Built as part of workshop materials; the repository also includes an unrelated sample (`application-scaler`), while the failure notifier lives under `notifier/`.

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Repository Layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Quickstart](#quickstart)
  - [Run on a local kind cluster](#run-on-a-local-kind-cluster)
  - [Trigger a failure to test notifications](#trigger-a-failure-to-test-notifications)
- [Operational Notes](#operational-notes)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Roadmap / Ideas](#roadmap--ideas)
- [License](#license)

---

## Overview

**Failure Informer** subscribes to Kubernetes Pod events and detects terminal failure conditions (for example, a Pod reaching `Failed` phase or crashing repeatedly). When a failure is observed, it formats a message with context (namespace, pod name, reason/message, timestamps) and **sends an email** to a configured recipient list via SMTP. The goal is to provide a compact, readable codebase you can learn from and adapt.

---

## How It Works

At a high level:

1. The controller (built on typical Kubebuilder/controller-runtime patterns) runs in-cluster with RBAC that allows it to **list/watch/get Pods**.
2. An informer/cache watches Pod add/update events. When a Pod transitions to a failure condition (or crosses a restart threshold), the controller enqueues a reconcile.
3. The reconcile logic extracts failure details (phase, container statuses, termination messages), renders an email, and **sends it via SMTP** using credentials you provide in environment variables/Secrets.
4. Optional filters let you **scope** which namespaces are monitored to reduce noise.

---

## Repository Layout

This repository contains two examples; the notifier you want is in `notifier/`:

```
.
├─ application-scaler/   # separate workshop sample (not part of the notifier)
└─ notifier/             # <<< failure-informer controller
```

There is a top-level `README.md` and a `LICENSE` (GPL-3.0).

---

## Prerequisites

- **Go** (for local builds)  
- **Docker** (to build/publish the image)  
- **kubectl** (v1.24+ recommended)  
- A **Kubernetes** cluster (kind, minikube, k3d, or managed)  
- An **SMTP account** capable of sending email (test mailbox is fine)

> The notifier often follows Kubebuilder-style make targets and deployment structure (e.g., `make docker-build`, `make deploy`). Adjust to the exact Makefile/paths in this repo.

---

## Configuration

Provide configuration via environment variables on the controller Deployment (recommend using Kubernetes **Secrets** for credentials):

| Variable             | Required | Example                       | Purpose |
|----------------------|:--------:|-------------------------------|---------|
| `SMTP_HOST`          |   ✅     | `smtp.example.com`            | SMTP server host |
| `SMTP_PORT`          |   ✅     | `587`                         | SMTP server port (STARTTLS) |
| `SMTP_USERNAME`      |   ✅     | `alerts@example.com`          | SMTP username (put in Secret) |
| `SMTP_PASSWORD`      |   ✅     | `s3cr3t`                      | SMTP password (put in Secret) |
| `MAIL_FROM`          |   ✅     | `alerts@example.com`          | Sender email |
| `MAIL_TO`            |   ✅     | `team-oncall@example.com`     | Comma-separated recipients |
| `WATCH_NAMESPACES`   |   ❌     | `apps,platform`               | Limit monitoring to listed namespaces; default: all |
| `MIN_RESTARTS`       |   ❌     | `3`                           | Minimum restarts before alerting (noise reduction) |
| `LOG_LEVEL`          |   ❌     | `info`                        | `debug` / `info` / `warn` / `error` |

> Tip: Include a container’s **termination message** in your email template when available for quick diagnosis.

---

## Quickstart

Clone the repo and switch to the notifier:

```bash
git clone https://github.com/Danil-Grigorev/failure-informer.git
cd failure-informer/notifier
```

### Run on a local kind cluster

1) **Create a cluster**

```bash
kind create cluster --name failure-informer
```

2) **Build the controller image**

If Kubebuilder-style `Makefile` targets are present:

```bash
# Build the image (adjust name as you like)
make docker-build IMG=failure-informer:dev
# Load into kind so the cluster can pull it
kind load docker-image failure-informer:dev --name failure-informer
```

(If there’s no `make` target, use `docker build -t failure-informer:dev .`.)

3) **Create a namespace and SMTP Secret**

```bash
kubectl create ns failure-informer
kubectl -n failure-informer create secret generic smtp-credentials   --from-literal=SMTP_USERNAME='alerts@example.com'   --from-literal=SMTP_PASSWORD='s3cr3t'
```

4) **Deploy the controller**

Common Kubebuilder flows:

```bash
# Install RBAC/manager manifests and deploy
make deploy IMG=failure-informer:dev
```

(If the repo includes raw manifests under `config/` or similar, you can `kubectl apply -n failure-informer -f` them directly.)

5) **Inject configuration into the Deployment**

```bash
kubectl -n failure-informer set env deploy/failure-informer   SMTP_HOST='smtp.example.com'   SMTP_PORT='587'   MAIL_FROM='alerts@example.com'   MAIL_TO='team-oncall@example.com'   WATCH_NAMESPACES='apps,platform'   MIN_RESTARTS='3'

kubectl -n failure-informer set env deploy/failure-informer --from=secret/smtp-credentials
```

6) **Verify it’s running**

```bash
kubectl -n failure-informer get pods
kubectl -n failure-informer logs deploy/failure-informer -f
```

### Trigger a failure to test notifications

Create an intentionally failing Pod in one of the watched namespaces:

```yaml
# failing-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-me
  namespace: apps
spec:
  containers:
  - name: boom
    image: busybox
    command: ["sh", "-c", "echo 'Going to crash' && exit 1"]
  restartPolicy: Always
```

Apply it and watch:

```bash
kubectl apply -f failing-pod.yaml
# in another terminal
kubectl -n failure-informer logs deploy/failure-informer -f
```

You should see the controller detect the failure and send an email to `MAIL_TO`.

---

## Operational Notes

- **Scoping**: Narrow the blast radius with `WATCH_NAMESPACES` and/or label selectors to reduce noise in large clusters.  
- **Noise control**: Tune `MIN_RESTARTS` (and any internal debounce/backoff) so that flapping pods don’t spam inboxes.  
- **Email content**: Include namespace, pod name, failing container(s), restart counts, termination reason/message, and hints to run `kubectl describe pod` and `kubectl logs`.  
- **RBAC**: The controller needs `get`, `list`, `watch` on Pods cluster-wide or in the watched namespaces only.  
- **SRE integration**: It’s straightforward to add alternative sinks (Slack, PagerDuty) if your org prefers chat/on-call tooling.

---

## Troubleshooting

- **No emails received**
  - Check controller logs for SMTP connection/auth errors.
  - Verify network egress to your SMTP host (some clusters block it).
  - Some providers require STARTTLS on **587** or explicit TLS on **465**; confirm port and security mode.

- **Too many alerts**
  - Increase `MIN_RESTARTS`.
  - Limit `WATCH_NAMESPACES` to teams/apps that need paging.

- **Controller isn’t reconciling**
  - Confirm RBAC is applied.
  - Ensure the manager Pod is `Ready`.
  - If you changed namespace scoping, confirm the Deployment’s permissions match.

- **Image pull errors in kind**
  - Ensure you `kind load docker-image ...` after each rebuild, or push to a registry the cluster can reach.

- **Stale informer / watch issues**
  - Extremely long network outages can interrupt watches in some client libraries; ensure reconnect/backoff strategies or restarts are in place.

---

## Security Notes

- Store `SMTP_USERNAME`/`SMTP_PASSWORD` in a **Secret**; never in plain manifests or ConfigMaps.  
- Limit RBAC to the least privileges required (typically read-only Pod access).  
- Avoid leaking sensitive contents in logs or emails (for example, redact environment variables or secrets mounted into containers).  
- If sending email through a corporate provider, use app passwords or service accounts as required by your org’s policy.

---

## Roadmap / Ideas

- Additional sinks: **Slack**, **PagerDuty**, **Opsgenie**.  
- Attach recent pod **events/log excerpts** to the email.  
- Support **label/annotation filters** (only alert on selected services).  
- Implement **deduplication** (group repeated failures) and **exponential backoff**.  
- Provide a **Helm chart** for one-command installs.

---

