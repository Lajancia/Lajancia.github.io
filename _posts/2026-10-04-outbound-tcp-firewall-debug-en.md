---
title: "The Day Outbound TCP Died — Debugging an All-Day Outage Caused by a Single Firewall Rule"
date: 2026-10-04 21:00:00 +0900
categories: [DevOps]
tags: [K3s, ArgoCD, GitOps, iptables, conntrack, tcpdump, Troubleshooting, Homelab]
author: L.J
lang: en
---

### **Introduction**

This post is a post-mortem of a debugging journey that started with a trivial symptom — "GitHub is unreachable" — and ended, after an entire day, at **a single iptables table-choice mistake**. The conclusion first:

> **A firewall rule in the raw table's PREROUTING chain was dropping the reply packets of every outbound TCP connection the server made. And because the raw table runs *before* conntrack, the ESTABLISHED exception rule never matched — no matter how we wrote it.**

---

### **The Symptoms**

On a homelab server (Hostinger VPS, Ubuntu 24.04, k3s + ArgoCD), the following kept happening:

- `github.com` — timeout after 134 seconds
- **Every** outbound IPv4 TCP connection failed (8.8.8.8:53 TCP, 1.1.1.1:80/443, github.com:443 …)
- ICMP worked fine (`ping 8.8.8.8` succeeded, RTT 1ms)
- **IPv6 worked fine** (Cloudflare edge was reachable)
- The Discord desktop app lost its gateway connection (TCP 443)
- It all started right after installing `nodeport-guard.service`, a k3s NodePort hardening unit

The working theory of the day was "external egress is blocked." Since the server seemingly couldn't reach GitHub over IPv4, we pivoted to designing a workaround.

---

### **The Workaround First: IPv6 + Cloudflare Proxy**

GitHub doesn't support IPv6 at all. But the server *could* reach the Cloudflare edge over IPv6. So we built a temporary bypass:

```
Server (ArgoCD) --IPv6--> Cloudflare edge (gh.example.com proxy)
                            ↓ CF internal network
                         github.com (Cloudflare fetches it for us)
```

DNS showed `gh.example.com` resolving to Cloudflare IPs (2606:4700::/32), and over IPv6 it answered in 0.3s. Pointing ArgoCD's repoURL at the proxy made syncs pass immediately. At the time, it read as "GitHub is unreachable, so we bypass." **What we didn't know then: the reason GitHub was unreachable was a firewall rule we ourselves had installed a few hours earlier.**

---

### **The Breakthrough: The Packets Were Arriving**

The turning point was `tcpdump`. While attempting a connection to 8.8.8.8:443, we captured eth0:

```
SYN  →  147.93.98.23.44886 > 1.1.1.1.443: Flags [S]
SYN-ACK ←  1.1.1.1.443 > 147.93.98.23.44886: Flags [S.]
SYN  →  (retransmission, over and over)
SYN-ACK ←  (arriving every time)
```

**The SYN-ACKs were arriving, every single time.** Yet the kernel kept retransmitting SYN as if it never saw them. Packets reached the NIC but vanished somewhere in the network stack. "Somewhere" was netfilter — and the rule counters gave the answer:

```
$ iptables -t raw -L PREROUTING -nv --line-numbers
1  RETURN  ctstate RELATED,ESTABLISHED   pkts: 0     ← never matched, not once
5  DROP   tcp dports !80,443            pkts: 7,793 ↑ ← climbing
```

The offending rule:

```
iptables -t raw -A PREROUTING -i eth0 -p tcp -m multiport ! --dports 80,443 -j DROP
```

The intent was sound: block direct inbound access to k3s NodePorts (30000+), leaving only 80/443 exposed. The flaw: **PREROUTING is also traversed by the reply packets of outbound connections.** When the server SYN'd to `github.com:443`, the reply entered eth0 with a dport not of 443 but of the **local ephemeral port** (e.g. 44886). `! --dports 80,443` matched. DROP. Handshake dead. Timeout.

This also explains why ICMP and IPv6 survived:

- ICMP wasn't TCP, so the drop rule ignored it (and that 1ms "RTT" was actually an upstream ICMP interceptor answering, not a real round trip)
- `-t raw` (iptables) is **IPv4-only**, so IPv6 traffic never touched the rule

---

### **The Deeper Trap: ctstate Matching Doesn't Work in the Raw Table**

The first fix attempt was the obvious one — add an established exception before the DROP:

```
iptables -t raw -I PREROUTING -i eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
iptables -t raw -A PREROUTING -i eth0 -p tcp -m multiport ! --dports 80,443 -j DROP
```

Still broken. The RETURN rule's match counter stayed at **zero, forever.**

The reason is netfilter's hook ordering. Packets traverse tables by priority:

```
raw (-300) → conntrack (-200) → mangle (-150) → nat → filter (0)
```

The raw table runs **before** conntrack. When the raw table evaluated the arriving SYN-ACK, connection tracking had not yet classified the packet's state — so `--ctstate ESTABLISHED` failed, the packet fell past the exception rule, and landed in DROP.

> **Lesson 1: Don't do conntrack state matching in the raw table. Use mangle or filter.**

---

### **The Final Fix**

We moved the entire guard to the mangle table. Mangle's PREROUTING hook sits at priority -150 — *after* conntrack (-200) — so ctstate matching works as expected:

```
# /etc/systemd/system/nodeport-guard.service
ExecStart=... iptables -t mangle -A PREROUTING -i eth0 -s 100.64.0.0/10 -j RETURN
ExecStart=... iptables -t mangle -A PREROUTING -i eth0 -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
ExecStart=... iptables -t mangle -A PREROUTING -i eth0 -p tcp -m multiport ! --dports 80,443 -j DROP
```

Verification:

```
github.com: 200 in 0.073s          ← direct access restored
Discord gateway: connected again    ← the "unrelated" symptom was the same root cause
ArgoCD case-study / next-r3f: Synced
```

The mangle RETURN rule immediately racked up hundreds of matches; the DROP rule kept dropping only NEW connections (5 packets).

> **Lesson 2: Passing ESTABLISHED replies doesn't weaken inbound protection.**
> What we're guarding is NEW connections. Established reply traffic adds no attack surface. SSH 22, k3s API 6443, kubelet 10250 — all stayed blocked to new connections, exactly as before.

---

### **Rolling Back the Workaround**

With direct github.com access restored, we reverted the Cloudflare proxy bypass to the original source of truth. Three places had to agree on `github.com`: the ArgoCD Application repoURLs, the repo credential secrets, and the GitOps repo's yaml. All Synced + Healthy after.

One more hiccup here: auto-sync re-applied the **pre-revert** yaml and trapped the Application on the old repoURL while credentials no longer matched it. It settled only once HEAD, the Application, and the secret all agreed again. In a self-referential GitOps structure, "yaml ↔ cluster state" consistency *is* the stability.

---

### **Takeaways**

1. **Run tcpdump before concluding "the network is blocking us."** If packets are arriving at the NIC while connections fail, the problem is local netfilter. Visible SYN-ACKs + failed handshakes = internal problem, 100%.
2. **iptables counters carry the answer.** A match counter frozen at 0 next to a climbing DROP counter identified the culprit rule with no further guessing.
3. **Choosing a table is choosing a priority.** ctstate matching in raw (-300) fails by design. State-based rules belong in mangle (-150) or filter (0).
4. **Verify the cause is truly external before building a bypass.** Our Cloudflare proxy wasn't a bad workaround — but its *reason to exist* was a bug we installed ourselves three hours earlier.
5. **One firewall rule can produce many symptoms.** GitHub timeouts, Discord disconnects, failing CI, ArgoCD sync errors — all the same rule. Group the symptoms first; the cause is usually singular.
