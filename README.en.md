# Self-Hosted Langfuse on a Local Network

English | [日本語](README.md)

A configuration for deploying [Langfuse](https://langfuse.com/) on a local area network (LAN) host using Docker Compose and directly sending traces from OpenTelemetry-compatible clients.

Traces are sent directly to Langfuse via OTLP/HTTP without using an OpenTelemetry Collector.

```mermaid
flowchart LR
    A["Copilot CLI/TUI"] --> D["OTLP/HTTP"]
    B["VS Code Chat"] --> D
    C["Other OTLP Clients"] --> D
    D --> E["Langfuse"]
```

## 1. Prerequisites

- Docker Engine
- Docker Compose v2
- OpenSSL

This README uses the following sample configuration:

```text
Host IP: 192.168.1.20
Port: 16300
URL: http://192.168.1.20:16300
```

> [!NOTE]
> Replace `192.168.1.20` with the actual LAN IP address of the host running Langfuse.

### About the Port Number

The Web server inside the Langfuse container listens on default port `3000/tcp`.

In this README, port `16300` is used as the host exposed port.  
`16300` has no special meaning or security implications. You can change it to any unused port.

> [!NOTE]
> Port 3000 is often used by other development Web applications or tools running on the host. The port is modified here to avoid port conflicts.

```mermaid
flowchart LR
    LAN["LAN"]

    subgraph HOST["Langfuse Host<br/>(192.168.1.20)"]
        direction LR

        PORT["Host port<br/>192.168.1.20:16300"]

        subgraph DOCKER["Docker"]
            WEB["langfuse-web<br/>3000/tcp"]
        end

        PORT -->|"16300 → 3000"| WEB
    end

    LAN --> PORT
```

## 2. Initial Setup

### 2.1 Generate `.env`

Run this at the root of the repository:

```bash
umask 077

LANGFUSE_HOST_PORT=16300
LANGFUSE_HOST_IP=192.168.1.20

POSTGRES_PASSWORD="$(openssl rand -hex 32)"
CLICKHOUSE_PASSWORD="$(openssl rand -hex 32)"
REDIS_AUTH="$(openssl rand -hex 32)"
MINIO_ROOT_PASSWORD="$(openssl rand -hex 32)"

cat > .env <<EOF
# -----------------------------------------------------------------------------
# Network Settings
# -----------------------------------------------------------------------------

# Exposed host port in docker-compose.yml.
# Container listening port remains 3000.
LANGFUSE_HOST_PORT=${LANGFUSE_HOST_PORT}

# Public URL used for Langfuse authentication and redirects.
NEXTAUTH_URL=http://${LANGFUSE_HOST_IP}:${LANGFUSE_HOST_PORT}


# -----------------------------------------------------------------------------
# Langfuse Settings
# -----------------------------------------------------------------------------

NEXTAUTH_SECRET=$(openssl rand -hex 32)
SALT=$(openssl rand -hex 32)

# 256-bit encryption key specified as a 64-character hex string.
ENCRYPTION_KEY=$(openssl rand -hex 32)

# Disable Langfuse's anonymous telemetry.
# Does not affect receiving OpenTelemetry from external clients.
TELEMETRY_ENABLED=false

# Keep signup enabled until the initial admin user is created.
AUTH_DISABLE_SIGNUP=false


# -----------------------------------------------------------------------------
# PostgreSQL Settings
# -----------------------------------------------------------------------------

POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/postgres


# -----------------------------------------------------------------------------
# ClickHouse Settings
# -----------------------------------------------------------------------------

CLICKHOUSE_PASSWORD=${CLICKHOUSE_PASSWORD}


# -----------------------------------------------------------------------------
# Redis Settings
# -----------------------------------------------------------------------------

REDIS_AUTH=${REDIS_AUTH}


# -----------------------------------------------------------------------------
# MinIO / S3 Settings
# -----------------------------------------------------------------------------

MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}

LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}

LANGFUSE_S3_MEDIA_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_MEDIA_UPLOAD_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}

LANGFUSE_S3_BATCH_EXPORT_ACCESS_KEY_ID=minio
LANGFUSE_S3_BATCH_EXPORT_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}
EOF

chmod 600 .env
```

Before running, update the following variables to match your environment:

```bash
LANGFUSE_HOST_PORT=16300
LANGFUSE_HOST_IP=192.168.1.20
```

### 2.2 Modifications in `docker-compose.yml`

The [docker-compose.yml](./docker-compose.yml) in this repository includes the following modifications based on the [official docker-compose.yml](https://github.com/langfuse/langfuse/blob/main/docker-compose.yml).  
These changes are already applied, so no additional edits are necessary unless you have specific requirements.

#### Parametrize Host Port for Langfuse Web Only

```yaml
services:
  langfuse-web:
    ports:
      - "0.0.0.0:${LANGFUSE_HOST_PORT:-16300}:3000"
```

Exposing on `0.0.0.0` allows access from other hosts on the same LAN.

If you want to expose only on a specific interface within the LAN, restrict source access using the host firewall.

#### Keep Ports Unexposed for Non-Web Services

Inter-service communication in Langfuse uses the internal Docker Compose network, so there is no need to publish PostgreSQL or Redis to the host. The official documentation also recommends exposing only `langfuse-web` to the outside.

- langfuse-worker
- clickhouse
- minio
- redis
- postgres

> [!NOTE]
> Whether MinIO needs to be exposed depends on your Langfuse media upload configuration.
> When using media features, check the official Compose configuration and `LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT`.

### 2.3 Verify Compose Configuration

Verify that `.env` is loaded correctly:

```bash
docker compose config --environment
```

You can inspect the expanded Compose configuration with:

```bash
docker compose config
```

> [!CAUTION]
> `docker compose config` may display expanded passwords and secrets. Do not paste the output into logs or public issues.

### 2.4 Start Services

```bash
docker compose up -d
```

Check service status:

```bash
docker compose ps
```

Access the Langfuse Web UI at:

```text
http://192.168.1.20:16300
```

## 3. Initial Langfuse Configuration

### 3.1 Create Admin User and Project

After the initial startup, perform the following in the Langfuse Web UI:

1. Create the initial admin user
2. Create or select an Organization
3. Create a Project
4. Generate a Public Key and Secret Key for the Project

Use the project-level Public Key and Secret Key for sending OpenTelemetry traces.

### 3.2 Disable Signup

The initial startup setting is:

```env
AUTH_DISABLE_SIGNUP=false
```

Once the initial admin user and Project are created, update `.env`:

```env
AUTH_DISABLE_SIGNUP=true
```

Recreate the containers to apply the change:

```bash
docker compose up -d
```

This prevents new user signups while preserving login access for existing accounts.

## 4. Direct OpenTelemetry Ingestion

The Langfuse OTLP endpoint is:

```text
http://192.168.1.20:16300/api/public/otel
```

Langfuse accepts OTLP/HTTP in JSON or protobuf formats. gRPC is not used.

### 4.1 Shared Authentication Environment File

Create a shared authentication environment file configured with your Langfuse Public Key and Secret Key:

```bash
mkdir -p ~/.config/langfuse
chmod 700 ~/.config/langfuse

umask 077

cat > ~/.config/langfuse/copilot-otel.env <<'EOF'
export LANGFUSE_HOST_IP="192.168.1.20"
export LANGFUSE_HOST_PORT="16300"
export LANGFUSE_SECRET_KEY="sk-lf-..."
export LANGFUSE_PUBLIC_KEY="pk-lf-..."
export LANGFUSE_BASE_URL="http://${LANGFUSE_HOST_IP}:${LANGFUSE_HOST_PORT}"

export LANGFUSE_AUTH_STRING="$(
  printf '%s' "${LANGFUSE_PUBLIC_KEY}:${LANGFUSE_SECRET_KEY}" \
    | base64 \
    | tr -d '\n'
)"

export OTEL_EXPORTER_OTLP_ENDPOINT="${LANGFUSE_BASE_URL}/api/public/otel"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/json"
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic ${LANGFUSE_AUTH_STRING},x-langfuse-ingestion-version=4"
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT="false"
EOF

chmod 600 ~/.config/langfuse/copilot-otel.env
```

Replace `sk-lf-...` and `pk-lf-...` with your actual Project keys created in the Langfuse Web UI.

> [!CAUTION]
> Never commit your Secret Key to a Git repository.

> [!TIP]
> Specifying `x-langfuse-ingestion-version: 4` reduces ingestion delay for the Langfuse v4 data model and Observations API.
>
> For details, refer to the [Langfuse OpenTelemetry documentation](https://langfuse.com/integrations/native/opentelemetry).

Add the following code to `~/.zshrc`:

```bash
# Wrap GitHub Copilot CLI command with Langfuse environment variables
copilot() (
  source "$HOME/.config/langfuse/copilot-otel.env"

  export COPILOT_OTEL_ENABLED="true"
  export OTEL_RESOURCE_ATTRIBUTES="langfuse.trace.metadata.execution_origin=manual-tui"

  # Use 'command' to call actual binary and prevent recursion
  command copilot "$@"
)

# Wrap VS Code launch command with Langfuse environment variables
code() (
  source "$HOME/.config/langfuse/copilot-otel.env"

  export OTEL_RESOURCE_ATTRIBUTES="langfuse.trace.metadata.execution_origin=vscode-chat"

  # Call actual VS Code CLI rather than the function itself
  command code "$@"
)
```

After adding, open a new terminal or reload configuration:

```bash
source ~/.zshrc
```

## 5. Client Configuration

### 5.1 Copilot CLI/TUI

Launch Copilot CLI/TUI from the VS Code integrated terminal or shell:

```bash
copilot
```

The `~/.zshrc` wrapper automatically attaches the following attribute:

```text
langfuse.trace.metadata.execution_origin=manual-tui
```

### 5.2 VS Code Built-in Chat

Configure in VS Code's `settings.json`:

```json
{
  "github.copilot.chat.otel.enabled": true,
  "github.copilot.chat.otel.exporterType": "otlp-http",
  "github.copilot.chat.otel.otlpEndpoint": "http://192.168.1.20:16300/api/public/otel",
  "github.copilot.chat.otel.captureContent": false
}
```

> [!TIP]
> `github.copilot.chat.otel.captureContent` controls whether message bodies (prompts, responses, tool arguments) are transmitted.

> [!WARNING]
> Setting `github.copilot.chat.otel.captureContent` to `true` may persist sensitive data, making it viewable to authorized users on Langfuse. We recommend setting it to `false`, especially in shared environments.

Do not write authentication headers into the VS Code settings file; set them via environment variables in the process that launches VS Code:

```bash
code .
```

When wrapping the `code` command, the following setting is applied automatically at startup:

```text
langfuse.trace.metadata.execution_origin=vscode-chat
```

> [!WARNING]
> Completely quit VS Code before running `code .` in a terminal with the OpenTelemetry environment variables set.
>
> If launched directly from the Dock or if you attach a folder to an already-running VS Code instance, the parent process environment variables might not be inherited.

If you run VS Code Remote Tunnel as a service and access VS Code Chat from a browser on a different host, refer to [VS Code Remote Tunnel Setup](./docs/vscode-remote-tunnel.en.md).

## 6. Origin Tracking

| Path                  | `execution_origin` |
| --------------------- | ------------------ |
| Copilot CLI/TUI       | `manual-tui`       |
| VS Code Built-in Chat | `vscode-chat`      |

Using `langfuse.trace.metadata.*` allows filtering and querying as metadata within Langfuse.

## 7. LAN Deployment Considerations

This configuration is intended for local area network (LAN) usage.

Exposing via `0.0.0.0:${LANGFUSE_HOST_PORT}:3000` listens on all host network interfaces.

We recommend the following safety measures:

- Allow access only from within the LAN using host firewall rules.
- Do not configure port forwarding from the internet on your router.
- Do not share Public Keys or Secret Keys.
- Do not access directly from untrusted networks.
- If public internet access is required, configure TLS, VPN, reverse proxies, and access controls separately.

Changing the port number to `16300` does not inherently secure the service.

> [!WARNING]
> Because this setup uses plain HTTP, Basic auth credentials and OpenTelemetry data are not encrypted in transit.
>
> Restrict usage to trusted LAN environments for development and verification.
> When accessing across untrusted networks, use HTTPS, a VPN, or a TLS-terminating reverse proxy.

## 8. Operations

### 8.1 Status Check

```bash
docker compose ps
```

### 8.2 Log Inspection

```bash
docker compose logs -f
```

To view logs for a specific service only:

```bash
docker compose logs -f langfuse-web
```

### 8.3 Apply Configuration Changes

After modifying `.env` or `docker-compose.yml`, run:

```bash
docker compose up -d
```

### 8.4 Stop

To stop the services (data volumes are normally preserved):

```bash
docker compose down
```

> [!CAUTION]
> To delete all volumes and data, run the following. Note that all stored data will be lost:
>
> ```bash
> docker compose down -v
> ```

## 9. Security

- Do not commit `.env` to Git.
- Add `.env` to `.gitignore`.
- Do not paste Public Keys, Secret Keys, or Basic auth strings in logs or issues.
- Keep `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=false`.
- Do not send sensitive message content (prompts, responses, tool arguments).
- Do not share the output of `docker compose config`.
- Avoid leaving Secret Keys in shell history.
- For production use, re-evaluate exposed ports, auth methods, backups, and data retention policies.

## 10. References

- [Langfuse OpenTelemetry](https://langfuse.com/integrations/native/opentelemetry)
- [GitHub Copilot CLI OpenTelemetry](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference)
- [VS Code OpenTelemetry monitoring](https://code.visualstudio.com/docs/agents/guides/monitoring-agents)

## License and attribution

This repository contains a modified version of the official Langfuse Docker Compose configuration.

- Original project: [Langfuse](https://github.com/langfuse/langfuse)
- Original Compose configuration: [docker-compose.yml](https://github.com/langfuse/langfuse/blob/main/docker-compose.yml)
- Original license: MIT Expat License
- Copyright: Copyright (c) 2023-2026 ClickHouse, Inc.

The modifications in this repository are intended for self-hosted Langfuse deployment on a local network.

See [licenses/Langfuse-LICENSE](./licenses/Langfuse-LICENSE) for the full original license text.

This repository is an unofficial community configuration and is not affiliated with or endorsed by Langfuse or ClickHouse.
