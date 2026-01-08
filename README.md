# Tapomix / Docker - Diun

Repository for basic [Diun](https://crazymax.dev/diun/) installation on VPS.

Diun is a CLI for checking image versions and reveice updates via notification.

## Installation

Clone this repository:

```bash
git clone https://github.com/tapomix/docker-diun.git diun
```

## Configuration

### 1. Environment file

Copy the `.env.dist` file to `.env` and customize it:

```bash
cp .env.dist .env
```

| Variable | Required | Description |
| -------- | -------- | ----------- |
| `CONTAINER_NAME` | Yes | Container name (e.g., `diun`) |
| `SERVICE_VERSION` | Yes | Diun image version (default: `latest`) |
| `SERVICE_NET` | Yes | Network name (default: `${CONTAINER_NAME}`) |
| `SOCKET_PROXY_NET` | Yes | Socket proxy network name (default: `socket-proxy-${CONTAINER_NAME}`) |
| `TZ` | Yes | Timezone (default: `Etc/UTC`) |
| `UID` | Yes | User ID to run the container (default: `1000`) |
| `GID` | Yes | Group ID to run the container (default: `1000`) |
| `CRON_SCHEDULE` | Yes | Cron schedule for scans (default: `@daily`) |
| `DOCKER_ENDPOINT` | Yes | Docker endpoint URL (default: `tcp://${SOCKET_PROXY_NET}:2375`) |
| `NOTIF_BODY` | No | Notification body template |
| `NOTIF_TITLE` | No | Notification title template |
| `NTFY_ENDPOINT` | No | Ntfy server URL (e.g., `https://ntfy.sh`) |
| `NTFY_TOPIC` | No | Ntfy topic name |
| ... | - | Add your custom variables |

### 2. Static configuration

The file `.docker/diun.yaml` is mounted in the container and defines the base behavior. You can customize the schedule and other options via environment variables in `compose.override.yaml`.

### 3. Providers

Diun uses providers to discover which images to monitor. By default, the `docker` provider is enabled and automatically discovers images from running containers.

#### Docker provider (default)

The Docker provider scans your containers to find images to monitor. This requires access to the Docker socket (see [Docker socket access](#5-docker-socket-access)).

No additional configuration is needed - it's enabled by default in `.docker/diun.yaml` with `watchStopped: true`, which also monitors stopped containers.

> **Note**: Other providers exist (Swarm, Kubernetes, Nomad, etc.) but are not covered here. See the [Diun providers documentation](https://crazymax.dev/diun/config/providers/).

#### File provider (optional)

You can additionally use the `file` provider to monitor specific images that are not running as containers (e.g., base images, images from other registries).

To enable the file provider, **add** the following to `compose.override.yaml`:

```yaml
// compose.override.yaml
services:
  diun:
    environment:
      DIUN_PROVIDERS_FILE_FILENAME: /etc/diun/images.yaml
    volumes:
      - .docker/images.yaml:/etc/diun/images.yaml:ro
```

Then **create** the file `.docker/images.yaml` with the images to monitor:

```yaml
// .docker/images.yaml
- name: nginx:latest
  notify_on:
    - update
  platform:
    os: linux
    arch: amd64
```

> **Note**: The file `.docker/images.yaml` is ignored by git, allowing you to customize it locally without committing changes.

For more information, see the [Diun providers documentation](https://crazymax.dev/diun/providers/file/).

### 4. Notifications

To receive update notifications, configure a notifier in `compose.override.yaml`.

#### Example with Ntfy

First, **set/uncomment** the required variables :

```bash
// .env
# Notification templates
NOTIF_TITLE="{{ .Entry.Image }} released"
NOTIF_BODY="Docker tag {{ .Entry.Image }} which you subscribed to through {{ .Entry.Provider }} provider has been released."

# Ntfy configuration
NTFY_ENDPOINT=https://ntfy.sh
NTFY_TOPIC=diun
```

Then **create** a secret file for authentication :

```bash
echo -n "your-ntfy-token" > .docker/.secrets/token-ntfy.txt
```

> **Important**: Use `echo -n` to avoid trailing newlines which will break authentication.

Finally, **configure** the notifier :

```yaml
// compose.override.yaml
services:
  diun:
    environment:
      - DIUN_NOTIF_NTFY_ENDPOINT=${NTFY_ENDPOINT}
      - DIUN_NOTIF_NTFY_TOKENFILE=/run/secrets/ntfy-token
      - DIUN_NOTIF_NTFY_TOPIC=${NTFY_TOPIC}
      - DIUN_NOTIF_NTFY_TEMPLATETITLE=${NOTIF_TITLE}
      - DIUN_NOTIF_NTFY_TEMPLATEBODY=${NOTIF_BODY}

    secrets:
      - ntfy-token

secrets:
  ntfy-token:
    file: .docker/.secrets/token-ntfy.txt
```

For other notifiers (Gotify, Discord, Slack, etc.), see the [Diun notifiers documentation](https://crazymax.dev/diun/config/notif/).

### 5. Docker socket access

By default, the container is configured to use a socket proxy via the `SOCKET_PROXY_NET` network. This is the recommended approach for security.

#### Option A: Socket proxy (default)

The container expects an external network named `${SOCKET_PROXY_NET}` (default: `socket-proxy-${CONTAINER_NAME}`).

Make sure your socket proxy container is running and connected to this network with the following permissions enabled:

| Permission | Required | Description |
| ---------- | -------- | ----------- |
| `CONTAINERS` | Yes | List and inspect containers |
| `IMAGES` | Yes | List and inspect images |

#### Option B: Direct socket access

If you prefer direct socket access, **add** the following to `compose.override.yaml`:

```yaml
// compose.override.yaml
services:
  diun:
    environment:
      DIUN_PROVIDERS_DOCKER_ENDPOINT: unix:///var/run/docker.sock
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - service-net

networks:
  socket-proxy-net:
    external: false
```

> **Note**: Direct socket access is less secure than using a socket proxy.

#### Option C: No socket access (file provider only)

If you don't need automatic container discovery, you can use only the `file` provider instead. In this mode, Diun contacts remote registries directly to check for updates and doesn't need access to the Docker socket at all.

**Add** the following to `compose.override.yaml`:

```yaml
// compose.override.yaml
services:
  diun:
    environment:
      DIUN_PROVIDERS_DOCKER_ENDPOINT: ""
    networks:
      - service-net

networks:
  socket-proxy-net:
    external: false
```

> **Note**: Make sure to configure the [file provider](#file-provider-optional) to specify which images to monitor.

### 6. Network configuration

#### External notifiers

If your notifier is accessible via the internet (e.g., `https://ntfy.sh`), the default network `service-net` configuration is sufficient.

#### Internal notifiers

If your notifier runs on the same Docker host (e.g., a self-hosted Ntfy or Gotify), you can use an internal Docker network to avoid routing through the internet.

**Update** the network configuration in `compose.override.yaml` to use the notifier's existing network:

```yaml
// compose.override.yaml
networks:
  service-net:
    external: true
```

Then **set** `SERVICE_NET` to match the notifier's network name :

```bash
// .env
SERVICE_NET=ntfy
```

> **Note**: The `NTFY_ENDPOINT` should use the container name as hostname (e.g., `http://ntfy:80`) instead of the public URL.

### 7. Example configuration

A complete example configuration is provided in `compose.override.dist.yaml`, combining:

- **File provider**: Monitor specific images defined in `.docker/images.yaml`
- **Ntfy notifier**: Receive notifications via Ntfy

To use it:

```bash
cp compose.override.dist.yaml compose.override.yaml
```

Then customize as needed (see sections above for details).

## Security

The container is configured with security best practices:

- **Non-root user**: Runs as `${UID}:${GID}` (default: `1000:1000`)
- **Read-only filesystem**: The container filesystem is mounted as read-only
- **Dropped capabilities**: All Linux capabilities are dropped (`cap_drop: ALL`)
- **No privilege escalation**: `no-new-privileges:true` prevents privilege escalation
- **Resource limits**: CPU, memory and process limits are enforced (`cpus: 0.25`, `mem_limit: 64m`, `pids_limit: 10`)

> The resources limitations are intentionally strict as Diun is a lightweight service. If needed, you can extend them via `compose.override.yaml

## Usage

### Start the container

```bash
docker compose up -d
```

### Test notifications

```bash
docker compose exec diun diun notif test
```

### Check logs

```bash
docker compose logs -f
```

### Stop the container

```bash
docker compose down --remove-orphans
```

## Resources

- [Diun documentation](https://crazymax.dev/diun/)
- [Diun notifiers](https://crazymax.dev/diun/config/notif/)
- [Diun providers](https://crazymax.dev/diun/config/providers/)
