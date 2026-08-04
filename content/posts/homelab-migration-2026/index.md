---
title: "Homelab!! Part 2"
date: 2026-08-02
summary: "the migration arc (this is taking forever)"
tags: ["Homelab", "Proxmox", "Kubernetes", "Networking", "GitOps"]
author: "Sean"
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: true
description: "adding another proxmox host and trying not to destroy the first one"
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
UseHugoToc: true
---

My [previous homelab article](/posts/homelab-showcase/) ended with two k8s clusters, several LXCs, Tailscale, Cloudflare, Proxmox, and one tiny Beelink doing far too much work.

Obviously, the solution was to buy / acquire another computer and make the architecture even more complicated.

I now have a Dell Proxmox host which is intended to take over more of the workloads from the original Beelink.

The important word there is **intended**.

The Beelink is still the production source for anything which has not completed an actual migration and cutover. A VM existing on the Dell does not make it production, no matter how emotionally attached I am to the new server.

# Current topology

At a high level, the network now looks like this:

```text
Internet
   |
OPNsense on Beelink Proxmox
   |
managed switch ---- access point
   |
Dell Proxmox
```

I also have separate trust zones for ordinary clients, private infrastructure, IoT, public workloads, and guests.

The funny part is that the original Beelink still hosts OPNsense, which the Dell needs for routing. This means I cannot just finish migrating a few apps, unplug the “old server,” and celebrate. I would also be unplugging the router which makes the new server reachable.

This is why I keep making network diagrams. They turn a vague dependency into a very obvious line which says “do not turn this off, idiot.”

# Moving this blog first

I used this Hugo blog as one of the earlier workloads on the Dell.

A static blog running inside an unprivileged LXC is relatively forgiving compared to something with a database, so it was a good way to test the new public-workload network.

The networking still managed to become a pain.

At first, `ping` failed inside the container. It looked like a VLAN or bridge problem, but it turns out unprivileged containers can also fail to open raw ICMP sockets because the binary does not have the required capability.

Fixing that did not automatically fix the network though. There were still separate questions:

1. Did the LXC receive the expected DHCP lease?
2. Could it reach the gateway?
3. Did DNS work?
4. Did the firewall block private networks?
5. Could it still reach the public internet?
6. Could someone outside reach the blog through the intended route?

This taught me to stop treating “ping does not work” as one problem.

A local Linux capability error, a bad VLAN tag, a missing DHCP reservation, DNS, and an OPNsense rule can all look like “the network is broken” from far away. Fixing one layer does not prove any of the others.

# VLANs / Firewall

The public-workload network uses a default-deny policy.

A new host needs explicit access to DNS, the gateway, and the public internet, while still being blocked from the private networks. Rule order also matters because OPNsense uses first-match behavior.

I lost a little time staring at a perfectly good allow rule which sat below a broader block rule and therefore did absolutely nothing. Very cool.

I have also become much more careful about assuming that a device's management IP describes all of its traffic.

The switch can have its management address in one VLAN while carrying trunks for several others. The access point can be managed from the IoT side while serving multiple SSIDs. Management plane and data plane are related, but they are definitely not the same thing.

# GitOps does not migrate data

Most of my public applications are managed through Git and Argo CD.

I love GitOps because I can review the desired state and avoid SSHing into a random server to change production. Unfortunately, Argo saying **Synced** does not mean the entire application migration is complete.

For every workload, there are several separate facts:

1. source code commit
2. published container image / digest
3. GitOps manifest
4. reconciled cluster resources
5. persistent data and secrets
6. actual user journey

A green GitHub check proves the first few pieces at best. It does not prove the PVC contains the right data or that the login flow works through Cloudflare / Tailscale.

I am also trying to use immutable digests or versioned tags at the deployment boundary. `latest` is very convenient until I need to answer the question “which image is actually running?”

# Backups

This migration has made me much more paranoid about backups, which is probably healthy.

A backup appearing inside the datastore only proves that some bytes exist. It does not prove that the snapshot is complete, readable, decryptable, or restorable.

Before cutting over a stateful service, I want:

- the source data preserved
- a fresh backup
- a successful verification result
- the secrets required to restore it
- enough destination storage
- a rollback path
- a read-only consistency check after moving it

Encrypted backups especially need real verification. Seeing an encrypted snapshot and feeling safe is not the same as knowing that I can restore it.

I used to think of a backup as a file. I am slowly learning that a backup is really a tested recovery process, which is much less convenient but unfortunately true.

# Stateful applications

Moving this blog is easy compared to moving Actual Budget, Immich, or anything else with meaningful state.

For every stateful service, I need to answer:

1. Which instance is the source of truth right now?
2. Can writes happen during the migration?
3. Where are the database, files, and encryption keys?
4. How do I detect a partial copy?
5. How long do I keep the old source?
6. What exact test allows me to switch traffic?

The safest flow is boring: keep the old source intact, copy the data, validate the target, and move production traffic last.

DNS and firewall changes are not debugging toys when they decide which database receives the next write.

# When is it actually migrated?

Not when the VM boots.

Not when Argo says Synced.

Not when the pod says Ready.

Not even when the homepage returns HTTP 200.

I consider a workload migrated when the real data is intact, secrets are correct, the actual user journey works, monitoring exists, backups are verified, rollback is possible, and production traffic intentionally points to the new instance.

The old source only gets retired after all of that stays boring for a while.

# Conclusion

The Dell is real and quite a few workloads are already moving over, but I am trying very hard not to declare the whole migration complete just because the new machine is exciting.

Infrastructure migrations are mostly a long series of tiny checks which prevent one dramatic disaster.

This is taking forever, but at least I still have a working homelab.

More updates when I inevitably add another computer and make the diagram worse.
