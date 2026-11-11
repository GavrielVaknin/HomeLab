# Phase 14 — GLPI Ticket Process (Proposal)

Result: draft proposal for how the administrator and the collaborator use GLPI alongside the `tickets/` folder and the phase documents. Nothing in it has been deployed or tested.

> Historical scope: November 11, 2026. Sources are listed in section 8 and referenced as `[S1]`, `[S2]`, and so on.

## 1. Purpose

The administrator and the collaborator want one place to send each other requests, track who owns them, and look back later at what happened and why. This document proposes how to use GLPI for that and how it relates to the `docs/` and `tickets/` folders in this repository.

## 2. How to read the labels

| Label | Meaning |
| --- | --- |
| **Documented** | A statement taken from a source in section 8. The source ID is given next to it. |
| **Proposal** | A decision for the two of us to approve or change. Not a GLPI fact. |
| **Unverified** | Something we need but could not confirm from a source. Check before relying on it. |

## 3. Tool and deployment facts

| # | Statement | Label | Source |
| --- | --- | --- | --- |
| 1 | GLPI publishes an official Docker image, `glpi/glpi`, on Docker Hub. | Documented | [S8] |
| 2 | The official example is a compose file with a `glpi` service and a `db` service using the `mysql` image, with database settings passed through a `.env` file (`GLPI_DB_HOST`, `GLPI_DB_PORT`, `GLPI_DB_NAME`, `GLPI_DB_USER`, `GLPI_DB_PASSWORD`). | Documented | [S9] |
| 3 | In that example, GLPI's persistent data lives in a named volume mounted at `/var/glpi`. | Documented | [S9] |
| 4 | The image installs or updates GLPI automatically when needed. The behavior can be turned off with `GLPI_SKIP_AUTOINSTALL` and `GLPI_SKIP_AUTOUPDATE`. | Documented | [S9] |
| 5 | The image includes a background worker that runs GLPI's scheduled tasks every minute by default, controlled by `GLPI_CRONTAB_ENABLED`. A host cron job is therefore not needed. | Documented | [S9] |
| 6 | The container runs as the non-root `www-data` user, so traditional cron is not available inside it. | Documented | [S9] |
| 7 | Images are rebuilt periodically for OS and PHP security patches without changing GLPI's own code. An image can be pinned by digest. | Documented | [S9] |
| 8 | In the example, the MySQL root password is random and appears in the `db` container's logs. | Documented | [S9] |
| 9 | Which GLPI version the `latest` tag currently points to. Current documentation exists for GLPI 12 [S1], but the Docker help page still shows older version examples [S8]. | Unverified | — |
| 10 | The `glpi/glpi` image runs on arm64 (the Raspberry Pi). | Unverified | — |
| 11 | MariaDB works as GLPI's database. The official example uses MySQL. | Unverified | [S9] |
| 12 | Single sign-on from authentik into GLPI. | Unverified | — |

**Proposals for deployment**

- Pin the image to a specific version tag (or digest) after the first successful start, instead of using `latest`.
- Use the database image from the official example unless MariaDB is verified.
- Before deploying on the Pi, check the architecture: `docker manifest inspect glpi/glpi:latest | grep -i architecture`. If `arm64` is missing, deploy on a Proxmox VM instead.
- After deployment, open the GLPI system information page, note the exact version, and read the documentation for that version.
- Defer single sign-on until local accounts work.

## 4. What GLPI provides (Documented)

### 4.1 Ticket type, category, and ITIL alignment

- Every ticket is either an **incident** or a **request**, stored in the Type field. [S1]
- A category belongs to only one type. [S3]
- Categories are managed under Setup > Dropdowns > Ticket Categories. They are shared by tickets, changes, and problems, and can be hierarchical. [S4]
- A category can trigger automatic processing when a ticket is created, such as changing ticket attributes, through business rules. [S4]
- Following ITIL is recommended but not mandatory, and the module can be adapted to the organization. [S6]

### 4.2 Status life cycle

GLPI's statuses are New, Approval, Processing (assigned), Processing (planned), Pending, Solved, and Closed. [S1]

| Status | Documented behavior | Source |
| --- | --- | --- |
| New | The status a ticket has when created. | [S1] |
| Processing (assigned) | Set when processing is assigned to a group, a technician, or a supplier. | [S1] |
| Processing (planned) | Set when a task is added to the ticket and planned. | [S1] |
| Pending | A technician can set it at any time. Pending reasons can be configured. In GLPI's older glossary, Pending means processing is temporarily suspended and does not count toward processing time. | [S1], [S14] |
| Solved | Set when a solution is found. | [S1] |
| Closed | Set when the requester or writer validates the proposed solution, or automatically. | [S1], [S13] |

Statuses cannot be added or renamed, but some can be hidden per profile through a life cycle matrix. [S1]

### 4.3 Urgency, impact, and priority

- The requester sets urgency and the technician sets impact. Priority is computed from them using a matrix. [S1], [S3]
- The matrix is shared by tickets, problems, and changes, and can be adjusted in Setup > General > Assistance, including disabling some levels. [S2]
- The **Major** priority is not part of the matrix. [S2]

### 4.4 Linking, history, and closing

- Tickets can be linked in four ways: Linked to, Duplicates, Child of, and Parent of. Linked to has administrative value only and groups tickets with a similar problem. [S3], [S5]
- Followups document the progress of a ticket and the exchanges around it, and a Historical tab shows the ticket's history. [S3]
- Solved tickets can be closed automatically after a set time, configured under Administration > Entity > Assistance. [S5]
- Tickets can also be created from incoming email through a collector. [S7]

### 4.5 Default accounts

- GLPI creates default accounts, and after first login it warns you to change their passwords and to remove the `install` directory. [S10]
- The older install documentation lists them as `glpi` (administrator), `tech`, `normal`, and `post-only`. [S11]

**Limit:** the Docker image may handle the install directory itself, so the second warning may not appear. Confirm on first login.

## 5. Proposed lab conventions (Proposals)

### 5.1 Types

Use **Incident** when something that worked is broken or behaving wrongly (for example the NTP failure). Use **Request** for planned work or a new capability (for example "add a service", "harden SSH").

### 5.2 Categories and teams

Ticket 002 already proposes example categories (Network Requests, System Administration, Security Incidents, Hardware Requests, Infrastructure Changes) and four departments (Systems, Network, Security, Development). [S15] Start from those instead of inventing a new list.

| Category | Typical use | Existing example |
| --- | --- | --- |
| Network Requests | Router, switch, firewall, DNS, NTP | Ticket 001 (NTP) |
| System Administration | Proxmox, VMs, Raspberry Pi, Linux updates, SSH | — |
| Security Incidents | Suspected compromise, exposed services, access problems | — |
| Hardware Requests | Equipment, cabling, storage | — |
| Infrastructure Changes | New services and tools, such as authentik, OpenBao, GLPI | Ticket 002 (management tools) |

Add "Backup and Recovery" or "Documentation and Process" only if tickets keep not fitting.

A category belongs to only one type [S3]. Either create each category once per type, or check the category's visibility options in the interface (unverified for this version).

The existing tickets name a "Network Team" and a "Systems Team" as the assigned team. [S15] Create those two as GLPI groups first, and add Security and Development when someone works in them.

### 5.3 How we use each status

| Status | Our rule |
| --- | --- |
| New | Logged, nobody owns it yet. |
| Processing (assigned) | One named owner. |
| Processing (planned) | Work is scheduled, such as a maintenance window. |
| Pending | Waiting on the other person or an outside party. The reason is written in a followup. |
| Solved | The fix is applied **and** the solution text records what changed, how it was verified, and what remains, matching the style of the phase documents. |
| Closed | The other person has confirmed, or auto-close has fired. |

### 5.4 Priority

Keep the default matrix. The requester sets urgency and the technician sets impact [S1]. Reserve **Major** [S2] for loss of access to the lab (for example, Proxmox or remote access unreachable).

### 5.5 Linking

Use **Parent of / Child of** to split a large piece of work into steps, **Duplicates** for repeated reports, and **Linked to** for related topics [S3], [S5].

### 5.6 Numbering and the repository

- The repository has two ticket files: 001 (NTP, closed on 2026-11-03) and 002 (management-tools proposal, status Proposed). [S15]
- Back-fill both into GLPI first, in that order, so GLPI IDs 1 and 2 match the file numbers. After creating the first one, check its ID (a fresh install is expected to start at 1, which is unverified). The next new ticket is then 003.
- The ticket files carry their own status in the header. Ticket 001's header still says Open even though its final update closes it. [S15] Once GLPI is live, GLPI holds the status, and the repository file records the sanitized outcome with the GLPI ticket number.
- Put the GLPI ticket number in the PR title and the ticket file name, and put the PR or document link in the GLPI solution text.

Mapping from the existing template to GLPI fields [S15], [S3]:

| Ticket file field | GLPI field |
| --- | --- |
| Ticket ID | Ticket ID |
| Type | Type (Incident or Request) plus Category |
| Priority | Urgency and impact, which compute the priority |
| Status | Status |
| Assigned Team | Assigned group |
| Requested By | Requester |

### 5.7 What goes where

| Where | What it holds |
| --- | --- |
| GLPI (private) | Requests, assignments, status, discussion, raw error output, anything with real identifiers. |
| Repository (public) | The sanitized result: what changed, why, what was verified, what remains. |

The existing README already omits passwords, MAC addresses, account emails, and real key fingerprints, and keeps real identifiers in a private working record. [S12] GLPI is the proposed home for that private record.

### 5.8 Accounts

- Create one named account per person.
- Change or disable the default accounts after first login. [S10]
- Review profiles before assigning them. A profile is a set of rights. [S13]

### 5.9 Retention and backup of the ticket history

GLPI's database is the only copy of our ticket history, so it needs a backup. The existing nightly Proxmox job covers VM 100 only. [S12] Decide whether GLPI runs on a Proxmox VM that the job can include, or on the Pi with a scheduled database dump copied elsewhere.

## 6. Open decisions

1. Where GLPI runs (Pi or Proxmox VM) and who owns its backup. The README already lists deployment location and backup ownership as open. [S12]
2. Category structure (section 5.2), including how to handle the one-type-per-category rule.
3. Whether to set auto-close, and after how many days. [S5]
4. Whether to configure email notifications, and through which mail account.
5. Whether to add authentik single sign-on after local accounts work.

## 7. Verification limits

- I read documentation pages and search results on the research date. Nothing was deployed or tested.
- GLPI documentation exists in several versions. Where I relied on an older page (S11, S14), I said so.
- The Docker image README was read through a repository that forks the official `glpi-project/docker-images` repository [S9]. Re-read the upstream copy before deploying.
- The ticket files and the authentik phase document were read from an archive of the collaborator's local clone made on 2026-11-11. The authentik phase exists only on a branch that was not merged at that time. The clone was not compared with the upstream repository's current state.
- The GLPI menu paths above come from the cited pages and may differ slightly by version.

## 8. Sources

All retrieved November 11, 2026.

| ID | Source | Used for |
| --- | --- | --- |
| S1 | https://help.glpi-project.org/documentation/12-en/assistance/tickets/ticketlifecycle | Types, statuses, status rules, urgency and impact |
| S2 | https://help.glpi-project.org/documentation/12-en/assistance/prioritymatrix | Priority matrix, Major priority |
| S3 | https://help.glpi-project.org/documentation/12-en/assistance/tickets/ticketmanagement | Type and category, linked tickets, followups, history |
| S4 | https://help.glpi-project.org/documentation/12-en/setup/dropdowns/categories | ITIL categories, hierarchy, rules |
| S5 | https://help.glpi-project.org/faq/glpi/ticket | Auto-close setting, link type differences |
| S6 | https://help.glpi-project.org/documentation/12-en/assistance/overview | ITIL alignment is optional |
| S7 | https://help.glpi-project.org/documentation/modules/assistance/tickets/ticketadvanced | Mail collector, duplicates, business rules |
| S8 | https://help.glpi-project.org/tutorials/procedures/running_glpi_on_docker | Official Docker image |
| S9 | https://github.com/links-ads/glpi-docker (fork of https://github.com/glpi-project/docker-images) | Compose example, environment variables, cron worker, rebuild policy |
| S10 | https://help.glpi-project.org/documentation/12-en/readme-1-1/interfaces | First-login warnings |
| S11 | https://glpi-install-ru.readthedocs.io/ru/latest/install/wizard.html | Default accounts (older install documentation) |
| S12 | https://github.com/sivanhorsun/HomeLab | Existing ticket process, sanitization approach, backup scope |
| S13 | https://help.glpi-project.org/documentation/12-en/glossary/gloassary | Closed, profile definitions |
| S14 | https://glpi-user-documentation.readthedocs.io/fr/latest/glossary.html | Pending definition (older documentation) |
| S15 | `tickets/001-network-allow-ntp.md` and `tickets/002-propose-homelab-it-management-tools.md` in the collaborator's clone of this repository | Existing ticket format, categories, teams, ticket status |
