# OS2display v3 Hosting and Deployment

This is a deployment tool designed for hosting the OS2display v3 application using Docker. It provides a Docker-based setup, pre-configured files, and task automation to simplify the deployment and management of the application.

## Prerequisites

Before you begin, ensure you have the following installed on your system:

1. **Docker**: Docker Engine (version 20.10 or later).
2. **Docker Compose**: Docker Compose v2 (integrated with the `docker compose` command).
3. **Task**: The Taskfile CLI tool. Installation instructions are available at [taskfile.dev](https://taskfile.dev/#/installation).

Make sure your user has the necessary permissions to run Docker commands (e.g., being part of the `docker` group).

### Check Prerequisites

```bash
docker --version
docker compose version
task --version
```

## Create the Deploy User

Files inside the `os2display-api-service` container are owned by a user with UID 1042 and GID 1042. To prevent permission issues with bind mounts (the `media` and `jwt` volumes), install the application using a user with the same UID and GID.

```bash
# Create group and user with UID/GID 1042
sudo groupadd -g 1042 deploy
sudo useradd -u 1042 -g 1042 -m -s /bin/bash deploy
sudo passwd deploy

# Add the user to the docker group
sudo usermod -aG docker deploy
```

## HTTPS Requirement

This project can only run in secure mode using HTTPS (port 443). You must provide a valid domain name and an SSL certificate.

1. Use a fully qualified domain name (FQDN) that resolves to your server's IP address.
2. Place the certificate file (`docker.crt`) and private key file (`docker.key`) in the `traefik/ssl` directory.
3. Set the domain name in `.env` via the `COMPOSE_SERVER_DOMAIN` variable.

## Configuration

Before running `task install`, generate the configuration file using one of these methods:

**Option A** — Interactive prompt (recommended):

```bash
task _env:build
```

This reads `.env.example`, prompts for each placeholder value, and writes `.env`.

**Option B** — Manual copy and edit:

```bash
cp .env.example .env
```

Edit `.env` with your local settings. The key variables are described below.

### Domain, Name and Version

| Variable | Description | Default |
|----------|-------------|---------|
| `COMPOSE_PROJECT_NAME` | Docker Compose project name (used for container prefixes) | `os2display` |
| `COMPOSE_SERVER_DOMAIN` | Domain name where the server will be accessible | `os2display.local.itkdev.dk` |
| `COMPOSE_IMAGE_VERSION` | Version of the os2display Docker images (applies to all services) | `latest` |

### Infrastructure Options

Which infrastructure services to include is controlled by the `COMPOSE_FILES` variable in `.env`. It is a comma-separated list of Docker Compose files to load.

| Value | Purpose |
|-------|---------|
| `docker-compose.yml` | **Required.** Core services (os2display API, nginx, redis) |
| `docker-compose.mariadb.yml` | Built-in MariaDB database. Omit if using an external database |
| `docker-compose.traefik.yml` | Built-in Traefik reverse proxy. Omit if using an external proxy |

Default: `docker-compose.yml,docker-compose.mariadb.yml`

### Database

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_DATABASE_URL` | Doctrine database connection URL | `mysql://db:db@mariadb:3306/db?serverVersion=mariadb-10.5.13` |
| `MARIADB_USER` | MariaDB user (only when using built-in MariaDB) | `db` |
| `MARIADB_PASSWORD` | MariaDB password (only when using built-in MariaDB) | `db` |
| `MARIADB_ROOT_PASSWORD` | MariaDB root password (only when using built-in MariaDB) | `dbrootpassword` |
| `MARIADB_DATABASE` | MariaDB database name (only when using built-in MariaDB) | `db` |

### Secrets

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_SECRET` | Symfony application secret | `CHANGE_ME` |
| `APP_JWT_PASSPHRASE` | JWT key pair passphrase | `CHANGE_ME` |
| `APP_ADMIN_LOGIN_METHODS` | JSON array configuring admin login methods (required) | `[{"type":"username-password","enabled":true,...}]` |

**NOTE:** Change both `APP_SECRET` and `APP_JWT_PASSPHRASE` to secure values before running in production.

### OIDC (OpenID Connect)

**Internal provider**:

| Variable | Description |
|----------|-------------|
| `APP_INTERNAL_OIDC_METADATA_URL` | OIDC metadata URL provided by the IdP |
| `APP_INTERNAL_OIDC_CLIENT_ID` | OIDC client ID |
| `APP_INTERNAL_OIDC_CLIENT_SECRET` | OIDC client secret |
| `APP_INTERNAL_OIDC_REDIRECT_URI` | OIDC redirect URI |

**External provider** (screen/device login):

| Variable | Description |
|----------|-------------|
| `APP_EXTERNAL_OIDC_METADATA_URL` | OIDC metadata URL provided by the IdP |
| `APP_EXTERNAL_OIDC_CLIENT_ID` | OIDC client ID |
| `APP_EXTERNAL_OIDC_CLIENT_SECRET` | OIDC client secret |
| `APP_EXTERNAL_OIDC_REDIRECT_URI` | OIDC redirect URI |

## Installation

1. Generate or edit `.env` with your chosen settings.
2. Place your SSL certificate files (`docker.crt` and `docker.key`) in the `traefik/ssl` directory.
3. Run the install task:

```bash
task install
```

The install process will:
- Create the external `frontend` Docker network (if it doesn't exist)
- Pull Docker images
- Start all containers
- Generate JWT key pair
- Run `app:update` (database migrations and other setup tasks)
- Prompt you to create a tenant and an admin user
- Clear the Symfony cache

After installation, the application is available at:
- **Admin:** `https://<COMPOSE_SERVER_DOMAIN>/admin`
- **Screen client:** `https://<COMPOSE_SERVER_DOMAIN>/client`

## Available Tasks

For a full list of tasks, run:

```bash
task --list
```

| Task | Description |
|------|-------------|
| `task install` | Install the project (pull images, start containers, generate JWT keys, add tenant/user) |
| `task purge` | Remove all containers. Use `-- --volumes` to also delete volumes, `-- --network` to remove the frontend network |
| `task db:backup` | Perform a database dump (only when using the built-in MariaDB). Saves to the `db_backups/` directory |
| `task compose -- <args>` | Run `docker compose` with the correct `-f` flags derived from `COMPOSE_FILES` in `.env` |
| `task console -- <cmd>` | Run a Symfony console command inside the os2display container |
| `task open:admin` | Open the admin interface in the default browser |
| `task open:client` | Open the client interface in the default browser |

### Common compose commands via task

```bash
task compose -- up --detach       # Start all containers
task compose -- down              # Stop and remove containers
task compose -- logs -f           # Follow container logs
task compose -- ps                # List running containers
```

### Common console commands via task

```bash
task console -- cache:clear       # Clear Symfony cache
task console -- app:tenant:add    # Add a new tenant
task console -- app:user:add      # Add a new user
```

## Updating an Existing Installation

To update to newer image versions:

1. Update `COMPOSE_IMAGE_VERSION` in `.env`.
2. Pull and recreate containers:

```bash
task compose -- pull
task compose -- up --detach --remove-orphans
```

## License

This project is licensed under the Mozilla Public License Version 2.0. See [LICENSE](LICENSE) for details.
