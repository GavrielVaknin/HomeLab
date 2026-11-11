# Phase 13 — Docker and authentik Deployment on Raspberry Pi

Result: Docker Compose deployment of authentik and PostgreSQL completed on the Raspberry Pi 5. All three containers report healthy. Remote access to authentik over Tailscale was restored after adjusting network access permissions.

> Historical scope: November 8, 2026. This phase documents the Raspberry Pi's transition from a remotely accessible Kali client to a host running containerized services.

## 1. Objective

Deploy a self-hosted identity and access management platform using authentik.

The deployment uses:
- Raspberry Pi 5 (8GB RAM)
- Kali GNU/Linux Rolling
- Docker Engine
- Docker Compose
- PostgreSQL
- authentik
- Tailscale for remote connectivity

## 2. Virtualization considerations

The Proxmox host runs on Intel x86 hardware. Nested virtualization is not enabled in its current configuration.

The Raspberry Pi 5 was selected as an independent ARM64 host.

KVM availability was checked independently on the Raspberry Pi:

    kvm-ok
    ls -l /dev/kvm

Observed result:

    INFO: /dev/kvm exists
    KVM acceleration can be used

    crw-rw----+ 1 root kvm 10, 232 Nov 8 14:51 /dev/kvm

This confirms KVM acceleration availability. An actual KVM virtual machine deployment was not verified in this phase.

Docker does not require KVM to run containers.

## 3. Host configuration

Recorded host information:

| Property | Value |
|---|---|
| Hardware | Raspberry Pi 5 |
| Memory | 8GB |
| OS | Kali GNU/Linux Rolling 2026.3 |
| Architecture | ARM64 (aarch64) |
| Kernel | 6.12.34+rpt-rpi-2712 |
| Docker Engine | 28.5.2+dfsg4 |
| Docker Compose | 2.40.3-3 |

User `gavriel` belongs to:

    sudo
    users
    docker
    kvm

## 4. Docker Compose deployment

Deployment directory:

    ~/Tools/authentik

Compose configuration:

    compose.yml

Three services were deployed:

| Service | Image | Purpose |
|---|---|---|
| postgresql | postgres:16-alpine | Database |
| server | ghcr.io/goauthentik/server:2026.8.5 | Web server |
| worker | ghcr.io/goauthentik/server:2026.8.5 | Background tasks |

### PostgreSQL

- Database: authentik
- Named Docker volume: database
- Container health check: pg_isready
- Restart policy: unless-stopped

### authentik server

- Depends on PostgreSQL passing its health check.
- HTTP port: 9000
- HTTPS port: 9443
- Shared data directory: ./data
- Custom templates: ./custom-templates
- Shared memory allocation: 512MB

### authentik worker

- Depends on PostgreSQL passing its health check.
- Runs as root.
- Mounts the Docker socket.
- Shares the authentik data directory.
- Mounts certificates and custom templates.
- Shared memory allocation: 512MB

### Persistence

| Data | Storage |
|---|---|
| PostgreSQL | Docker named volume: database |
| authentik data | ./data |
| Custom templates | ./custom-templates |
| Certificates | ./certs |

PostgreSQL data is stored in a Docker-managed named volume.

The authentik directories are bind-mounted from the host filesystem.

### Environment configuration

Configuration is loaded from `.env`.

Required variables include:

- PG_PASS
- AUTHENTIK_SECRET_KEY

Additional variables configure the database name, database user, image version and published ports.

Secrets are excluded from this documentation and must not be committed to Git.

### Container restart and dependency behavior

All three services use:

    restart: unless-stopped

The server and worker wait for PostgreSQL's health check to pass before starting.

## 5. Container verification

Command:

    docker ps

Observed containers:

| Container | Status |
|---|---|
| authentik-server-1 | Up (healthy) |
| authentik-worker-1 | Up (healthy) |
| authentik-postgresql-1 | Up (healthy) |

All three containers reported healthy during verification.

## 6. Network configuration

Tailscale provides remote connectivity between the Raspberry Pi and the ThinkPad.

Public documentation uses substituted overlay addresses.

| Device | Tailscale address |
|---|---|
| Raspberry Pi 5 | 100.101.10.10 |
| ThinkPad T470 | 100.101.10.20 |

### Initially published ports

| Port | Protocol | Purpose |
|---|---|---|
| 9000 | TCP | HTTP |
| 9443 | TCP | HTTPS |

The Compose configuration publishes both ports using configurable environment variables.

Docker listening sockets were verified:

    sudo ss -tulpn | grep -E '9000|9443'

Observed:

    0.0.0.0:9000
    0.0.0.0:9443
    [::]:9000
    [::]:9443

The ports are published on all IPv4 and IPv6 interfaces, not exclusively on Tailscale.

## 7. Network troubleshooting

Initially, the authentik web interface was not reachable remotely from the ThinkPad.

Investigation included:

- Docker container health verification.
- Docker port mapping inspection.
- Listening socket inspection.
- Firewall rule inspection.
- Packet capture on tailscale0.
- Remote curl connectivity tests.

Packet capture:

    sudo tcpdump -ni tailscale0 tcp port 9000

The investigation identified a Tailscale network access restriction affecting TCP port 9000.

Sivan was contacted to adjust the relevant network access permissions.

After the policy change, remote access was restored.

Application endpoint:

    http://100.101.10.10:9000

Initially, the Pi published ports 9000 and 9443. In the final configuration, Docker maps port 80 on the Pi to port 9000 in the container, and port 443 to port 9443. After the Tailscale access rule was updated, Gavriel confirmed that he could log in to authentik over HTTPS.

## 8. Security considerations

- `.env` contains secrets and must remain outside version control.
- PostgreSQL credentials and the authentik secret key must never be published.
- The authentik worker runs as root.
- The worker mounts `/var/run/docker.sock`, providing access to the Docker API.
- Docker publishes ports on wildcard interfaces.
- Tailscale access policies should restrict access to the required devices and ports.
- PostgreSQL is not published to a host port in the current Compose configuration.

The Docker socket mount and root execution increase the worker's privileges and should be considered when reviewing the deployment's security.

## 9. References

- authentik First Steps:
  https://docs.goauthentik.io/install-config/first-steps/

- Linux KVM Documentation:
  https://docs.kernel.org/virt/kvm/index.html

- Docker Getting Started:
  https://docs.docker.com/get-started/get-docker/

## 10. End-of-phase state

| Component | Result |
|---|---|
| Raspberry Pi 5 | Operational |
| KVM availability | Verified |
| Docker Engine | Installed |
| Docker Compose | Installed |
| PostgreSQL | Running and healthy |
| authentik server | Running and healthy |
| authentik worker | Running and healthy |
| Tailscale connectivity | Operational |
| Remote HTTPS login | Confirmed over | 

Completed work establishes the Raspberry Pi as an independent container host within the homelab.

Pending work:
- Complete authentik initial configuration.
- Configure users, groups and authentication policies.
- Integrate additional homelab applications.
- Implement and test backups.
- Review Docker socket access and worker privileges.
- Test persistent data recovery.

End-of-phase state: authentik and PostgreSQL are deployed and healthy. Remote HTTPS login over Tailscale was verified. Further IAM configuration and security validation remain open.
