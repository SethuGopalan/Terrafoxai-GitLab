# TerrafoxAI — Self-Hosted GitLab CE

A reproducible, security-conscious deployment of **GitLab Community Edition** with **Docker Compose** and **Cloudflare Tunnel**. The public web interface uses HTTPS while the GitLab origin stays on a restricted private network.

> **Documentation-only example.** This repository deliberately contains no production credentials, tunnel IDs, internal hostnames, or real private IP addresses. Adapt placeholders to your environment before deployment.

## Features

- Self-hosted Git repositories, issues, merge requests, and project management
- Version-pinned GitLab CE Docker image
- Persistent application configuration, logs, and data
- CPU and memory safeguards
- Cloudflare Tunnel for public HTTPS access without inbound router port forwarding
- Private origin connectivity and separate administrator/developer accounts
- Restricted self-registration and hardened Web IDE settings

## Architecture

```text
Developer / Browser
        |
        | HTTPS
        v
Cloudflare Edge
        |
        | Authenticated outbound tunnel
        v
Tunnel Connector (tunnel-host)
        |
        | HTTP over restricted private network
        v
GitLab Origin (gitlab-host:8080)
        |
        v
GitLab CE container
  ├── GitLab Rails / Puma
  ├── Workhorse / Nginx
  ├── PostgreSQL / Redis
  └── Git repositories and persistent storage
```

**Public URL:** https://gitlab.terrafoxai.com

The tunnel connector and GitLab may run on the same host or separate hosts. The examples below default to a **same-host** connector to avoid exposing an unrestricted private-network port. For separate hosts, see [Separate-host deployment](#separate-host-deployment).

## Prerequisites

- Linux server with Docker Engine and Docker Compose plugin
- Sufficient CPU, RAM, and disk space for the intended workload
- A Cloudflare-managed domain and an authenticated Cloudflare Tunnel
- Secure persistent storage and a tested backup destination
- Firewall access to control traffic between the tunnel connector and GitLab

## Directory layout

```text
terrafoxai-gitlab/
├── README.md
├── compose.yaml             # Example, create from this guide
└── data/                    # Local persistent volumes; DO NOT COMMIT
    ├── config/
    ├── logs/
    └── storage/
```

Add `data/`, `.env`, credentials, and private keys to `.gitignore` before committing a repository.

## Docker Compose deployment

Create `compose.yaml` with the following **illustrative configuration**:

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:19.2.6-ce.0
    container_name: gitlab-ce
    hostname: gitlab.terrafoxai.com
    restart: "no"

    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.terrafoxai.com'
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        letsencrypt['enable'] = false
        prometheus_monitoring['enable'] = false

    ports:
      - "127.0.0.1:8080:80"

    volumes:
      - ./data/config:/etc/gitlab
      - ./data/logs:/var/log/gitlab
      - ./data/storage:/var/opt/gitlab

    mem_limit: 5g
    cpus: 2
```

The version is **pinned for reproducibility**, not a claim that it is the newest or currently patched release. Check GitLab release notes and supported upgrade paths before production use. A 5 GiB memory cap is a safeguard, **not a guarantee that all GitLab workloads will fit**; monitor memory and adjust to the host's capacity.

Start the container:

```bash
# Start GitLab in the background
docker compose up -d

# Check container state
docker compose ps
```

First boot and reconfiguration can take several minutes. Avoid repeatedly restarting the container while Puma is still loading.

## Cloudflare Tunnel configuration

An existing authenticated Cloudflare Tunnel can route a hostname to the GitLab origin. For a connector **on the same host**, add an ingress entry before the catch-all rule in the tunnel's configuration:

```yaml
ingress:
  - hostname: gitlab.terrafoxai.com
    service: http://127.0.0.1:8080
  - service: http_status:404
```

**Important:** This is a minimal example. If the tunnel already serves other applications, **preserve their existing ingress rules** and insert only the GitLab rule above the final catch-all. Do not replace the entire production configuration.

Validate and reload the connector:

```bash
# Validate tunnel ingress syntax
cloudflared tunnel ingress validate

# If managed by systemd, reload its service
sudo systemctl restart cloudflared
```

Create or verify the Cloudflare DNS route for `gitlab.terrafoxai.com` using the existing authenticated tunnel configuration. DNS records and ingress routing are **separate** requirements: both must be correct.

### Separate-host deployment

When `tunnel-host` and `gitlab-host` are different machines:

1. Bind the container's HTTP port to an **appropriate private interface**, rather than `127.0.0.1`.
2. Replace the tunnel ingress service with `http://<GITLAB_PRIVATE_ADDRESS>:8080`.
3. Restrict the origin port using firewall rules so **only the tunnel connector** can connect.
4. Do not forward the origin port from the router or expose it publicly.
5. Use a stable private endpoint or internal DNS name, and document it **outside the public repository**.

The placeholder `<GITLAB_PRIVATE_ADDRESS>` must be replaced locally; never commit the actual address to this public example.

## HTTPS behind Cloudflare

GitLab must generate links using its public HTTPS hostname even when the origin listens on private HTTP:

```ruby
external_url 'https://gitlab.terrafoxai.com'
nginx['listen_port'] = 80
nginx['listen_https'] = false
letsencrypt['enable'] = false
```

This is a **TLS termination at Cloudflare** design. The connector-to-origin hop uses HTTP, so it must remain on a trusted, restricted network. For stronger internal transport security, configure HTTPS to the origin separately.

After changing Compose configuration, apply it with:

```bash
# Apply updated container settings
docker compose up -d
```

## Initial setup and security

1. Open https://gitlab.terrafoxai.com and complete initial administrator sign-in.
2. Replace the initial administrator password with a unique strong password; store it in a password manager.
3. Enable multi-factor authentication, especially for administrators.
4. Under **Admin → Settings → General → Sign-up restrictions**, disable **Allow new user accounts** if public registration is not needed.
5. Review and disable **Web IDE single-origin fallback** if it is enabled and a separate secure extension-host setup is not available. This can limit Web IDE functionality.
6. Create a separate **regular user** for daily development; reserve the administrator account for administration.
7. Use private projects/groups by default and restrict invitations appropriately.
8. Keep Docker, GitLab, the tunnel connector, and the host OS updated through planned maintenance.
9. Apply network firewall restrictions and use Cloudflare Access policies if the instance should only be reachable by authorized users.

> A publicly reachable GitLab sign-in page is not equivalent to a private application. Cloudflare Tunnel avoids direct inbound exposure, but access controls, authentication, and updates remain essential.

## Verification

```bash
# Check Docker container status
docker compose ps

# Check bundled GitLab services
docker exec gitlab-ce gitlab-ctl status

# Check local origin response (same-host example)
curl -I --max-time 15 http://127.0.0.1:8080
```

An origin response such as `302 Found` can be normal when redirecting to sign-in. Because the public hostname is HTTPS, redirects and generated links should be checked **through the public URL**, not judged solely from a raw IP-based origin request.

Visit https://gitlab.terrafoxai.com and confirm that the sign-in page loads and that project URLs use HTTPS.

## Troubleshooting

### Cloudflare 502 / Bad Gateway

A 502 can originate from the GitLab backend even when Cloudflare and the tunnel are healthy. Test the origin independently, then inspect services:

```bash
# Check GitLab service supervision
docker exec gitlab-ce gitlab-ctl status

# Inspect Puma startup logs
docker exec gitlab-ce tail -n 40 /var/log/gitlab/puma/current

# Inspect Workhorse logs
docker exec gitlab-ce tail -n 40 /var/log/gitlab/gitlab-workhorse/current
```

If Workhorse reports that `/var/opt/gitlab/gitlab-rails/sockets/gitlab.socket` does not exist, Puma may still be initializing. Wait for a Puma log entry indicating that it is **listening on the Unix socket**, then retest. If the socket never appears, inspect Puma errors and memory pressure before restarting anything.

### DNS name does not resolve

- Confirm the hostname has a DNS record in the **correct Cloudflare zone**.
- Confirm the record points to the intended tunnel.
- Confirm the hostname also has a matching tunnel ingress rule.
- Check domain delegation and authoritative DNS responses if records appear inconsistent.

### GitLab generates HTTP links

Verify `external_url` uses `https://gitlab.terrafoxai.com` and that internal Nginx HTTPS listening is disabled only when TLS is intentionally terminated at the proxy.

### Resource pressure

GitLab can be memory-intensive. Check Docker memory consumption and application logs. Raise limits or tune services based on actual workload and supported GitLab recommendations rather than assuming a 5 GiB cap is universally sufficient.

## Backups and recovery

Back up **both** GitLab application data and its configuration/secrets using GitLab's supported backup procedures. Keep backup files outside the Git repository and preferably off-host. Test restore procedures periodically and record the exact GitLab version required for recovery.

Never commit database contents, GitLab secrets, private keys, tunnel credentials, access tokens, or backup archives.


## Scope

This project is a sanitized deployment guide, **not a live backup or exact export of the production configuration**. Values and examples are intentionally generalized to protect the environment.
