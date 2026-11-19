---
title: "Server as Code — Provisioning and Auditing My Homelab with Ansible"
date: 2026-11-19 23:00:00 +0900
categories: [개발, DevOps]
tags: [Ansible, K3s, GitOps, IaC, Security, Homelab]
author: L.J
lang: en
---

### **Intro**

This post documents how I turned the entire state of my homelab server (soominlab.com) into Ansible code. Two goals:

1. If I ever move to a new server (cheaper hosting, or AWS), a single `ansible-playbook site.yml` should bring up the exact same stack — k3s, nodeport-guard, tailscale serve, kube-prometheus-stack, and a weekly security audit.
2. Prevent a repeat of a past incident where a hosting provider's forced shutdown left missing services and an **unexpectedly exposed port** — now enforced by a server-side weekly audit that stays silent unless something is wrong.

---

## 1. What is Ansible?

Ansible is a tool that **declares the desired state of servers as code (YAML) and reconciles the actual state to match**.

- No agent to install, unlike Chef/Puppet — SSH access is enough.
- You don't write *procedures* ("run apt install curl"); you write *states* ("curl must be present").
- Running the same playbook any number of times yields the same result — **idempotency**. If the state is already correct, nothing changes. That property is the heart of this post.

Three concepts are enough to get started:

| Concept | Role |
|---|---|
| **Inventory** | List of managed servers (`inventory/hosts.yml`) |
| **Playbook** | "Apply these roles to this server" (`site.yml`) |
| **Role** | A reusable bundle of state (k3s, firewall, monitoring, ...) |

---

## 2. Why Ansible — Two Incidents and One Plan

### Incident: what a hosting provider's shutdown left behind

In the past, my hosting provider force-shutdown and rebooted the server. Some services came back missing, and a Docker container ended up **exposing an unintended port to the public internet**. A dead service behind an open port is the worst combination.

Since then I built two layers of defense on this server (covered in a previous post):

- **nodeport-guard**: an iptables rule in the mangle table that only allows ports 80/443 from eth0, dropping all k3s NodePorts. Tailscale (100.64.0.0/10) is exempted via RETURN.
- **Dual persistence**: the rule must exist in BOTH the live iptables state and `/etc/iptables/rules.v4`, because the provider's hypervisor-maintenance reboots wipe the live rules.

But having defenses is one thing; knowing **they're alive right now** is another. Eventually, the checks had to be automated too.

### Plan: moving to a cheaper server

The second reason is a future migration. The current setup — k3s, iptables, systemd timers, tailscale, helm — was built by hand. Reproducing it on a new server from memory is **guaranteed to produce omissions** — the incident above was exactly that pattern. With the server state in code, a migration becomes "git clone → inject secrets → run playbook".

> IaC is not a big-company tool. It's *more* essential for a solo operator — because memory weathers faster than code.

---

## 3. How I Use It — The Actual Setup

The repo is [Lajancia/server-ops](https://github.com/Lajancia/server-ops):

```
server-ops/
├── site.yml              # role application order
├── inventory/hosts.yml   # target servers
└── roles/
    ├── common/           # base packages (Docker intentionally NOT installed)
    ├── k3s/              # k3s server install + wait for node Ready
    ├── nodeport_guard/   # mangle firewall unit + rules.v4 dual persistence
    ├── tailscale/        # install + serve (ArgoCD :8443, Grafana :8444)
    ├── monitoring/       # kube-prometheus-stack (helm)
    └── weekly_audit/     # weekly security audit (systemd timer)
```

### Secrets never touch the repo

The kube-prometheus-stack values file contains the Grafana admin password. Such files live outside the repo in `/root/server-ops-secrets/` (chmod 700) and are referenced by path at run time. Same for the Discord webhook URL. Same rule as "never store PATs in files": secrets are injected manually.

### The weekly audit — the server checks itself, and only complains on failure

A systemd timer (`weekly-audit.timer`, Sundays 03:00 UTC, `Persistent=true` so missed runs after reboots are caught up) runs a script whose checks mirror the past incidents precisely:

1. **Docker container detection** — the healthy state of this server is *zero* containers. One running container is the incident pattern all over again.
2. **Guard rule dual check** — both the live mangle rule and the persisted `rules.v4` copy. If either disappears after a reboot, it gets caught.
3. **External NodePort scan** — curling NodePorts 30801 (Grafana) / 32085 (ArgoCD) *from the server itself* reports misleading results (self-connections traverse `lo`/hairpin). So the script asks an **external scanner (check-host.net)**: "can the public internet connect to this port?"
4. **Control plane** — k3s/nodeport-guard service active, monitoring pods Running.

A clean run appends a single `OK: all checks passed` to the log and stays quiet. **Only failures trigger one Discord alert.** Daily "everything is fine" pings become unread noise — designing against alert fatigue was a deliberate choice.

### Idempotency verification — the most important measurement

A playbook only deserves to be called "state as code" if re-running it changes nothing. I ran it twice in a row:

```
$ ansible-playbook site.yml --skip-tags manual-step   # 1st run
PLAY RECAP ... changed=0 failed=0
$ ansible-playbook site.yml --skip-tags manual-step   # 2nd run
PLAY RECAP ... changed=0 failed=0
```

Getting `changed=0` required fixing several traps I hit along the way:

- **Don't add `changed_when: true` out of habit.** I put it on an iptables-persistence task; it flagged "changed" on every run and even triggered a k3s restart. Ansible's built-in change detection is already correct.
- **Gate helm upgrades on state.** A bare `helm upgrade` is always "changed". Read the release version with `helm get metadata -o json` and upgrade only when the release is missing or stale.
- **Stamp first-run-only logic.** The audit's initial run is detected by the absence of its own log file.
- **`--check` mode limitation**: tasks that enable systemd units whose unit *files* the same play creates will fail in check mode (the file never gets written). Validate such plays with a real run.

This habit — "periodically re-run and confirm idempotency hasn't broken" — pairs naturally with incidents where settings fail to survive a reboot. If settings aren't persisted and validation doesn't repeat, the operator is being lied to in between.

---

## 4. What I Expect Next

- **A server migration becomes three commands**: git clone → inject secrets → `ansible-playbook site.yml --skip-tags manual-step`. Only the interactive tailnet auth (tagged `manual-step`) needs a human; everything else is decided by code.
- **AWS expansion**: add the host to the inventory and apply the same roles. If a multi-node k3s becomes necessary, the k3s role grows a worker-join step — nothing else changes.
- **Pairing with CI**: today the audit runs inside the server on a systemd timer. Next step is running the playbook *from outside* (e.g. GitHub Actions) on a schedule, adding an out-of-band drift check — this post's idempotency measurement is the safety net for that.
- **Expanding the audit**: currently ports/containers/pods; the same frame can accumulate auth-log review (verifying zero external login attempts) and more.

Most of introducing IaC on a running server turned out to be not coding, but **honestly transcribing what the server's real state is into code** — and then verifying, twice, that the resulting code changes nothing (`changed=0`). Once those two are done, the server exists as code, not as memory.

A follow-up post will cover actually running this playbook against a different machine (or an AWS instance) — the real migration scenario.
