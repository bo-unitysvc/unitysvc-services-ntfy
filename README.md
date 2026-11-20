# UnitySVC ntfy

Service data for [ntfy.sh](https://ntfy.sh) push notifications on the [UnitySVC](https://unitysvc.com) platform.

## Overview

This repo contains the provider, offering, and listing data for the **ntfy Push Notifications** service. `labs/ntfy-sh` is a **container service**: UnitySVC runs the official `binwiederhier/ntfy` image as one private instance per enrollment on the UnitySVC compute cluster, at the customer's expense. Publish messages via simple HTTP POST requests to the instance URL and receive them on mobile (iOS/Android), desktop, or via a streaming API.

**Key features:**

- HTTP POST/PUT publishing (no client library required)
- Real-time subscriptions via WebSocket and JSON stream
- Mobile apps for iOS and Android
- Message priorities (1–5), emoji tags, click actions, and attachments
- Scheduled/delayed delivery
- A private server per enrollment, protected by an access token (`NTFY_ACCESS_TOKEN`)
- Billed per instance-month (list price $5 per month)

## Data Structure

```
services/specs/
└── labs/
    └── ntfy-sh/                       # the folder name IS the service name
        ├── provider.json              # Provider: UnitySVC Labs (`labs`)
        ├── offering.json              # Offering: container channel, entry, params, payout
        ├── listing.json               # Listing: documents and price
        ├── service.json               # Identity sidecar (backend service_id)
        ├── compose.yaml               # Instance spec: the ntfy image and fixed settings
        ├── connectivity.json          # Declarative probes (GET /v1/health)
        ├── usage.md                   # "How to use this service" (public)
        ├── user-guide.md              # Tutorial: publish, subscribe, troubleshooting (public)
        └── images/                    # Screenshots used by the guide
```

The service is published as **`labs/ntfy-sh`** — `listing.name` is the folder's
path under `services/specs/`, and `offering.name` is the bare service name (`ntfy-sh`).

### How the instance is configured

`compose.yaml` holds the fixed, non-secret ntfy settings (cache and auth
databases in `/tmp`, 12 h retention, 4 KB messages, `deny-all` default access,
health check). The offering's `base` channel (`access_method: container`,
`type: enrollable`, tier `nano`) exposes one public entry, `api`, on the ntfy
HTTP port with `auth: app`, so **ntfy itself** authenticates every request.

The enrollment parameter **`NTFY_ACCESS_TOKEN`** (generated default, can be
replaced; ntfy's `tk_` + 29-character format) is injected through the
channel's `environment` as `NTFY_AUTH_TOKENS`, owned by a provisioned `admin`
user whose password is a placeholder hash that nobody knows. Customers
publish and subscribe with `Authorization: Bearer <token>`.

Storage is ephemeral: cached messages are lost when the instance is replaced
or stopped.

### Documents

Documents are defined in `listing.json` and referenced via relative `file_path`:

| Document | Category | Public | Description |
|----------|----------|--------|-------------|
| Connectivity test | `connectivity_test` | No | `container_connectivity` preset |
| Connectivity probes | `connectivity_test` | No | `connectivity.json`: `GET /v1/health` expects `"healthy":true` |
| How to use this service | `getting_started` | Yes | Enroll, token, publish, subscribe, limits |
| User guide | `tutorial` | Yes | Publishing options, subscribing, troubleshooting |

## Setup

Supply your seller credentials (this service needs no seller secrets, so there is no secrets manifest):

```bash

export UNITYSVC_SELLER_API_KEY="<seller-key>"
export UNITYSVC_SELLER_API_URL="https://seller.staging.unitysvc.com/v1/"
```

The CLI ships with [`unitysvc-sellers`](https://github.com/unitysvc/unitysvc-sellers).
Invoke it as `usvc_seller …` if it's on PATH, else
`uvx --from unitysvc-sellers usvc_seller …`.

## Commands

```bash
# Schema + cross-file validation (no network)
usvc_seller specs validate

# Canonical formatting; commit the result
usvc_seller specs format

# Local container test — runs the image in Docker, waits for the health check and probes /v1/health
usvc_seller specs run-tests labs/ntfy-sh --force

# Push to staging (re-upload creates a revision)
usvc_seller specs upload labs/ntfy-sh

# Platform test — starts a temporary instance on the compute cluster and runs the probes
usvc_seller services run-tests labs/ntfy-sh --force
```


## Architecture

```
Publisher (cURL/Python/etc.)
    │  Authorization: Bearer <NTFY_ACCESS_TOKEN>
    ▼
https://<enrollment-code>.<apps-domain>     (one ntfy container per enrollment)
    │
    ├──► Mobile app (iOS/Android)
    ├──► WebSocket clients
    └──► JSON stream subscribers
```

- Requests go **directly to the instance**, not through the UnitySVC API gateway; the UnitySVC API key is not used.
- **Authentication** is ntfy's own access control, seeded at start-up from `NTFY_ACCESS_TOKEN`; anonymous requests are denied.
- **Mobile apps** connect to the instance URL and sign in with the token.
