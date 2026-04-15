# Prowler ECS Fargate Deployment Guide

## Prowler Demo Environment Overview

This guide deploys Prowler app on ECS Fargate using the default VPC with public
subnets, an Application Load Balancer, EFS for database persistence, and ECS
Service Connect for service discovery. For production deployments, consider using
PostGreSQL RDS, Elasticache for Valkey, ensure proper secrets management, backup
settings and multi-AZ.

| Service | Image Source | Port | Role |
|---|---|---|---|
| Postgres | `postgres:16.3-alpine3.20` (Docker Hub) | 5432 | Primary database |
| Valkey | `valkey/valkey:7-alpine3.19` (Docker Hub) | 6379 | Celery message broker (Redis-compatible) |
| Neo4j | `graphstack/dozerdb:5.26.3.0` (Docker Hub) | 7687, 7474 | Graph DB for Attack Paths |
| API | `prowlercloud/prowler-api:stable` (Docker Hub) | 8080 | Django REST API |
| Worker | `prowlercloud/prowler-api:stable` (Docker Hub) | — | Celery worker for scans |
| Worker-Beat | `prowlercloud/prowler-api:stable` (Docker Hub) | — | Celery scheduler |
| MCP Server | `prowlercloud/prowler-mcp:stable` (Docker Hub) | 8000 | AI assistant features |
| UI | Custom ECR image (see below) | 3000 | Next.js frontend |

### Why the UI Needs a Custom Image (No Code Changes Required)

The only custom image in this deployment is the UI. No application code was modified. The
custom image uses the exact same source code and Dockerfile from the Prowler repository.
The only difference is the build argument value passed during `docker build`.

**The problem:**

The published `prowlercloud/prowler-ui:stable` image is built by the Prowler CI pipeline
(`.github/workflows/ui-container-build-push.yml`) with this build arg:

```
NEXT_PUBLIC_API_BASE_URL=http://prowler-api:8080/api/v1
```

Next.js inlines `NEXT_PUBLIC_*` environment variables as string literals into the compiled
JavaScript during `next build`. This means the URL `http://prowler-api:8080/api/v1` is
hardcoded into the JS bundle and cannot be overridden by setting environment variables at
runtime. This is standard Next.js behavior, not a Prowler bug.

The hostname `prowler-api` works in docker-compose because Docker Compose creates a network
where bare service names resolve automatically (the API service has `hostname: "prowler-api"`
in docker-compose.yml). In ECS with Service Connect, hostnames require the namespace suffix
(e.g., `prowler-api.prowler`). Bare names like `prowler-api` do not resolve.

This is a known issue: https://github.com/prowler-cloud/prowler/issues/8211

**Workaround:**

Rebuilt the UI image with a different build arg value:

```
NEXT_PUBLIC_API_BASE_URL=http://prowler-api.prowler:8080/api/v1
```

This is the same build process the Prowler team uses, just with a URL that includes the
ECS Service Connect namespace suffix (`.prowler`). The Dockerfile exposes
`NEXT_PUBLIC_API_BASE_URL` as a build arg specifically for this customization.

**What this affects:**

The `NEXT_PUBLIC_API_BASE_URL` value (via `apiBaseUrl` in `ui/lib/helper.ts`) is used by
every Next.js server action that calls the Django API. This includes sign-up, sign-in,
token refresh, provider management, scan management, compliance, findings, roles,
invitations, integrations, attack paths, and Lighthouse AI — essentially every feature.

It is also used client-side in one place: the SAML SSO configuration form displays the
ACS URL using this value. If you set up SAML, the displayed ACS URL will show the internal
hostname. You would manually enter the correct public URL in your identity provider instead
of copying the displayed one. There is no cryptographic signature involved. The ACS URL is
just a callback endpoint configured on both sides independently.

**What was NOT changed:**

- Zero lines of application code were modified
- All other images (API, worker, worker-beat, MCP server) use the published Docker Hub
  images unmodified
- The Dockerfile itself was not modified
- All behavioral differences come from environment variables in ECS task definitions

### How the Microservices Communicate

Understanding the communication flow explains why certain hostnames and configurations matter.

**Browser → ALB → UI / API (external traffic):**

The browser only talks to the ALB. Path-based routing sends `/api/v1/*` to the API container
and everything else to the UI container. Auth.js routes (`/api/auth/*`) must go to the UI,
which is why the ALB rule uses `/api/v1/*` not `/api/*`.

**UI → API (server-side, internal):**

When a user performs any action (sign up, sign in, view findings, run scans), the browser
POSTs to the UI's Next.js server action. The server action then calls the Django API using
the URL baked into the JS bundle (`http://prowler-api.prowler:8080/api/v1`). This call
happens inside the UI container, goes through the Service Connect Envoy sidecar, and reaches
the API container. The user's JWT token is passed along in the Authorization header.

**UI → MCP Server → API (Lighthouse AI):**

When using Lighthouse AI, the UI's LangChain agent calls the MCP server at
`http://mcp-server.prowler:8000/mcp` (runtime env var `PROWLER_MCP_SERVER_URL`). The MCP
server receives tool execution requests with the user's JWT token. For `prowler_app_*` tools,
the MCP server calls the API at `API_BASE_URL` (`http://prowler-api.prowler:8080/api/v1`)
passing the JWT token. For `prowler_hub_*` and `prowler_docs_*` tools, the MCP server calls
external services (hub.prowler.com, Mintlify) directly — no API auth needed.

**API / Worker / Beat → Postgres, Valkey, Neo4j (data layer):**

The Python services (API, Worker, Beat) connect to the data stores using runtime environment
variables (`POSTGRES_HOST=postgres.prowler`, `VALKEY_HOST=valkey.prowler`,
`NEO4J_HOST=neo4j.prowler`). These resolve via Cloud Map DNS (not Service Connect) to avoid
the Envoy HTTP proxy interfering with non-HTTP protocols.

**Worker ↔ Valkey (task queue):**

The Celery worker and beat scheduler communicate through Valkey as a message broker. Beat
publishes scheduled tasks to Valkey queues. Workers consume tasks from those queues. The API
also publishes tasks (e.g., when a user triggers a scan) to Valkey for workers to pick up.

**Summary of hostname resolution methods:**

| Connection | Hostname | Resolution Method | Why |
|---|---|---|---|
| UI → API | `prowler-api.prowler` | Service Connect | Baked into UI image, HTTP protocol works with Envoy |
| UI → MCP | `mcp-server.prowler` | Service Connect | Runtime env var, HTTP protocol |
| MCP → API | `prowler-api.prowler` | Service Connect | Runtime env var `API_BASE_URL`, HTTP protocol |
| API → Postgres | `postgres.prowler` | Cloud Map DNS | Runtime env var, binary protocol (no Envoy) |
| API → Valkey | `valkey.prowler` | Cloud Map DNS | Runtime env var, Redis protocol (no Envoy) |
| API → Neo4j | `neo4j.prowler` | Cloud Map DNS | Runtime env var, Bolt protocol (no Envoy) |

### Key Networking Decisions

**Data services (Postgres, Valkey, Neo4j) use Cloud Map DNS without Service Connect.**
Service Connect adds an Envoy HTTP proxy sidecar. Postgres, Valkey, and Neo4j use binary
protocols (not HTTP), and the Envoy proxy interferes with them. The `appProtocol` field
only accepts `http`, `http2`, or `grpc` — there is no `tcp` option. Removing `appProtocol`
should default to TCP passthrough, but this requires CLI (the console forces a value).
Cloud Map DNS avoids the proxy entirely.

**App services (API, MCP) use Service Connect as Client and Server.**
These are HTTP services that benefit from Service Connect's proxy and service discovery.

**Worker, Worker-Beat, UI use Service Connect as Client Side Only.**
They call other services but nothing calls into them (ALB handles UI inbound traffic directly).

**ALB path routing: `/api/v1/*` not `/api/*`.**
Next.js Auth.js uses `/api/auth/*` routes. If the ALB routes `/api/*` to the Django API,
auth callbacks break. Use `/api/v1/*` so only Django API calls go to the API container.

---

## Prerequisites

- AWS account with permissions for ECS, EFS, ALB, IAM, ECR, CloudWatch
- Default VPC with default subnets (public, one per AZ)
- Docker installed locally (for building the custom UI image)
- AWS CLI configured (for ECR push)
- A domain name with DNS pointing to the ALB (optional but recommended)
- ACM certificate for HTTPS (optional but recommended)

---

## Step 1: Security Groups

Go to **VPC → Security Groups → Create security group**. Create four in your default VPC.

### 1.1 — ALB Security Group
- Name: `prowler-alb-sg`
- Inbound rules:
  - HTTPS (443) from your IP (or `0.0.0.0/0` if public access is acceptable)
  - HTTP (80) from your IP

### 1.2 — App Security Group
- Name: `prowler-app-sg`
- Inbound rules:
  - Custom TCP 8080 from `prowler-alb-sg`
  - Custom TCP 3000 from `prowler-alb-sg`
  - Custom TCP 8000 from `prowler-alb-sg`
  - All TCP from **itself** (`prowler-app-sg`) — for inter-container communication

### 1.3 — Data Security Group
- Name: `prowler-data-sg`
- Inbound rules:
  - Custom TCP 5432 from `prowler-app-sg`
  - Custom TCP 6379 from `prowler-app-sg`
  - Custom TCP 7687 from `prowler-app-sg`
  - Custom TCP 7474 from `prowler-app-sg`

### 1.4 — EFS Security Group
- Name: `prowler-efs-sg`
- Inbound rules:
  - NFS (2049) from `prowler-data-sg`

All security groups: leave outbound as default (all traffic).

---

## Step 2: EFS File System

Go to **EFS → Create file system → Customize**.

### 2.1 — Create the file system
- Name: `prowler-data`
- Storage class: One Zone (cheaper) or Standard
- Automatic backups: disable (reason: setup for demo use only)
- Encryption: enable
- Performance: Bursting

### 2.2 — Network
- Mount targets: select your default VPC subnets
- Security group: `prowler-efs-sg` on each mount target

### 2.3 — Access Points

Create two access points after the file system is created:

**Postgres access point:**
- Root directory path: `/postgres-data`
- POSIX user: UID `70`, GID `70`
- Root directory creation permissions: Owner UID `70`, GID `70`, Permissions `0755`

**Neo4j access point:**
- Root directory path: `/neo4j-data`
- POSIX user: UID `7474`, GID `7474`
- Root directory creation permissions: Owner UID `7474`, GID `7474`, Permissions `0755`

Note the file system ID and both access point IDs.

Do not enable any EFS policy settings (prevent root access, read-only, etc.) They
interfere with how the containers initialize their data directories.

---

## Step 3: CloudWatch Log Group

Go to **CloudWatch → Log groups → Create log group**.
- Name: `/ecs/prowler`
- Retention: 7 days

(Each task definition can also auto-create its own log group if you prefer separate groups.)

---

## Step 4: ECS Cluster

Go to **ECS → Clusters → Create cluster**.
- Cluster name: `prowler`
- Infrastructure: AWS Fargate only
- Namespace: `prowler` (creates a Cloud Map namespace for service discovery)

---

## Step 5: IAM Roles

### Task Execution Role
If `ecsTaskExecutionRole` doesn't exist, create it:
- Trusted entity: AWS Service → Elastic Container Service → Elastic Container Service Task
- Attach policy: `AmazonECSTaskExecutionRolePolicy`

This role also needs ECR pull permissions (included in the managed policy).

### Task Role (optional)
Create a task role if you need ECS Exec or S3 output:
- Same trust policy as above
- For ECS Exec, add inline policy:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## Step 6: Build and Push Custom UI Image

### 6.1 — Create ECR Repository
Go to **ECR → Create repository**:
- Visibility: Private
- Name: `prowler-ui` (or your preferred name)

### 6.2 — Build the Image
From the root of the Prowler repo:
```bash
docker build --target prod \
  --build-arg NEXT_PUBLIC_API_BASE_URL=http://prowler-api.prowler:8080/api/v1 \
  --build-arg NEXT_PUBLIC_PROWLER_RELEASE_VERSION=v5.16.0 \
  -t <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<REPO_NAME>:stable \
  -f ui/Dockerfile \
  ui/
```

The `NEXT_PUBLIC_API_BASE_URL` must match the Service Connect discovery name you'll use
for the API service (`prowler-api`) plus the namespace suffix (`.prowler`).

### 6.3 — Push to ECR
```bash
aws ecr get-login-password --region <REGION> | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com

docker push <ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<REPO_NAME>:stable
```

---

## Step 7: Task Definitions

Common settings for all task definitions:
- Launch type: AWS Fargate
- OS/Architecture: Linux/X86_64 (or ARM64 for cost savings — all images support both)
- Task execution role: your execution role
- Network mode: awsvpc

### 7.1 — Postgres
- CPU: 0.5 vCPU, Memory: 1 GB (or larger)
- Container name: `postgres`
- Image: `postgres:16.3-alpine3.20`
- Port: 5432 TCP (name: `postgres`, **no appProtocol** — remove via JSON if console forces a value)
- Environment variables:
  - `POSTGRES_USER` = `prowler_admin`
  - `POSTGRES_PASSWORD` = `<your-password>`
  - `POSTGRES_DB` = `prowler_db`
- Health check: `CMD-SHELL,pg_isready -U prowler_admin -d prowler_db`
  - Interval: 10, Timeout: 5, Retries: 5, Start period: 30
- Volume: EFS, your file system ID, postgres access point, transit encryption enabled
  - Mount: `/var/lib/postgresql/data`

### 7.2 — Valkey
- CPU: 0.25 vCPU, Memory: 0.5 GB
- Container name: `valkey`
- Image: `valkey/valkey:7-alpine3.19`
- Port: 6379 TCP (name: `valkey`, **no appProtocol**)
- Health check: `CMD-SHELL,valkey-cli ping`
  - Interval: 10, Timeout: 5, Retries: 3, Start period: 10
- No volumes

### 7.3 — Neo4j
- CPU: 1 vCPU, Memory: 3 GB
- Container name: `neo4j`
- Image: `graphstack/dozerdb:5.26.3.0`
- Ports:
  - 7687 TCP (name: `neo4j-bolt`, **no appProtocol**)
  - 7474 TCP (name: `neo4j-http`, appProtocol: `http` is fine for this one)
- Environment variables:
  - `NEO4J_AUTH` = `neo4j/<your-password>`
  - `NEO4J_dbms_max__databases` = `1000`
  - `NEO4J_server_memory_pagecache_size` = `512M`
  - `NEO4J_server_memory_heap_initial__size` = `512M`
  - `NEO4J_server_memory_heap_max__size` = `512M`
  - `NEO4J_PLUGINS` = `["apoc"]`
  - `NEO4J_dbms_security_procedures_allowlist` = `apoc.*`
  - `dbms.connector.bolt.listen_address` = `0.0.0.0:7687`
- Health check: `CMD-SHELL,wget --no-verbose -O /dev/null http://localhost:7474 || exit 1`
  - Interval: 15, Timeout: 10, Retries: 10, Start period: 60
- Volume: EFS, your file system ID, neo4j access point, transit encryption enabled
  - Mount: `/data`


### 7.4 — API
- CPU: 0.5 vCPU, Memory: 1 GB (or larger)
- Container name: `api`
- Image: `prowlercloud/prowler-api:stable`
- Port: 8080 TCP (name: `api`, appProtocol: `http`)
- Entry point (JSON): `["/home/prowler/docker-entrypoint.sh", "prod"]`
- Environment variables:
  - `DJANGO_SETTINGS_MODULE` = `config.django.production`
  - `DJANGO_ALLOWED_HOSTS` = `*`
  - `DJANGO_BIND_ADDRESS` = `0.0.0.0`
  - `DJANGO_PORT` = `8080`
  - `DJANGO_DEBUG` = `False`
  - `DJANGO_LOGGING_FORMATTER` = `human_readable`
  - `DJANGO_LOGGING_LEVEL` = `INFO`
  - `DJANGO_WORKERS` = `2`
  - `DJANGO_MANAGE_DB_PARTITIONS` = `True`
  - `DJANGO_SECRETS_ENCRYPTION_KEY` = `<generate: openssl rand -base64 32>`
  - `DJANGO_TOKEN_SIGNING_KEY` = (empty — auto-generated on first boot)
  - `DJANGO_TOKEN_VERIFYING_KEY` = (empty)
  - `DJANGO_ACCESS_TOKEN_LIFETIME` = `30`
  - `DJANGO_REFRESH_TOKEN_LIFETIME` = `1440`
  - `DJANGO_CACHE_MAX_AGE` = `3600`
  - `DJANGO_STALE_WHILE_REVALIDATE` = `60`
  - `DJANGO_BROKER_VISIBILITY_TIMEOUT` = `86400`
  - `DJANGO_THROTTLE_TOKEN_OBTAIN` = `50/minute`
  - `DJANGO_CORS_ALLOWED_ORIGINS` = `https://<your-domain>` (must include `https://`)
  - `POSTGRES_HOST` = `postgres.prowler`
  - `POSTGRES_PORT` = `5432`
  - `POSTGRES_ADMIN_USER` = `prowler_admin`
  - `POSTGRES_ADMIN_PASSWORD` = `<your-password>`
  - `POSTGRES_USER` = `prowler_admin`
  - `POSTGRES_PASSWORD` = `<your-password>`
  - `POSTGRES_DB` = `prowler_db`
  - `VALKEY_HOST` = `valkey.prowler`
  - `VALKEY_PORT` = `6379`
  - `VALKEY_DB` = `0`
  - `NEO4J_HOST` = `neo4j.prowler`
  - `NEO4J_PORT` = `7687`
  - `NEO4J_USER` = `neo4j`
  - `NEO4J_PASSWORD` = `<your-neo4j-password>`
- Health check: `CMD-SHELL,wget --no-verbose --spider http://localhost:8080/api/v1/docs || exit 1`
  - Interval: 15, Timeout: 5, Retries: 5, Start period: 120

> Note: `DJANGO_ALLOWED_HOSTS=*` is used because the ALB health check sends the container's
> private IP as the Host header, which changes on every task restart. The ALB and security
> groups provide the perimeter security. `DJANGO_CORS_ALLOWED_ORIGINS` (locked to your domain)
> prevents cross-origin browser abuse.

### 7.5 — Worker
- CPU: 1 vCPU, Memory: 2 GB
- Container name: `worker`
- Image: `prowlercloud/prowler-api:stable`
- No port mappings
- Entry point (JSON): `["/home/prowler/docker-entrypoint.sh", "worker"]`
- Environment variables: **same as API**
- Ulimits (JSON): `[{"name": "nofile", "softLimit": 65536, "hardLimit": 65536}]`
- No health check

### 7.6 — Worker-Beat
- CPU: 0.25 vCPU, Memory: 0.5 GB
- Container name: `worker-beat`
- Image: `prowlercloud/prowler-api:stable`
- No port mappings
- Entry point (JSON): `["../docker-entrypoint.sh", "beat"]`
- Environment variables: **same as API**
- Ulimits: same as Worker
- No health check

> Never run more than 1 beat instance. Always set desired count to exactly 1.

### 7.7 — MCP Server
- CPU: 0.25 vCPU, Memory: 0.5 GB
- Container name: `mcp-server`
- Image: `prowlercloud/prowler-mcp:stable`
- Port: 8000 TCP (name: `mcp`, appProtocol: `http`)
- Command (JSON, not entry point): `["uvicorn", "--host", "0.0.0.0", "--port", "8000"]`
- Environment variables:
  - `PROWLER_MCP_TRANSPORT_MODE` = `http`
- Health check: `CMD-SHELL,wget -q -O /dev/null http://127.0.0.1:8000/health || exit 1`
  - Interval: 10, Timeout: 5, Retries: 3, Start period: 15

### 7.8 — UI
- CPU: 0.25 vCPU, Memory: 0.5 GB (or larger)
- Container name: `ui`
- Image: `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<REPO_NAME>:stable` (your custom ECR image)
- Port: 3000 TCP (name: `ui`, appProtocol: `http`)
- Environment variables:
  - `NEXT_PUBLIC_API_BASE_URL` = `http://prowler-api.prowler:8080/api/v1` (cosmetic — baked into image)
  - `API_BASE_URL` = `http://prowler-api.prowler:8080/api/v1` (cosmetic — not used by code)
  - `AUTH_URL` = `https://<your-domain>` (runtime, used by Auth.js)
  - `AUTH_SECRET` = `<generate: openssl rand -base64 32>` (runtime)
  - `AUTH_TRUST_HOST` = `true` (runtime)
  - `NEXT_PUBLIC_API_DOCS_URL` = `https://<your-domain>/api/v1/docs` (baked, cosmetic)
  - `PROWLER_MCP_SERVER_URL` = `http://mcp-server.prowler:8000/mcp` (runtime)
  - `NEXT_PUBLIC_PROWLER_RELEASE_VERSION` = `v5.16.0` (baked, cosmetic)
  - `UI_PORT` = `3000` (runtime)
- No health check needed (the Dockerfile sets `HOSTNAME=0.0.0.0` but the ALB health check
  on the target group handles liveness)

> The `NEXT_PUBLIC_*` env vars are baked into the image at build time and ignored at runtime.
> They are included in the task definition for documentation purposes only.

---

## Step 8: Application Load Balancer

### 8.1 — Create ALB
Go to **EC2 → Load Balancers → Create → Application Load Balancer**.
- Name: `prowler-alb`
- Scheme: Internet-facing
- Mappings: at least 2 AZs
- Security group: `prowler-alb-sg`
- Listener: HTTP:80 (add HTTPS:443 later with ACM certificate)

### 8.2 — Create Target Groups
Create three target groups (Target type: IP, VPC: default, do not register targets):

| Target Group | Protocol/Port | Health Check Path | Success Codes |
|---|---|---|---|
| `prowler-api-tg` | HTTP 8080 | `/api/v1/docs` | 200 |
| `prowler-ui-tg` | HTTP 3000 | `/` | 200,302,307 |
| `prowler-mcp-tg` | HTTP 8000 | `/health` | 200 |

> The UI returns 302 redirects on `/` (to sign-in page), so include 302 and 307 in success codes.

### 8.3 — Listener Rules
On the HTTP:80 listener (or HTTPS:443 if configured):

| Priority | Condition | Action |
|---|---|---|
| 1 | Path pattern = `/api/v1/*` | Forward to `prowler-api-tg` |
| Default | — | Forward to `prowler-ui-tg` |

> Important: Use `/api/v1/*` not `/api/*`. Next.js Auth.js uses `/api/auth/*` routes that
> must be handled by the UI container, not the Django API.

Add the MCP target group only if you need browser-direct MCP access (usually not needed).

---

## Step 9: ECS Services

### Service Connect Configuration Summary

| Service | Service Connect Mode | Discovery Name | Port |
|---|---|---|---|
| Postgres | None (Cloud Map DNS only) | `postgres` | 5432 |
| Valkey | None (Cloud Map DNS only) | `valkey` | 6379 |
| Neo4j | None (Cloud Map DNS only) | `neo4j` (bolt), `neo4j-http` (http) | 7687, 7474 |
| API | Client and Server | `prowler-api` | 8080 |
| MCP | Client and Server | `mcp-server` | 8000 |
| Worker | Client Side Only | — | — |
| Worker-Beat | Client Side Only | — | — |
| UI | Client Side Only | — | — |

> The API discovery name must be `prowler-api` (not `api`) to match the hostname baked into
> the custom UI image: `http://prowler-api.prowler:8080/api/v1`.

### Launch Order

**Phase 1 — Data services (no dependencies):**

### 9.1 — Postgres
- Task definition: `prowler-postgres`
- Desired tasks: 1
- Service Connect: None — use Service Discovery (Cloud Map DNS)
  - Namespace: `prowler`
  - Service discovery name: `postgres`
  - DNS record type: A
- Security group: `prowler-data-sg`
- Public IP: Enabled

### 9.2 — Valkey
- Same pattern as Postgres
- Service discovery name: `valkey`

### 9.3 — Neo4j
- Same pattern as Postgres
- Service discovery name: `neo4j`

**Wait for all three to show healthy before proceeding.**

**Phase 2 — API (needs Postgres, Valkey, Neo4j):**

### 9.4 — API
- Task definition: `prowler-api`
- Desired tasks: 1
- Service Connect: Client and Server
  - Namespace: `prowler`
  - Port alias: `api`
  - Discovery name: `prowler-api`
  - Port: 8080
- Security group: `prowler-app-sg`
- Public IP: Enabled
- Load balancer: `prowler-alb`, target group `prowler-api-tg`, container `api:8080`
- Health check grace period: 180 seconds (migrations run on first boot)

**Wait for API to show healthy.**

**Phase 3 — Workers, MCP, UI:**

### 9.5 — Worker
- Task definition: `prowler-worker`
- Desired tasks: 1
- Service Connect: Client Side Only
- Security group: `prowler-app-sg`
- Public IP: Enabled
- No load balancer

### 9.6 — Worker-Beat
- Task definition: `prowler-worker-beat`
- Desired tasks: 1 (never more)
- Service Connect: Client Side Only
- Security group: `prowler-app-sg`
- Public IP: Enabled
- No load balancer

### 9.7 — MCP Server
- Task definition: `prowler-mcp`
- Desired tasks: 1
- Service Connect: Client and Server
  - Discovery name: `mcp-server`
  - Port: 8000
- Security group: `prowler-app-sg`
- Public IP: Enabled
- Load balancer: `prowler-alb`, target group `prowler-mcp-tg`, container `mcp-server:8000`

### 9.8 — UI
- Task definition: `prowler-ui`
- Desired tasks: 1
- Service Connect: Client Side Only
- Security group: `prowler-app-sg`
- Public IP: Enabled
- Load balancer: `prowler-alb`, target group `prowler-ui-tg`, container `ui:3000`

---

## Step 10: Post-Deployment

### DNS and HTTPS
1. Point your domain to the ALB (Route 53 Alias record or CNAME)
2. Add ACM certificate to the ALB HTTPS:443 listener
3. Add HTTP→HTTPS redirect rule on the HTTP:80 listener

### Update API CORS
After confirming your domain, ensure `DJANGO_CORS_ALLOWED_ORIGINS` in the API task
definition matches your domain exactly (including `https://`).

### First Login
Navigate to `https://<your-domain>` and sign up with email and password.
There is no default admin account — the first user you create becomes the tenant owner.

---

## Troubleshooting

### "Network error or server is unreachable" on sign-up/sign-in
The UI's server actions can't reach the API. Verify:
1. The API service's Service Connect discovery name is `prowler-api`
2. The custom UI image was built with `NEXT_PUBLIC_API_BASE_URL=http://prowler-api.prowler:8080/api/v1`
3. ECS Exec into the UI container and test: `wget -O - http://prowler-api.prowler:8080/api/v1/docs`

### 404 on `/api/auth/session`
The ALB path rule is routing Auth.js requests to Django. Change the rule from `/api/*` to `/api/v1/*`.

### Postgres/Valkey connection errors mentioning "SSL negotiation" or "HTTP/1.1 400"
The data service has `appProtocol: http` set, causing the Service Connect Envoy proxy to
interpret the binary protocol as HTTP. Remove `appProtocol` from the port mapping (requires
JSON editing or CLI) or switch the service to Cloud Map DNS without Service Connect.

### Worker/Beat shows "unknown" health status
Expected — these containers don't expose HTTP endpoints and have no health check defined.
Container status "running" is what matters.

### UI health check fails / task keeps restarting
The Next.js standalone server binds to the container hostname by default, not `0.0.0.0`.
The custom image's Dockerfile sets `ENV HOSTNAME="0.0.0.0"` which fixes this. If using the
published image, add `HOSTNAME=0.0.0.0` as an environment variable.

### Scans stuck in "queued" state
Scans may appear as "queued" in the UI but never execute. This happens when the worker
receives the Celery task but crashes or restarts before the scan begins. The scan record
is created in the database with state `available` and `started_at = None`, but no worker
picks it up again.

To diagnose, ECS Exec into the API container and check for stuck scans:
```
/home/prowler/.cache/pypoetry/virtualenvs/prowler-api-NnJNioq7-py3.12/bin/python manage.py shell -c "
from api.models import Scan
for s in Scan.objects.using('admin').filter(state='available', started_at__isnull=True):
    print(f'ID: {s.id}, State: {s.state}, Started: {s.started_at}')
"
```

Before deleting, verify these scans have no associated findings or output:
```
/home/prowler/.cache/pypoetry/virtualenvs/prowler-api-NnJNioq7-py3.12/bin/python manage.py shell -c "
from api.models import Scan, Finding
stuck = Scan.objects.using('admin').filter(state='available', started_at__isnull=True)
for s in stuck:
    finding_count = Finding.objects.using('admin').filter(scan_id=s.id).count()
    print(f'ID: {s.id}, Findings: {finding_count}, Started: {s.started_at}')
"
```

Only delete scans that show `Findings: 0` and `Started: None`:
```
/home/prowler/.cache/pypoetry/virtualenvs/prowler-api-NnJNioq7-py3.12/bin/python manage.py shell -c "
from api.models import Scan
deleted, _ = Scan.objects.using('admin').filter(state='available', started_at__isnull=True).delete()
print(f'Deleted {deleted} empty scans')
"
```

After cleanup, restart the worker service (scale to 0 then back to 1) and trigger a new
scan from the UI.

### DisallowedHost errors in API logs
The ALB health check sends the container's private IP as the Host header. Use
`DJANGO_ALLOWED_HOSTS=*` since the ALB and security groups handle perimeter security.


### Reset User Password
Use the below command to reset user password from ECS Connect in the api container task.
```
/home/prowler/.cache/pypoetry/virtualenvs/prowler-api-NnJNioq7-py3.12/bin/python manage.py shell -c "from api.models import User; u = User.objects.using('admin').get(email='<user_email_address>'); u.set_password('NewPassword123!'); u.save(using='admin'); print('Done')"

```
Run the below command from the database container task to check if the user exists in the database.

```
SELECT id, email, name, is_active FROM users;
```


---

## Reset Database (Clean Slate)

To wipe all data and start fresh:

1. Scale all services to 0 (app services first, then data services)
2. Delete the EFS file system entirely (or delete and recreate the access points)
3. Create a new EFS file system with the same configuration (security group, mount targets, encryption)
4. Create new access points with the same settings (same UIDs, GIDs, paths, permissions as Step 2.3)
5. Update the Postgres and Neo4j task definitions with the new file system ID and access point IDs
6. Scale services back up in order (data → API → workers/MCP/UI)

Postgres initializes a fresh database when it starts with an empty data directory.
The API entrypoint runs migrations automatically, recreating all tables.

---

## Scale to Zero (Save Money)

Set desired count to 0 for all services when not using the environment.
EFS keeps data safe ($0.30/GB/month for stored data only).

To restart, bring up services in order: data services → API → workers/MCP/UI.

The ALB costs ~$16/month even with no targets. Delete it if the environment will be
unused for extended periods.

Enjoy! Prowler is AWS-some
