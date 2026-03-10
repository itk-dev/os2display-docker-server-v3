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

This project can only run in secure mode using HTTPS (port 443). 

You can configure an external reverse proxy til handle SSL termination or you can go with the default
installation options, that sets up Traefik as an integrated reverse proxy.

If you go with internal Traefik service, you must provide the SSL certificate file (`docker.crt`) and private file (`docker.key`) in the `traefik/ssl` directory.

## Configuration
You need to create a `.env` with your local settings. The installation package provides an example configuration file (`.env.example`) for you to copy.

If you run `task install` with no `.env` present, the installer will interactively prompt you for the most important settings and create `.env` for you.

See [.env.example](.env.example) for descriptions of all configurable variables.


## Installation

You install by executing the install task:

```bash
task install
```

With default installation options the install process will:
- Create the external `frontend` Docker network (if it doesn't exist)
- Create and populate `.env` if it does not exists
- Check that the installer version matches the OS2Display version
- Check that the SSL certificate and key is present
- Pull Docker images (OS2display, MariaDB, Traefik)
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
task
```

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
