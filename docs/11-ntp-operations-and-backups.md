# Phase 11 — NTP Verification, Operations Planning, and Off-Device Backups

**Result:** NTP synchronization was verified directly on the Raspberry Pi, Proxmox host, and Ubuntu VM 100; the collaborator subsequently reported that their VM also worked. No NTP firewall change was needed for the tested connections. I established terminal access to the Proxmox host, reviewed the centralized-management proposal, downloaded an encrypted post-update router backup, and created and downloaded the first recorded VM 100 backup. Recovery testing remains open.

> **Historical scope:** Work following Phase 10, through November 4, 2026. This is a record of observed results and reported progress, not a live inventory. Successful backup creation and transfer do not establish that restoration works.

Previous phase: [Network Hardening and Router Update](10-network-hardening-and-router-update.md). Project overview: [README](../README.md).

> **Publication convention:** Overlay addresses are examples: Pi `100.101.10.10`, MacBook `100.101.10.20`, Proxmox `100.101.10.30`, Ubuntu `100.101.10.40`. Private LAN addresses and backup filenames are retained. Personal account identifiers, real host-key fingerprints, passwords, private keys, and backup contents are omitted. Commands document the workflow with these substitutions.

## 1. Objective

The next stage focused on operational evidence: verify the requested NTP access before changing the firewall, agree on responsibilities for new management tools, and preserve recoverable copies of the existing lab configuration and Ubuntu guest.

## 2. Starting point after Phase 10

| Component | Starting state |
| --- | --- |
| MikroTik | RouterOS and RouterBOOT 7.23.8; selected unused management services disabled and checked after reboot |
| Router backup | Encrypted pre-update backup downloaded; post-update copy still pending |
| Pi | NTP previously working; America/Chicago configured; collaborator had full sudo |
| Proxmox | Remote HTTPS available; earlier tailnet policy did not permit host SSH |
| Ubuntu VM 100 | Key-only SSH verified; no completed VM backup recorded |
| Collaboration | Separate Proxmox account and resource pool; further operational planning through GitHub |

## 3. Work sequence

| Stage | Recorded work |
| --- | --- |
| Ticket review | Reviewed the outbound NTP request and centralized-management proposal |
| Host access | Compared Proxmox SSH host-key fingerprints and established terminal access |
| Time checks | Inspected the Pi, Proxmox, and VM 100; received the collaborator's VM confirmation |
| Documentation | Updated the two proposal/request files on the existing PR branch |
| Responsibilities | Selected Network/Security responsibilities; collaborator took GLPI work |
| Router backup | Created and downloaded a separately named encrypted post-update backup |
| VM backup | Created a Snapshot-mode, ZSTD-compressed VM 100 archive on local storage |
| Off-device copy | Downloaded the actual VM archive to the MacBook using SCP |

## 4. Ticket 001 — Verify NTP before changing the firewall

The request proposed allowing outbound UDP destination port 123 from lab systems to approved NTP servers. Earlier evidence already showed synchronization on the Pi. We therefore checked the other systems rather than assuming a missing firewall rule.

### Direct checks

On each inspected Linux system:

```bash
timedatectl status
```

On Proxmox, which was using chrony:

```bash
chronyc tracking
```

On the Pi, using systemd-timesyncd:

```bash
timedatectl timesync-status
sudo journalctl -u systemd-timesyncd --since "24 hours ago" --no-pager -n 80
```

| System | Evidence | Result |
| --- | --- | --- |
| Proxmox host | Direct command output, November 2 | Clock synchronized; NTP active; chrony leap status Normal; stratum 3; clock about 0.325 ms slow relative to NTP |
| Raspberry Pi / Kali | Direct command output, November 2 | Clock synchronized; NTP active; source in `0.debian.pool.ntp.org`; leap Normal; stratum 2; offset +1.278 ms; packet count 721 |
| Ubuntu VM 100 | Direct command output, November 2 | Clock synchronized; NTP active; display zone Etc/UTC |
| Collaborator VM | Collaborator report | Reported working; raw diagnostic output was not included in this record |

Proxmox and the Pi displayed America/Chicago. Ubuntu displayed UTC. Different time zones change how time is displayed, not whether the clock is synchronized.

After correcting the service name and command spelling, the Pi's journal query returned `-- No entries --`. That result alone did not show a stopped service or a blocked NTP connection. It also could not confirm or rule out an earlier transient failure described by the collaborator.

**Decision:** No new NTP firewall rule was applied during these checks. The observed clients could synchronize through the existing network configuration. This did not implement a restricted approved-server allowlist or validate every possible NTP destination. Router and switch time-client checks were outside this phase's evidence.

## 5. Proxmox SSH from the MacBook

Earlier policy allowed the Proxmox web interface on TCP 8006 but did not grant TCP 22 to the host. The intended change was to allow host SSH from the administrator's MacBook and update the corresponding policy tests, while keeping collaborator host SSH outside that grant.

The final saved policy JSON was not captured in this record. Successful SSH and the subsequent SCP transfer demonstrate that the administrator's connection path worked; they are not a complete ACL audit.

From the **MacBook**:

```bash
ssh -o ConnectTimeout=10 root@100.101.10.30
```

Before accepting the first connection, the displayed fingerprint was compared with this output from the **trusted Proxmox console**:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

The fingerprints matched. An initial connection closed after adding the host key; a later connection succeeded. No confirmed cause was established for the first closure.

This used ordinary OpenSSH over the Tailscale network. It did not enable the separate Tailscale SSH feature or grant the collaborator Proxmox root access. The Ubuntu guest's key-only SSH settings also do not automatically apply to the Proxmox host.

## 6. Ticket 002 — Centralized management proposal

| Proposed tool | Purpose | Recorded status |
| --- | --- | --- |
| Authentik | Central identities, SSO, MFA, and application access integration | Proposed; deployment not verified |
| OpenBao | Secrets storage and policy-controlled access | Proposed; deployment not verified |
| GLPI | Tickets, asset inventory, and operational organization | Collaborator handling implementation; working deployment not yet verified |
| Ansible | Repeatable server configuration and user/package automation | Proposed; automation not yet verified |

I selected responsibility for **Network and Security**. The collaborator confirmed taking the GLPI work. A broader Systems/Development split was discussed, but this record does not establish that every responsibility or integration was finalized.

The collaborator was reported to have the permissions needed to proceed. No new effective-permission dump, upload test, or boundary test was recorded here, so the earlier detailed permission checks remain the evidence for those controls.

Before deploying the full stack, machine placement, RAM/CPU/storage allocation, access paths, and backup ownership still need a concrete plan. This proposal did not itself create departments, accounts, SSO integrations, or automated provisioning.

## 7. GitHub workflow

The PR contained the NTP request and the management-tools proposal. We updated their documentation on the existing `bugfix/NTP-failing` branch using **Commit directly to this branch**.

The NTP notes distinguish direct checks from the collaborator's confirmation. The management proposal records agreement and the next planning steps.

A commit to the PR branch updates its proposed changes. It does not merge them into `main`. A completed merge and final ticket closure were not confirmed in the available record.

## 8. Encrypted post-update MikroTik backup

On November 3, a separately named encrypted backup was created:

```text
after-update-2026-11-03.backup
```

| Item | Recorded result |
| --- | --- |
| Encryption | AES-SHA256 with a separately retained password |
| Router file size | 37.9 KiB |
| File listing timestamp | November 3, 2026, 11:08:15 |
| Transfer | SFTP download reached 100% |
| Off-device destination | MacBook Downloads |
| Earlier copy | `before-update-2026-10-29.backup` retained separately |

At the **SFTP prompt**, the transfer used:

```text
get after-update-2026-11-03.backup /Users/<MAC_USER>/Downloads/after-update-2026-11-03.backup
```

Replace `<MAC_USER>` with the local macOS account name; the successful session used its full Downloads path.

This copy captures the post-update router state. Neither backup was restored during this phase, and a password/recovery test was not performed. Raw configuration backups are not repository artifacts.

## 9. First recorded VM 100 backup

In Proxmox, I selected **VM 100 → Backup → Backup now**.

| Setting | Selected value |
| --- | --- |
| Guest | VM 100 / `ubuntu-lab-01` |
| Storage | `local` |
| Mode | Snapshot |
| Compression | ZSTD |
| Notes | `{{guestname}}` |

The backup appeared as:

```text
vzdump-qemu-100-2026_11_04-10_21_38.vma.zst
```

The interface showed approximately **1.80 GB**. The archive was stored under `/var/lib/vz/dump/`. Snapshot here is the backup mode; the resulting `.vma.zst` file is the guest backup archive, not merely a named VM snapshot.

An initial downloaded file was the task log. We identified the actual archive by its `.vma.zst` extension and size, then copied it from the host.

From the **MacBook terminal**, using the illustrative host address:

```bash
scp root@100.101.10.30:/var/lib/vz/dump/vzdump-qemu-100-2026_11_04-10_21_38.vma.zst ~/Downloads/
```

The transfer reached **100%**, reported approximately **1718 MB**, and returned to the MacBook prompt. The earlier 84% line was an intermediate progress display for the same transfer. The displayed sizes are consistent with different units and rounding; no checksum comparison was performed.

The archive now has an off-device copy on the MacBook, in addition to the copy on Proxmox. It backs up VM 100; it is not a complete backup of the Proxmox host configuration, all guests, or the Pi.

## 10. Verification limits and remaining work

| Completed | Still pending |
| --- | --- |
| Direct NTP checks on three systems; collaborator confirmation for their VM | Historical incident evidence if NTP failures recur |
| Administrator SSH and SCP path to Proxmox worked | Capture the final tailnet policy and test intended denials |
| Encrypted post-update router backup downloaded | Router recovery/password test and sanitized text export |
| VM 100 backup created and copied off-device | Checksum comparison and isolated restore test |
| GLPI responsibility assigned | Resource plan, deployment verification, and access review |
| Ticket documentation updated on PR branch | Confirm merge and final ticket state |

The next recovery exercise should use an unused VM ID, preserve VM 100, and keep the restored guest's network adapter disconnected **before its first boot**. A restored guest can retain the original IP configuration and Tailscale identity. Recovery should first be inspected through the console.

Scheduled backups, retention, and additional independent copies are not yet documented as configured. Earlier open work—including switch restart persistence, remaining updates, and deliberate network/permission boundary tests—remains open unless separately verified.

## 11. Skills practiced

Evidence-based ticket review · NTP client diagnostics · Time-zone interpretation · SSH host-key verification · Proxmox terminal administration · PR-branch documentation · Responsibility planning · Encrypted router backups · Proxmox guest backup modes · SFTP and SCP transfers · Distinguishing logs from archives · Recovery planning

**End-of-phase state:** Time synchronization was working on the directly checked machines and reported working on the collaborator's VM. The router and VM 100 now have downloaded backup copies. The next evidence gap is recovery validation, alongside a concrete deployment plan for the proposed management tools.
