# Services Overview

This document gives a service-by-service view of the live platform. 

## Shared Edge Layer

### Traefik

**Role**

Shared reverse proxy and TLS termination point for all public services on the host.

**Why it exists**

- centralizes host port exposure to `80/443`
- reads routing config from Docker labels
- makes it possible to run several separate Compose projects behind one ingress layer

**Network placement**

- `traefik-public`

## Automation Stack

### n8n

**Role**

Workflow automation service exposed at `automation.example.com`.

**Why it exists**

- orchestrates automation flows
- receives public webhooks
- depends on PostgreSQL for persistence

**Network placement**

- `n8n_internal`
- `traefik-public`

### PostgreSQL for n8n

**Role**

Persistent datastore for n8n.

**Why it exists**

- keeps workflow state and credentials out of the application container
- uses a health check so n8n does not start before the database is ready

**Network placement**

- `n8n_internal`

### Gotenberg

**Role**

Internal helper service for document or PDF-related processing.

**Why it exists**

- keeps rendering logic separate from the main automation app
- is reachable only from services that need it

**Network placement**

- `n8n_internal`

### offers-app

**Role**

Public helper application exposed at `offers.example.com`.

**Why it exists**

- provides a dedicated workflow-facing web entrypoint
- talks to Gotenberg internally
- forwards into automation flows through n8n webhooks

**Network placement**

- `n8n_internal`
- `traefik-public`

## Ecommerce Stack

### Next.js storefront

**Role**

Public website served on `shop.example.com` and `www.shop.example.com`.

**Why it exists**

- customer-facing frontend
- sits behind Traefik instead of exposing its own host port

**Network placement**

- `mystic-aroma-prod_internal`
- `traefik-public`

### Medusa backend

**Role**

Commerce/API backend exposed at `api.example.com`.

**Why it exists**

- provides backend commerce logic for the storefront
- uses internal PostgreSQL and Redis dependencies

**Network placement**

- `mystic-aroma-prod_internal`
- `traefik-public`

### Strapi admin

**Role**

Admin or CMS interface exposed at `admin.example.com`.

**Why it exists**

- separates content/admin workflows from the public storefront
- still uses the same shared ingress layer as the other public services

**Network placement**

- `mystic-aroma-prod_internal`
- `traefik-public`

### PostgreSQL for ecommerce

**Role**

Private relational datastore for the ecommerce stack.

**Why it exists**

- supports backend application services without public exposure

**Network placement**

- `mystic-aroma-prod_internal`

### Redis

**Role**

Private cache / internal service dependency for the ecommerce stack.

**Why it exists**

- keeps service coordination and caching internal to the stack

**Network placement**

- `mystic-aroma-prod_internal`

## CRM Stack

### CRM frontend

**Role**

User-facing CRM interface served on `crm.example.com`.

**Why it exists**

- keeps the browser-facing app separate from the API implementation
- uses Traefik for hostname-based routing

**Network placement**

- `nma-crm_internal`
- `traefik-public`

### CRM backend

**Role**

Backend API exposed through the same hostname on the `/api` path.

**Why it exists**

- keeps frontend and API responsibilities separated
- uses a more specific Traefik router rule with path matching
- connects internally to MySQL only

**Network placement**

- `nma-crm_internal`
- `traefik-public`

### MySQL

**Role**

Private datastore for the CRM backend.

**Why it exists**

- persists CRM application data without exposing a database port externally

**Network placement**

- `nma-crm_internal`

