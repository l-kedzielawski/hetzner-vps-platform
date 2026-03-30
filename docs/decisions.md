# Architecture Decisions

This project is intentionally small in footprint and explicit in trade-offs. These are the main decisions behind the current setup.

## 1. Traefik as the Reverse Proxy

**Decision**

Use Traefik as the shared edge proxy for all Docker Compose stacks on the VPS.

**Why**

- routing is defined next to each service through Docker labels
- new services can be exposed without manually editing and reloading a central proxy config
- TLS handling stays close to the routing layer
- one reverse proxy can front multiple independent Compose projects cleanly

**Trade-off**

Traefik is less familiar to some teams than Nginx, and its dynamic behavior can feel more implicit if labels are not kept tidy.

## 2. One Shared Public Network, Separate Internal Networks

**Decision**

Use one shared `traefik-public` network for externally reachable services, and a separate internal bridge network per stack.

**Why**

- Traefik only needs one common network to reach public services
- each stack keeps east/west traffic isolated
- databases and helper services stay off the shared edge network
- the boundary between public and private services is obvious in Compose

**Trade-off**

Any service that joins `traefik-public` becomes part of a shared routing surface, so network membership needs to stay deliberate.

## 3. Docker Compose Instead of Kubernetes

**Decision**

Run the platform with Docker Compose on a single Hetzner VPS.

**Why**

- the workload size does not justify Kubernetes complexity
- Compose is fast to audit, debug, and recover on a single host
- fewer moving parts means lower operational overhead for a small production environment
- this fits the actual hosting model: one VPS, multiple contained services, one edge proxy

**Trade-off**

Simpler and faster to operate, but no scheduling, self-healing, or node-level redundancy you'd get from a larger orchestrated platform."

## 4. Keep Databases Internal-Only

**Decision**

Do not publish PostgreSQL, MySQL, or Redis directly on the host.

**Why**

- reduces accidental exposure
- keeps application data paths limited to internal Docker networking
- makes Traefik the only ingress point for public traffic
- matches the principle that storage services should not be internet-facing by default

**Trade-off**

Direct admin access becomes slightly less convenient and usually requires Docker exec, SSH tunnelling, or a temporary internal client container.

## 5. Cloudflare for DNS and Edge, Traefik for Origin Routing

**Decision**

Use Cloudflare for DNS and edge routing, but keep container-aware routing at the origin in Traefik.

**Why**

- DNS stays centralized across domains and subdomains
- the origin server keeps full control over which container receives each request
- the split of responsibilities is clean: Cloudflare handles entry to the server, Traefik handles entry to the application

**Trade-off**

There are two layers involved in request flow, so debugging sometimes means checking both Cloudflare and the origin proxy before the issue is fully explained.

## 6. Document the Real State, Including Gaps

**Decision**

Document the verified live state even when it differs from the intended target design.

**Why**

- honest documentation is more valuable than polished fiction
- the current host uses ACME TLS challenge, not Cloudflare DNS challenge
- `fail2ban` is currently absent, and that is worth stating clearly

**Trade-off**

Document the real state, even when it's not ideal yet."

