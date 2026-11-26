

# Docker Infrastructure Setup & Deployment Documentation

## 1. Overview

The application infrastructure runs on an Azure Ubuntu VM using Docker Compose.

The setup supports two independent environments:

| Environment | Purpose |

|------------|---------|

| Staging | Automatically updated when changes are merged into `main` |

| Pre-production | Automatically updated when a semantic version release (`vX.Y.Z`) is published |

---

## Technology Stack

| Component | Purpose |

|------------|---------|

| Azure VM | Hosting infrastructure |

| Ubuntu | Operating system |

| Docker Engine | Container runtime |

| Docker Compose | Container orchestration |

| Azure Container Registry (ACR) | Docker image registry |

| GitHub Actions | CI/CD pipelines |

| Caddy | Reverse proxy and HTTPS |

| Watchtower | Automatic container updates |

| PostgreSQL | Database |

| Strapi | CMS backend |

| NGINX | Frontend static asset server |

---

# 2. Infrastructure Repository

Infrastructure configuration is maintained separately from the frontend and CMS repositories.

Directory structure:

```text

kiel-sealevel-infra/

│

├── README.md

├── .env

├── compose.yaml

│

├── caddy/

│   ├── Caddyfile

│   └── compose.yaml

│

├── cms/

│   └── compose.yaml

│

├── frontend/

│   └── compose.yaml

│

└──  watchtower/

    └── compose.yaml

```

Each component maintains its own Docker Compose configuration.

The root `compose.yaml` defines shared infrastructure resources:

- Docker networks

- Persistent volumes

- Environment separation

---

# 3. Docker Networking

Each environment has isolated Docker networks.

| Network | Purpose |

|----------|---------|

| staging_proxy | Caddy and staging applications |

| preprod_proxy | Caddy and pre-production applications |

| staging_internal | Private staging communication |

| preprod_internal | Private pre-production communication |

The internal networks isolate application services and databases from public access.

Only containers that need external traffic should be attached to the proxy networks.

---

# 4. Deployment Model

All deployment commands must be executed from the repository root:

```bash

cd ~/kiel-sealevel-infra

```

The root directory contains the shared:

```text

.env

```

file that provides environment variables used by the component Compose configurations.

The component compose files are not intended to be executed independently from their own directories.

---

## Create Shared Infrastructure Resources

Create the external networks and volumes defined by the root compose file:

```bash

docker compose create

```

This creates the required:

- networks

- persistent volumes

without starting application containers.

---

## Start Services

Start Caddy:

```bash

docker compose -f caddy/compose.yaml up -d

```

Start CMS:

```bash

docker compose -f cms/compose.yaml up -d

```

Start Frontend:

```bash

docker compose -f frontend/compose.yaml up -d

```

Start Watchtower:

```bash

docker compose -f watchtower/compose.yaml up -d

```

These commands must be executed from the repository root so the shared `.env` file is available.
---

# 5. Public Architecture

The Azure VM hostname is:

```text
https://kiel-sealevel.westeurope.cloudapp.azure.com
```

Both frontend and CMS share the same hostname.
Caddy acts as the single public entrypoint.

---

## Public URLs

### Pre-production

```text
https://kiel-sealevel.westeurope.cloudapp.azure.com
```

### Staging

```text
https://kiel-sealevel.westeurope.cloudapp.azure.com:8443
```

---

## Routing Architecture

### Pre-production

```text

https://kiel-sealevel.westeurope.cloudapp.azure.com/

    -> Frontend

https://kiel-sealevel.westeurope.cloudapp.azure.com/cms/admin

    -> Strapi Administration

https://kiel-sealevel.westeurope.cloudapp.azure.com/cms/api/*

    -> Strapi APIs

https://kiel-sealevel.westeurope.cloudapp.azure.com/cms/uploads/*

    -> Strapi Media Assets

```

### Staging

```text

https://kiel-sealevel.westeurope.cloudapp.azure.com:8443/

    -> Frontend

https://kiel-sealevel.westeurope.cloudapp.azure.com:8443/cms/admin

    -> Strapi Administration

https://kiel-sealevel.westeurope.cloudapp.azure.com:8443/cms/api/*

    -> Strapi APIs

https://kiel-sealevel.westeurope.cloudapp.azure.com:8443/cms/uploads/*

    -> Strapi Media Assets

```

---

## Architecture Diagram

```text

Internet
    |
    v
  Caddy
    |
    +---------------------------+
    |                           |
    v                           v
Frontend                    Strapi
    /                       /cms/*
                                |
                                v
                          PostgreSQL

```

The frontend and CMS share a single hostname.

Strapi is deployed entirely behind the `/cms` URL prefix.

---

# 6. Caddy Reverse Proxy

Caddy is responsible for:

- HTTPS termination
- Automatic TLS certificate management
- Reverse proxy routing
- Environment separation
- Exposing frontend and CMS through a single hostname
---
## Routing Rules

Frontend:
```text
/
```

CMS:

```text
/cms/*
```

The frontend NGINX container serves static frontend assets only.
The frontend does not proxy API requests.
All API and upload traffic is routed through Caddy.

---

## TLS Certificates

Certificates are automatically issued and renewed using ACME.
Persistent storage uses:
```text
myapp_caddy_data
myapp_caddy_config
```

These volumes must be retained across updates and redeployments.
Do not remove them unless certificate regeneration is intentionally required.

---

# 7. Strapi Configuration

## Deployment Path

Strapi is deployed under:

```text
/cms
```

Public resources:

```text
/cms/admin
/cms/api/*
/cms/uploads/*
```

---

## Public URL

### Pre-production

```env
PUBLIC_URL=https://kiel-sealevel.westeurope.cloudapp.azure.com/cms
```

### Staging

```env
PUBLIC_URL=https://kiel-sealevel.westeurope.cloudapp.azure.com:8443/cms
```

Changing the public URL requires rebuilding the CMS container image.

---

## Administration Interface

The administration UI is exposed through:

```text
/cms/admin
```

The admin build configuration and runtime configuration must remain consistent with the deployment URL structure.

---

## Uploaded Media

Media files are accessible through:

```text
/cms/uploads/*
```

Examples:

```text
/cms/uploads/example-image.webp
/cms/uploads/document.pdf
```

Testing:

```text
/cms/uploads
```

alone is not meaningful because it is not a file resource and may legitimately return HTTP 404.

---

# 8. Frontend Integration

The frontend is a static SPA served by NGINX.

NGINX serves only:

```text
/
/assets/*
```

The frontend should access CMS resources through:

```text
/cms/api/*
```

and

```text
/cms/uploads/*
```

---

## Correct API Usage

Correct:

```text
/cms/api/public/scenes
```

```text
/cms/uploads/example-image.webp
```

Incorrect:

```text
/api/public/scenes
```

```text
/uploads/example-image.webp
```

---

# 9. Watchtower

Watchtower is used to monitor container images and automatically recreate containers when new images are published.
Recommended scopes:

```text
staging
preprod
```

Example labels:

```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=true"
  - "com.centurylinklabs.watchtower.scope=staging"
```

Watchtower monitors configured image tags and redeploys updated images automatically.

---

# 10. CI/CD Flow

## Staging

Triggered by push to `main`.

```text
GitHub Push
      |
      v
GitHub Actions
      |
      v
Build Image
      |
      v
Push to ACR
      |
      v
Tag: staging
      |
      v
Watchtower
      |
      v
Container Recreated

```

---

## Pre-production

Triggered by GitHub Release.

```text
GitHub Release
      |
      v
GitHub Actions
      |
      v
Build Image
      |
      v
Push to ACR
      |
      +----> pre-prod
      |
      +----> vX.Y.Z
      |
      v
Watchtower
      |
      v
Container Recreated

```

---

# 11. Persistent Storage

Persistent Docker volumes preserve data across container recreation.

Expected volumes:

| Volume | Purpose |

|----------|---------|

| myapp_caddy_data | TLS certificates |

| myapp_caddy_config | Caddy state |

| staging_postgres_data | Staging database |

| preprod_postgres_data | Pre-production database |

| staging_cms_uploads | Staging uploads |

| preprod_cms_uploads | Pre-production uploads |

Staging and pre-production must never share storage.

Regular backup procedures should be implemented separately.

---

# 12. Deployment Verification

Verify Frontend:

```bash
curl -I https://kiel-sealevel.westeurope.cloudapp.azure.com
```

Verify CMS Admin:

```bash
curl -I \
https://kiel-sealevel.westeurope.cloudapp.azure.com/cms/admin

```

Expected:

```text
HTTP 200
X-Powered-By: Strapi
```

Verify API:

```bash
curl \
https://kiel-sealevel.westeurope.cloudapp.azure.com/cms/api/public/scenes
```

Expected:

```json
{
  "data": []
}
```

(or application data)

Verify Uploaded Assets:

```bash
curl -I \
https://kiel-sealevel.westeurope.cloudapp.azure.com/cms/uploads/<file>
```

Expected:

```text
HTTP 200
```

---

# 13. Operations

Pull latest images:

```bash

docker compose -f frontend/compose.yaml pull
docker compose -f cms/compose.yaml pull
```

Restart Frontend:

```bash
docker compose -f frontend/compose.yaml down
docker compose -f frontend/compose.yaml up -d
```

Restart CMS:

```bash
docker compose -f cms/compose.yaml down
docker compose -f cms/compose.yaml up -d
```

Check the Caddyfile correctness then reload the configuration:

```bash
docker exec caddy-rp \
  caddy validate \
  --config /etc/caddy/Caddyfile
  
docker exec caddy-rp \
  caddy reload \
  --config /etc/caddy/Caddyfile
```

---

# 14. Useful Commands

Running containers:

```bash
docker ps
```

All containers:

```bash
docker ps -a
```

Docker networks:

```bash
docker network ls
```

Docker volumes:

```bash
docker volume ls
```

Follow CMS logs:

```bash
docker logs -f cms-preprod
```

Follow Frontend logs:

```bash
docker logs -f frontend-preprod
```

Follow Caddy logs:

```bash
docker logs -f caddy-rp
```

Check Docker service:

```bash
sudo systemctl status docker
```

Start Docker:

```bash
sudo systemctl start docker
```


---

# 15. Troubleshooting

## Verify Caddy Can Reach Strapi

Enter the Caddy container:

```bash
docker exec -it caddy-rp sh
```

Test connectivity:

```bash
wget -qO- http://cms-preprod:1337/admin
```

Expected:

```text
HTML response from Strapi
```

---

## Verify Network Attachments

Inspect network:

```bash
docker network inspect kiel-sealevel_preprod_proxy
```

Confirm:

```text
caddy-rp
cms-preprod
frontend-preprod
```

are attached.

---

## Verify Strapi Is Listening

```bash
docker exec cms-preprod \
  sh -c "netstat -tlnp 2>/dev/null || ss -tlnp"
```

Expected:

```text
0.0.0.0:1337 LISTEN
```

---

## Verify Runtime Configuration

```bash
docker exec cms-preprod env | sort
```

Verify:

```text
HOST=0.0.0.0
PORT=1337
PUBLIC_URL=https://kiel-sealevel.westeurope.cloudapp.azure.com/cms
```

---

## Verify Running Image

CMS:

```bash
docker inspect cms-preprod \
  --format='{{.Config.Image}}'
```

Frontend:

```bash
docker inspect frontend-preprod \
  --format='{{.Config.Image}}'
```

Useful when validating deployments after image updates.

---

## Common Symptoms

| Symptom | Likely Cause |

|----------|-------------|

| HTTP 502 from Caddy | CMS container unavailable or incorrect network attachment |

| Frontend HTML returned from API endpoint | Request routed to frontend instead of Strapi |

| Admin loads but API returns 404 | Incorrect API route or frontend configuration |

| Upload URLs return 404 | Invalid file path or missing uploaded asset |

| Admin assets fail to load | Incorrect `PUBLIC_URL` or admin base URL |

| TLS certificate problems | Missing or deleted Caddy volumes |

| Old application version visible | Updated image not pulled or container not recreated |

Always verify routing, image versions, and network attachments before modifying volume or TLS configuration.
