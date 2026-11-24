
# Docker Deployment Infrastructure

## Overview

This repository contains the infrastructure configuration for hosting the application on a single Azure VM using Docker Compose.

Two independent environments are maintained:

- **Staging:** Automatically updated when changes are merged into `main`.
- **Pre-production:** Automatically updated when a semantic version tag (`vX.Y.Z`) is created.

Both environments run simultaneously with independent containers, networks, PostgreSQL databases, persistent storage, and application configuration.

## Architecture

| Component | Responsibility |
|---|---|
| Azure VM | Hosts the Docker infrastructure |
| Docker Compose | Container orchestration |
| GitHub Actions | Builds and publishes application images |
| GHCR | Docker image registry |
| Caddy | Reverse proxy and automatic HTTPS |
| Watchtower | Detects and deploys updated images |
| PostgreSQL | Persistent application database |

### Application components

Each environment consists of:

- **Frontend:** Static application served through Nginx.
- **CMS:** Strapi application running on Node.js.
- **Database:** Independent PostgreSQL instance.

### Routing

Caddy handles HTTPS and routes incoming requests to the appropriate containers.

| Environment | Application | URL |
|---|---|---|
| Staging | Frontend | `https://staging.example.com` |
| Staging | Strapi API | `https://api-staging.example.com` |
| Pre-production | Frontend | `https://preprod.example.com` |
| Pre-production | Strapi API | `https://api-preprod.example.com` |

Only Caddy exposes ports 80 and 443 publicly. Application containers communicate through private Docker networks. PostgreSQL is never publicly exposed.

## Repository structure

The infrastructure and applications are maintained in separate GitHub repositories.

### Infrastructure repository

    myapp-deployment/
    ├── compose.yaml
    ├── .env.example
    ├── caddy/
    │   ├── compose.yaml
    │   └── Caddyfile
    ├── frontend/
    │   └── compose.yaml
    ├── backend/
    │   └── compose.yaml
    └── watchtower/
        └── compose.yaml

### Frontend repository

    myapp-frontend/
    ├── Dockerfile
    ├── nginx.conf
    └── .github/workflows/
        └── deploy.yml

### CMS repository

    myapp-cms/
    ├── Dockerfile
    ├── config/
    ├── src/
    ├── public/
    └── .github/workflows/
        └── deploy.yml

## CI/CD and image publishing

Both application repositories use GitHub Actions to build Docker images and publish them to GitHub Container Registry (GHCR).

The image tagging strategy uses moving environment tags and immutable release tags.

| Git event | Published image tags |
|---|---|
| Merge to `main` | `:staging`, `:sha-COMMIT` |
| Create `v1.2.3` | `:preprod`, `:v1.2.3` |

The moving tags are used by Watchtower to detect new images. Semantic version tags provide release traceability and support rollback.

### Deployment lifecycle

1. A change is merged into `main` or a semantic version tag is created.
2. GitHub Actions builds the corresponding application image.
3. The image is published to GHCR.
4. Watchtower checks the registry for newer images.
5. Watchtower pulls the updated image and recreates the corresponding container.
6. Existing database and persistent storage volumes remain intact.

Frontend and CMS deployments are independent. Updating one component does not require restarting the other.

No manual Docker Compose restart is required for application image updates.

## Watchtower

Two Watchtower instances are configured:

- `watchtower-staging` monitors staging containers.
- `watchtower-preprod` monitors pre-production containers.

Both use separate scopes to prevent cross-environment updates.

Application containers must have the appropriate labels.

Example for staging:

    labels:
      - "com.centurylinklabs.watchtower.enable=true"
      - "com.centurylinklabs.watchtower.scope=staging"

Pre-production containers use `preprod` as their scope.

The configured polling interval is 60 seconds.

**Maintenance warning:** The original `containrrr/watchtower` project was archived in December 2025. It no longer receives upstream maintenance or security fixes. Evaluate a maintained alternative before relying on it for long-term production infrastructure.

## Environment configuration and secrets

Environment-specific configuration is maintained on the Azure VM and must never be committed to Git.

The repository contains `.env.example` as a reference template.

The actual `.env` file contains:

- GHCR organization and image names.
- Image tags for each environment.
- PostgreSQL database names, users, and passwords.
- Strapi application secrets.
- Caddy ACME email address.
- Docker registry configuration path.

### Required Strapi secrets

Each environment must have independent values for:

| Secret | Purpose |
|---|---|
| `APP_KEYS` | Session signing |
| `API_TOKEN_SALT` | API token generation |
| `ADMIN_JWT_SECRET` | Admin authentication |
| `TRANSFER_TOKEN_SALT` | Transfer token security |
| `JWT_SECRET` | Users & Permissions authentication |
| `ENCRYPTION_KEY` | Encryption of stored configuration |

Generate secrets using:

    openssl rand -hex 32

`APP_KEYS` requires multiple comma-separated values.

Staging and pre-production must never share database passwords or application secrets.

### GitHub Actions secrets

Both application repositories require the following GitHub Actions secrets:

| Secret | Description |
|---|---|
| `AZURE_VM_HOST` | Azure VM public IP or hostname |
| `AZURE_VM_USER` | SSH deployment user |
| `AZURE_VM_SSH_KEY` | Private SSH key |
| `AZURE_VM_KNOWN_HOSTS` | Verified SSH host key |

GHCR publishing uses the GitHub-provided `GITHUB_TOKEN` with `packages: write` permission.

The Azure VM requires a separate GitHub credential with `read:packages` permission to pull private images.

## Initial setup

### Requirements

- Ubuntu Azure VM.
- Docker Engine and Docker Compose plugin.
- DNS records pointing to the VM.
- Azure firewall allowing inbound TCP ports 80 and 443.
- SSH access for administration.
- GHCR authentication configured on the VM.

### Configure the deployment environment

    sudo mkdir -p /opt/myapp
    sudo chown "$USER":"$USER" /opt/myapp

    cd /opt/myapp

    cp .env.example .env
    chmod 600 .env

Edit `.env` and configure all required values.

### Authenticate with GHCR

    docker login ghcr.io

Use a dedicated GitHub credential with `read:packages` permission.

### Validate the Compose configuration

    docker compose \
      -f compose.yaml \
      -f caddy/compose.yaml \
      -f frontend/compose.yaml \
      -f backend/compose.yaml \
      -f watchtower/compose.yaml \
      config

### Start the infrastructure

    docker compose \
      -f compose.yaml \
      -f caddy/compose.yaml \
      -f frontend/compose.yaml \
      -f backend/compose.yaml \
      -f watchtower/compose.yaml \
      up -d

This starts Caddy, both application environments, PostgreSQL instances, and both Watchtower instances.

## Monitoring

### Check running containers

    docker ps

### Inspect application logs

    docker logs -f cms-staging
    docker logs -f cms-preprod
    docker logs -f frontend-staging
    docker logs -f frontend-preprod

### Inspect Watchtower

    docker logs -f watchtower-staging
    docker logs -f watchtower-preprod

### Check application health

    docker inspect cms-staging \
      --format '{{.State.Health.Status}}'

    docker inspect frontend-staging \
      --format '{{.State.Health.Status}}'

## Persistence and isolation

Each environment has independent persistent storage:

- PostgreSQL data.
- Strapi uploaded media.
- Application configuration and secrets.

Updating or recreating application containers does not remove their named Docker volumes.

Database backups and uploaded media backups must be managed separately.

Database migrations should be handled deliberately and should remain compatible with the currently deployed application version.

## Rollback

Immutable semantic version tags are retained in GHCR.

If a pre-production release needs to be reverted, change the corresponding image reference in the deployment configuration to the previous version and recreate the affected container.

For example:

    myapp-cms:preprod
            |
            v
    myapp-cms:v1.2.2

Watchtower follows the image reference configured for a container. Rollback therefore requires changing that reference and recreating the container.

Watchtower does not automatically roll back failed deployments.

## Security considerations

- Never commit `.env` or application secrets.
- Never expose PostgreSQL or application ports publicly.
- Restrict SSH access to trusted IP addresses.
- Use separate credentials and secrets for each environment.
- Keep Docker and the host operating system updated.
- Protect the Docker socket: access to it effectively grants root-level control over the host.
- Back up databases and uploaded media regularly.
- Pin GitHub Actions to reviewed commit SHAs.
- Protect semantic version tags against unauthorized creation or modification.

## References

- [Docker Compose documentation](https://docs.docker.com/compose/)
- [Docker Compose networking](https://docs.docker.com/compose/how-tos/networking/)
- [Caddy documentation](https://caddyserver.com/docs/)
- [Strapi documentation](https://docs.strapi.io/)
- [Strapi Docker installation](https://docs.strapi.io/cms/installation/docker)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Watchtower documentation](https://containrrr.dev/watchtower/)
- [PostgreSQL Docker image](https://hub.docker.com/_/postgres)

---

**Objective:** Maintain two isolated application environments on a single Azure VM, with automated image publishing, HTTPS routing, persistent storage, and automatic container updates without introducing the operational overhead of Kubernetes.

