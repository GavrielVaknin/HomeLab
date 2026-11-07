# Phase 12 — Recovery Testing, Access Validation, and SSH Hardening

**Result:** The VM 100 backup was verified by checksum and restored to an isolated test VM. A daily backup job completed manual and scheduled runs. Selected access paths were checked, PR #1 was merged, switch HTTP was disabled, and switch NTP was repaired and checked after reboot. Pi SSH password authentication was disabled and a fresh administrator key login succeeded.

> **Historical scope:** Work after Phase 11 through November 7, 2026. Direct output, screenshots, and user/collaborator reports are distinguished below. Earlier phase snapshots remain historical; this chapter updates their open items.

Previous phase: [NTP Verification, Operations Planning, and Off-Device Backups](11-ntp-operations-and-backups.md). Project overview: [README](../README.md).

> **Publication convention:** Overlay addresses in examples are substituted: Pi `100.101.10.10`, MacBook `100.101.10.20`, Proxmox `100.101.10.30`, Ubuntu `100.101.10.40`. The collaborator account is normalized to `lab-user` and tailnet identity to `<COLLABORATOR_ID>`. Private LAN addresses and backup filenames are retained. Personal identifiers, real key fingerprints, passwords, private keys, and backup contents are omitted. Commands record the workflow; they are not instructions to repeat completed changes.

## 1. Starting point and work sequence

At the end of Phase 11, router and VM 100 backup copies had been downloaded, but VM recovery and scheduled backups were not verified. NTP worked on the tested Linux systems; the switch had not been included. Documentation remained on the open PR branch.

| Stage | Recorded progress |
| --- | --- |
| Recovery | Compared archive checksums; restored and booted an isolated test VM |
| Automation | Created daily 02:00 backups with three retained copies; observed successful runs |
| Access | Reviewed saved Tailscale JSON and checked selected allowed/denied paths |
| Collaboration | Received Pi, Proxmox, and working-Ubuntu confirmations; resolved browser symptom with Firefox |
| GitHub | Merged PR #1 into main |
| Switch | Disabled HTTP, verified HTTPS, saved configuration, diagnosed post-reboot NTP failure |
| Pi SSH | Backed up configuration, disabled passwords, reloaded, and verified fresh key access |

## 2. Checksum comparison and isolated restore

The archive from Phase 11 remained on Proxmox under `/var/lib/vz/dump/` and in the MacBook Downloads folder:

```text
vzdump-qemu-100-2026_11_04-10_21_38.vma.zst
```

SHA256 matched on both devices:

```text
01d901c46acba0a25d39f9671195cdd700ab1f6d6416a21d454a670f8d68deb2
```

| Item | Action or result |
| --- | --- |
| Original | VM 100 / ubuntu-lab-01 preserved |
| Restored guest | VM 900 / ubuntu-restore-test |
| Storage | local-lvm |
| Network | Unique MAC; NIC disconnected with link_down=1 before first boot |
| Validation | Ubuntu booted; console login succeeded; SSH settings inspected |
| Cleanup | Test VM shut down and destroyed; VM 100 remained present and running |

The matching hashes verify identical archive content. The restore verifies that the selected archive could be restored and booted. Keeping the restored guest off the network avoided conflicts involving copied addressing or Tailscale identity. This was not a full application/network test or a recovery of the entire Proxmox host, router, or Pi.

## 3. Daily backup job

An enabled job was created in **Datacenter → Backup** for **VM 100 only**:

| Setting | Value |
| --- | --- |
| Schedule | Daily 02:00 according to the Proxmox host clock |
| Storage | local |
| Mode / compression | Snapshot / ZSTD |
| Retention | keep-last=3 |

The task history showed:

| Run | Start | End | Status |
| --- | --- | --- | --- |
| Manual | November 6, 18:21:51 | November 6, 18:22:21 | OK |
| Scheduled | November 7, 02:00:03 | November 7, 02:00:37 | OK |

The scheduled run demonstrates automatic execution and successful completion. Pruning after more than three archives accumulated was not separately observed. The restore test used the November 4 archive, not every subsequent backup.

The job writes to the same host's local storage. The MacBook archive is a manually downloaded off-device copy; no automated off-host transfer of every nightly backup was configured.

## 4. Saved Tailscale policy

The saved JSON contains four host aliases, restricted grants, an empty ssh array, and tests.

| Source | Destination | Allowed TCP ports |
| --- | --- | --- |
| sivan-mac | lab-pi | 22 |
| sivan-mac | lab-proxmox | 22, 8006 |
| sivan-mac | lab-ubuntu | 22 |
| `<COLLABORATOR_ID>` | lab-pi | 22 |
| `<COLLABORATOR_ID>` | lab-proxmox | 8006 |

This uses ordinary OpenSSH over Tailscale. The separate Tailscale SSH feature was not enabled.

### Observed tests

| Source | Path tested | Result |
| --- | --- | --- |
| MacBook | Pi 22; Proxmox 22 and 8006; Ubuntu 22 | All four TCP checks succeeded |
| MacBook | Pi 80 over Tailscale | Timed out |
| Pi | Proxmox 22 and 8006 over Tailscale | Both timed out |
| Collaborator | Pi SSH and Proxmox web | Login succeeded |
| Collaborator | Proxmox 22 and Ubuntu 22 | Both timed out |

Example command shapes with substituted addresses:

```bash
# macOS
nc -vz -G 3 100.101.10.10 22
# Linux
nc -vz -w 3 100.101.10.30 22
```

The results match the intended paths, but timeout alone does not identify the blocking component and is not a connection-refused response. Policy tests are assertions, not independent deny rules. The complete live denial matrix and a complete policy-test pass output were not captured; MacBook ports 22/445/5900 were not tested from every source. A saved copy is not continuous verification of the deployed policy. Overlay restrictions do not isolate devices on the shared lab LAN.

## 5. Collaborator confirmations and browser troubleshooting

The collaborator confirmed Pi SSH, Proxmox web access, and working Ubuntu. VM 100 was reportedly not visible in their account. This is evidence of successful use and reported UI scope, not a comprehensive API-level authorization audit or current guest inventory.

The SSH journal showed **Accepted publickey** for both administrator and collaborator. Collaborator key login worked again after the Pi's November 7 reboot. The login-keyring warning appeared alongside successful session opening and did not prevent access.

The collaborator's original browser became unresponsive at the **No valid subscription** dialog, including the OK/close controls. Firefox then worked without the reported freezing. No server upgrade was performed for the symptom, and an outdated web server was not established as its cause.

| Topic discussed | Status |
| --- | --- |
| Proxmox LXC permissions | Deferred; no new container-creation grant recorded |
| Docker versus KVM | Clarified that Docker in a Linux VM does not require nested KVM for containers |
| Always-on RHEL server | Proposed for a web dashboard or OS-image service; deployment not verified |
| Keycloak | Docker guide and localhost:8080 admin-console screenshot shared; lab integration not verified |
| GLPI and other management tools | Planning/work continues; no complete management-stack deployment verified here |

## 6. GitHub PR merged

The collaborator reported that ticket files existed locally and in their fork but were missing upstream. The PR was still Open; prior comments and commits had not merged it.

| Item | Verified state |
| --- | --- |
| PR | #1 — docs: add NTP request and IT management proposal |
| Source | bugfix/NTP-failing in collaborator fork |
| Target | main in HomeLab |
| Changes | Four commits; two changed files |
| Before merge | No conflicts with base branch |
| After confirmation | Merged; pull request successfully merged and closed |
| Merge commit | 644b578 |

**Merge pull request → Confirm merge** completed the operation. The change set included the NTP request, management proposal, validation results, Network/Security ownership, and the Ticket 001 document update. The collaborator can synchronize their fork and pull the merged main branch.

PR closure does not establish closure of a separate GitHub Issue or external ticket. Merging documents does not deploy software or change permissions. This new chapter and README update are prepared files; their publication is separate from the completed PR #1 merge.

## 7. Switch management hardening

The MacBook joined lab Wi-Fi to reach **https://192.168.88.252**. HTTPS login was verified, including in a private browser window. The certificate warning remained; no replacement certificate was installed.

HTTP was disabled under **Security → Access Security → HTTP Config**, applied, and saved.

| Check | Result |
| --- | --- |
| Pi → switch LAN TCP 80 | Connection refused |
| Pi → switch LAN TCP 443 | Connection succeeded |
| HTTP / Telnet / switch SSH after reboot | Disabled in System Summary |
| Access Control | Disabled; no new management-source allowlist |
| SNMP | Enabled; unchanged in this task |

The VLAN/routing design was retained. Firmware shown was **3.20.0 Build 20230818 Rel.72032**, hardware SG3210 3.20; no firmware update was performed. Configuration exports were saved privately on the MacBook.

Time zone changed from GMT+08 to **GMT-06 Central Time**, with **USA predefined DST** enabled. Those settings persisted after reboot, but the restart exposed an NTP problem.

## 8. Switch NTP failure and repair

After the first reboot the switch showed January 1, 2006, current time source **Manual**, and **SNTP Disabled** in System Summary. The configured NTP addresses and time zone remained. Apply produced:

```text
Failed to get time from NTP server.
```

| Diagnostic | Result |
| --- | --- |
| Management interface | VLAN1, DHCP, 192.168.88.252/24, Up |
| Default route | 0.0.0.0/0 via 192.168.88.1 present |
| Ping 192.168.88.1 | Four replies, 0% loss |
| Ping old server 139.78.100.163 | Four timeouts |
| Ping 1.1.1.1 | Four replies, 0% loss, 10 ms |
| Swapping old endpoints | Still failed with 199.165.76.11 as primary |

Gateway and public-IP connectivity worked. ICMP ping does not validate UDP/123, so the old endpoint's ping failure did not by itself establish an NTP outage or firewall block.

Both old endpoints were replaced:

| Setting | Final value |
| --- | --- |
| Primary | 216.239.35.0 |
| Secondary | 216.239.35.4 |
| Time zone | GMT-06 Central Time |
| DST | Enable / Predefined Mode / USA |
| Update rate | 12 hours |

Apply succeeded. The administrator confirmed accurate date/time, saved configuration, rebooted again, and confirmed that the time was still correct without another Apply. A fresh export was saved on the MacBook; the exact final filename was not shown.

Changing endpoints resolved the observed failure without a firewall change. The old endpoint failure was not isolated to a server, route policy, or compatibility problem. This switch incident does not invalidate Phase 11's successful Linux checks. Switch configuration recovery was not tested.

## 9. Pi SSH configuration backup and hardening

The administrator key worked, but effective settings still showed **PasswordAuthentication yes**. The SSH configuration was backed up locally:

```bash
sudo cp -a /etc/ssh /root/ssh-config-backup-2026-11-06
sudo ls -l /root/ssh-config-backup-2026-11-06/sshd_config
```

This is not an off-device Pi backup. The directory may contain private server keys and is not published.

The configuration search found an active include for the drop-in directory and existing keyboard-interactive disablement:

```text
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
```

The following new file was written:

```text
/etc/ssh/sshd_config.d/00-lab-key-only.conf
```

```bash
printf '%s\n' 'PasswordAuthentication no' 'KbdInteractiveAuthentication no' 'PubkeyAuthentication yes' | sudo tee /etc/ssh/sshd_config.d/00-lab-key-only.conf
sudo /usr/sbin/sshd -t
sudo systemctl reload ssh
```

The syntax check and reload returned without an error. The existing SSH session remained open during validation.

## 10. Fresh-session verification

A second **local MacBook terminal** was used to connect with passwords and keyboard-interactive authentication disabled in the client:

```bash
ssh -i ~/.ssh/id_ed25519_lab -o IdentitiesOnly=yes -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no kali@100.101.10.10
```

After the local key passphrase was entered, the new Pi session opened successfully. A key passphrase unlocks the private key on the MacBook; it is not the Linux account password used for server password authentication.

The final effective-settings command on the Pi was:

```bash
sudo /usr/sbin/sshd -T | grep -Ei '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

| Setting | Before | After |
| --- | --- | --- |
| PermitRootLogin | no | no |
| PubkeyAuthentication | yes | yes |
| PasswordAuthentication | yes | no |
| KbdInteractiveAuthentication | no | no |

The effective settings and fresh administrator key login were verified after reload. A separate password-only rejection test, a Pi reboot after the final change, and a collaborator reconnect after that change were not recorded. The collaborator key was not edited.

### CLI errors corrected

- Uppercase `-I` invokes a PKCS#11 provider; lowercase `-i` selects a key file.
- Running the client command inside the Pi looked for the key under `/home/kali`. That attempt was cancelled and rerun locally on the MacBook; the private key was not copied to the Pi.
- The `printf` format needed `\n`, a space before `yes`, and a `.conf` filename.
- Case-insensitive `grep -i` matched the capitalized effective output. An unintended literal space after the regex `*` was removed.
- The destination address was corrected before the successful login.

## 11. End-of-phase state

| Completed | Evidence boundary |
| --- | --- |
| Identical archive hashes and isolated restore/boot | Not every application, backup, or whole-host recovery tested |
| Daily backup and retention; scheduled run OK | Automated off-host copy and pruning behavior not separately verified |
| Saved policy and selected access tests | Not the complete denial matrix or API authorization surface |
| Working collaborator Ubuntu/Proxmox access | No complete new guest inventory |
| PR #1 merged and closed | New chapter/README still require repository publication |
| Switch HTTP off, settings persist | No config restore or certificate replacement |
| Switch time fixed and correct after reboot | Original endpoint cause not isolated |
| Pi SSH passwords off, administrator key verified | Collaborator reconnect and password-only rejection after final change unrecorded |

Router/switch configuration recovery, additional isolation exercises, and management-stack deployment remain separate work. Container permissions and the RHEL/image-server proposal were deferred. They are not silently counted as completed or added as new requirements to the checks already finished.

## 12. Skills practiced

SHA256 comparison · Isolated VM recovery · Backup scheduling and retention · Task-history review · Tailscale grants and assertions · Positive/negative TCP testing · Timeout interpretation · Browser troubleshooting · Fork/PR workflow · HTTPS-only switch management · Apply versus Save · Post-reboot NTP diagnosis · Routing-table interpretation · SSH includes · Syntax validation and reload · Fresh key sessions · Local versus remote terminal context · Technical documentation

**End-of-phase state:** Core recovery and hardening changes are completed with the evidence above. Chapter 12 is a documentation sequence number, not a claim that every possible lab test or future service is complete.
