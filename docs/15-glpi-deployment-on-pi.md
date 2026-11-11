# Phase 15 — GLPI Deployment on the Raspberry Pi

Result: GLPI and MySQL are running in Docker Compose on the Raspberry Pi 5, and the web server answers locally on the Pi. First login, account setup, version pinning, and remote access are still pending.

> Historical scope: November 11, 2026. This phase follows [Phase 14 — GLPI Ticket Process](14-glpi-ticket-process.md) and Ticket 002, and builds on the Pi's container setup in Phase 13.

Passwords, session cookie values, and the database root password are omitted. As in earlier phases, the Tailscale address shown is a substituted example.

## 1. Decision and starting point

The collaborator and the administrator approved the GLPI proposal in Ticket 002 and Phase 14. The Pi (Raspberry Pi 5, Kali Rolling, ARM64) already runs authentik (Phase 13). Phase 13 records authentik's published ports as 9000 and 9443; according to the collaborator, they were later changed to 80 and 443 through the `.env` file, which was not re-verified in this phase. Because authentik uses 80 and 443, GLPI needed a different host port. The Pi had roughly 8 GB of RAM, with about 6.5 GB available while authentik was running, so memory was not a constraint for adding GLPI and MySQL.

## 2. Architecture check

Before deploying, the image's published platforms were checked:

```bash
docker manifest inspect glpi/glpi:latest | grep -i architecture
```

The output listed `amd64` and `arm64`, plus two entries reported as `unknown`. The `unknown` entries were interpreted as image build metadata and not as additional platforms; this was not confirmed from a source. `arm64` being present was the condition for deploying on the Pi.

## 3. Files created

Working folder: `~/Tools/glpi`, separate from the authentik folder.

**`.env`** holds the database settings: `GLPI_DB_HOST=db`, `GLPI_DB_PORT=3306`, `GLPI_DB_NAME=glpi`, `GLPI_DB_USER=glpi`, and a generated random `GLPI_DB_PASSWORD`. File permissions were set to `600`. The file is not committed to the repository.

**`compose.yml`** follows the official example from the image's README (see Phase 14, sources S8 and S9), with two changes: the host port is `8080` instead of `80`, and each service has Docker log-size limits.

```yaml
name: glpi

services:
  glpi:
    image: "glpi/glpi:latest"
    restart: "unless-stopped"
    volumes:
      - glpi_data:/var/glpi
    env_file: .env
    depends_on:
      - db
    ports:
      - "8080:80"
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: "mysql"
    restart: "unless-stopped"
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_RANDOM_ROOT_PASSWORD: "yes"
      MYSQL_DATABASE: ${GLPI_DB_NAME}
      MYSQL_USER: ${GLPI_DB_USER}
      MYSQL_PASSWORD: ${GLPI_DB_PASSWORD}
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  glpi_data:
  db_data:
```

## 4. Start-up

```bash
docker compose config -q
docker compose up -d
docker compose logs -f glpi
```

| Observation | Result |
| --- | --- |
| Compose validation | Completed without errors |
| `docker compose ps` shortly after start | Both containers `Up`; `glpi` published `8080` to container port 80; `db` listed only its internal ports 3306 and 33060, with nothing published to the host |
| First GLPI log lines | The entrypoint waited for the database, counting down from 120 attempts |
| MySQL log | First-start initialization completed, the `glpi` database and user were created, and the server reported ready for connections on port 3306 |
| Later GLPI log lines | "The database is now ready and reachable"; GLPI reported it was not installed and began its automatic installation |
| Install completion | The log line announcing the end of the installation was not captured in this record |

The wait for the database was caused by MySQL's first-start initialization and resolved without any change.

The MySQL logs also showed warnings about a self-signed CA certificate, TLS not requiring post-quantum key exchange, and the pid-file location. These were treated as default-image behavior and were not investigated further.

## 5. Local check from the Pi

From the Pi, a HEAD request to GLPI's published port was made using the Pi's Tailscale address:

```bash
curl -kI http://100.101.10.10:8080
```

| Result | Detail |
| --- | --- |
| Status | `HTTP/1.1 200 OK` |
| Server | Apache |
| Headers | A `glpi_`-prefixed session cookie was set with `HttpOnly` and `SameSite=Strict`, and caching was disabled |

This shows that the web server inside the container answers on port 8080 when reached from the Pi itself. It does not show that the installation finished correctly, that login works, or that any other device can reach the page.

## 6. Remote access

The collaborator cannot currently open GLPI from their own device over Tailscale. This is attributed to the tailnet policy, which at the last recorded review did not grant any path to TCP 8080 on the Pi. This was **not diagnosed**; no policy change or test was made in this phase. Opening TCP 8080 for the intended users requires the administrator, who owns the tailnet policy.

## 7. Follow-ups

| Item | Owner | Status |
| --- | --- | --- |
| Allow TCP 8080 to the Pi for the intended users in the tailnet policy, then test from the collaborator's device. Phase 13's earlier rule covered TCP 9000; check whether authentik's current ports (80 and 443) need the same treatment | Administrator | Pending |
| Confirm the install finished (final greeting in the `glpi` log) | Collaborator | Pending |
| First login; create one named account per person; change or disable the default accounts | Collaborator | Pending |
| Record the exact GLPI version and pin the image tag (or digest) in place of `latest` | Collaborator | Pending |
| Rotate the MySQL root password, which was generated at first start and printed in the `db` container's logs, and store the new one in a password manager or OpenBao | Collaborator | Pending |
| Apply the Phase 14 conventions: categories and groups from Ticket 002, status rules, and back-filling tickets 001 and 002 so the GLPI IDs match the file numbers | Both | Pending |
| Decide where the GLPI database is backed up. The nightly Proxmox job covers VM 100 only, so GLPI on the Pi currently has no backup | Both | Open |
| HTTPS and a hostname through a reverse proxy | Both | Deferred; the lab uses plain HTTP for now |

## 8. Evidence limits

- The architecture check shows the image offers an `arm64` build. It does not prove every GLPI feature works on ARM.
- The container logs showed the installation starting. Completion of the installation and the state of the GLPI database were not inspected.
- The `curl` check was made from the Pi to itself, so it says nothing about firewall rules, Tailscale policy, or reachability from other devices.
- The image is pinned to `latest` for now, so the running GLPI version is not yet recorded.
- No restore or backup test was made for GLPI.
