# Server Hardening

This document reflects the actual state of the live host."

## Disable Direct Root SSH Login

**What**

Direct root SSH login is disabled.

**Why**

Root is the most valuable account on the machine. Disabling direct root login removes the simplest brute-force target and makes administrative access go through a normal user account first.

**How**

The live host has this in `/etc/ssh/sshd_config`:

```text
PermitRootLogin no
```

No drop-in overrides in /etc/ssh/sshd_config.d/ - the main config is the effective configuration.

## Keep SSH Configuration Small and Explicit

**What**

The visible SSH configuration is close to the Ubuntu baseline, with the strongest confirmed override being `PermitRootLogin no`.

**Why**

Minimal SSH configuration is easier to audit than a stack of partial overrides. On a VPS like this, clarity matters just as much as the setting itself.

**How**

Current State:

- no custom files in `/etc/ssh/sshd_config.d/`
- no explicit `Port` override in the readable SSH config
- `PasswordAuthentication` and `PubkeyAuthentication` were visible only as commented defaults

That means key-only enforcement could not be fully proven from non-root access alone. The right verification command on the host is:

```bash
sudo sshd -T | grep -E 'passwordauthentication|pubkeyauthentication|permitrootlogin|port'
```

## Enable a Host Firewall with UFW

**What**

UFW is enabled and active on the server.

**Why**

Even in a Docker-first setup, the host should still control the exposed surface. For this architecture, the expected public ports are SSH plus Traefik on `80` and `443`.

**How**

The following runtime state was confirmed:

```text
ufw.service -> enabled
ufw.service -> active
```

For this kind of VPS, the expected baseline is:

- allow `22/tcp`
- allow `80/tcp`
- allow `443/tcp`
- deny everything else by default

## Keep Databases Off the Public Edge

**What**

The stacks keep PostgreSQL, MySQL, and Redis on internal Docker networks only.

**Why**

This reduces accidental exposure and keeps the reverse proxy as the only intended public ingress path.

**How**

Verified from the running Docker networks:

- `n8n-postgres` is only on `n8n_internal`
- `crm-mysql` is only on `nma-crm_internal`
- `mystic-postgres` and `mystic-redis` are only on `mystic-aroma-prod_internal`
- only HTTP-facing services join `traefik-public`

That network split is one of the more useful hardening decisions on the host.

## Fail2ban Status

**What**

`fail2ban`  is not currently installed.

**Why**

This matters because it creates a gap between the intended target posture and the actual host state. SSH keys reduce risk, but `fail2ban` is still a useful second layer when a server is publicly reachable.

**How**

The live checks returned:

```text
dpkg -l fail2ban -> no package found
systemctl status fail2ban -> unit not found
```

So it is documented here as missing, not as an assumed control.

If the target setup includes it, the validation flow should be:

```bash
sudo apt update
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

## Practical Hardening Model for This VPS

**What**

The server follows a pragmatic layered model:

- SSH access constrained at the host level
- firewall enabled on the host
- Traefik used as the only public reverse proxy
- databases isolated on internal networks
- public exposure controlled through explicit Docker labels

**Why**

On a single Hetzner VPS hosting several services, the goal is not complexity for its own sake. The goal is to keep exposure deliberate, keep the blast radius small, and make the setup easy to understand later.

**How**

The live network layout supports that model:

- `traefik-public` is the shared public edge
- each stack has its own internal bridge network
- data services stay internal
- only web-facing services join the edge network

