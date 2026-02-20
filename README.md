# OS2display v2 Hosting and Deployment

This is a deployment tool designed for hosting the OS2display v2 application using Docker. It provides a Docker-based setup, pre-configured files, and task automation to simplify the deployment and management of the application.

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
3. Set the domain name in `.env.docker.local` via the `COMPOSE_SERVER_DOMAIN` variable.

## Configuration

Before running `task install`, copy and edit the configuration file:

```bash
cp .env.docker.example .env.docker.local
```

Edit `.env.docker.local` with your local settings. The key variables are described below.

### Domain and Versions

| Variable | Description | Default |
|----------|-------------|---------|
| `COMPOSE_SERVER_DOMAIN` | Domain name where the server will be accessible | `demo.os2display.dk` |
| `COMPOSE_VERSION_API` | Version of `itkdev/os2display-api-service` | `2.6.0` |
| `COMPOSE_VERSION_ADMIN` | Version of `itkdev/os2display-admin-client` | `2.6.0` |
| `COMPOSE_VERSION_CLIENT` | Version of `itkdev/os2display-client` | `2.3.0` |

### Infrastructure Options

| Variable | Description | Default |
|----------|-------------|---------|
| `INTERNAL_DATABASE` | Set to `true` to use the built-in MariaDB, `false` for an external database | `true` |
| `INTERNAL_PROXY` | Set to `true` to use the built-in Traefik proxy, `false` for an external proxy | `true` |

### Database

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_DATABASE_URL` | Doctrine database connection URL | `mysql://db:db@mariadb:3306/db?serverVersion=mariadb-10.11.11` |
| `MARIADB_USER` | MariaDB user (only when `INTERNAL_DATABASE=true`) | `db` |
| `MARIADB_PASSWORD` | MariaDB password (only when `INTERNAL_DATABASE=true`) | `db` |
| `MARIADB_ROOT_PASSWORD` | MariaDB root password (only when `INTERNAL_DATABASE=true`) | `dbrootpassword` |
| `MARIADB_DATABASE` | MariaDB database name (only when `INTERNAL_DATABASE=true`) | `db` |

### Secrets

| Variable | Description | Default |
|----------|-------------|---------|
| `APP_SECRET` | Symfony application secret | `pleasuchangethis` |
| `APP_JWT_PASSPHRASE` | JWT key pair passphrase | `pleasechangethistoo` |

**NOTE:** Change both `APP_SECRET` and `APP_JWT_PASSPHRASE` to secure values before running in production.

### Templates and Screen Layouts

| Variable | Description | Default |
|----------|-------------|---------|
| `TASK_VERSION_TEMPLATES` | Version/branch of [os2display/display-templates](https://github.com/os2display/display-templates/releases) | `2.6.0` |
| `TASK_TEMPLATES` | Comma-separated list of templates to load | See `.env.docker.example` |
| `TASK_SCREEN_LAYOUTS` | Comma-separated list of screen layouts to load | See `.env.docker.example` |

### OIDC (OpenID Connect)

| Variable | Description |
|----------|-------------|
| `INTERNAL_OIDC_METADATA_URL` | OIDC metadata URL provided by the IdP |
| `INTERNAL_OIDC_CLIENT_ID` | OIDC client ID |
| `INTERNAL_OIDC_CLIENT_SECRET` | OIDC client secret |
| `INTERNAL_OIDC_REDIRECT_URI` | OIDC redirect URI |

## Installation

1. Edit `.env.docker.local` with your domain name, secure passwords, and other settings.
2. Place your SSL certificate files (`docker.crt` and `docker.key`) in the `traefik/ssl` directory.
3. Run the install task:

```bash
task install
```

The install process will:
- Pull Docker images
- Start all containers
- Create the database schema
- Generate JWT key pair
- Prompt you to create a tenant and an admin user
- Load templates and screen layouts

After installation, the application is available at:
- **Admin:** `https://<COMPOSE_SERVER_DOMAIN>/admin`
- **Screen client:** `https://<COMPOSE_SERVER_DOMAIN>/client`

## Available Tasks

For a full list of tasks, run:

```bash
task --list
```

### Installation and Setup

| Task | Description |
|------|-------------|
| `task install` | Install the project (pull images, create DB, generate JWT keys, add tenant/user, load templates) |
| `task reinstall` | Reinstall from scratch. **WARNING:** Deletes the database |
| `task up` | Start the environment (recompiles configuration) |
| `task down` | Stop and remove all containers |
| `task stop` | Stop all containers without removing them |
| `task purge` | Remove all containers and volumes. **WARNING:** Deletes the database |

### Tenant and User Management

| Task | Description |
|------|-------------|
| `task tenant:add` | Add a new tenant group |
| `task user:add` | Add a new user (editor or admin) to a tenant |

### Templates and Screen Layouts

| Task | Description |
|------|-------------|
| `task template:load` | Load templates and screen layouts based on configuration in `.env.docker.local` |

### Maintenance

| Task | Description |
|------|-------------|
| `task logs` | Follow logs from the Docker containers |
| `task cache:clear` | Clear the application cache |
| `task db:backup` | Perform a database dump (only when using the built-in MariaDB). Saves to the `db_backups/` directory |

## Updating an Existing Installation

To update to newer image versions:

1. Update the version variables (`COMPOSE_VERSION_API`, `COMPOSE_VERSION_ADMIN`, `COMPOSE_VERSION_CLIENT`) in `.env.docker.local`.
2. Run the restart script:

```bash
./restart.sh
```

This pulls new images, recreates containers, and runs database migrations. If templates have changed, run `task template:load` afterwards.

## License

This project is licensed under the European Union Public License 1.2. See [LICENSE](LICENSE) for details.
