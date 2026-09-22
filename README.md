# Self-Hosted Langfuse on a Local Network

[English](README.en.md) | 日本語

Docker Compose で LAN 内のホストに [Langfuse](https://langfuse.com/) を構築し、OpenTelemetry 対応クライアントから直接トレースを送信する構成です。

OpenTelemetry Collector は使用せず、Langfuse へ OTLP/HTTP で直接送信します。

```mermaid
flowchart LR
    A["Copilot CLI/TUI"] --> D["OTLP/HTTP"]
    B["VS Code Chat"] --> D
    C["その他のOTLP対応クライアント"] --> D
    D --> E["Langfuse"]
```

## 1. 前提条件

- Docker Engine
- Docker Compose v2
- OpenSSL

この README では、次の設定を例にします。

```text
Host IP: 192.168.1.20
Langfuse Web port: 16300
MinIO S3 API port: 16900
Langfuse URL: http://192.168.1.20:16300
```

> [!NOTE]
> `192.168.1.20` は、Langfuse を起動するホストの LAN 内 IP アドレスに置き換えてください。

### ポート番号について

コンテナ内部のサーバーは、標準構成では `3000/tcp` および `9090/tcp` で待ち受けます。

この README では、ホスト側の公開ポートに `16300` および `16900` を使用します。  
これらのポート番号自体に特別な意味やセキュリティ上の効果はありません。未使用のポートであれば、別の番号へ変更できます。

> [!NOTE]
> これは、ホスト上で開発中の別の Web アプリケーションやツール類が `3000` や `9090` を使用しているケースが多いため、ポート重複回避目的で変更しています。

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

## 2. 初期設定

### 2.1 `.env` を生成

リポジトリのルートで実行します。

```bash
umask 077

LANGFUSE_HOST_PORT=16300
MINIO_HOST_PORT=16900
LANGFUSE_HOST=192.168.1.20

POSTGRES_PASSWORD="$(openssl rand -hex 32)"
CLICKHOUSE_PASSWORD="$(openssl rand -hex 32)"
REDIS_AUTH="$(openssl rand -hex 32)"
MINIO_ROOT_PASSWORD="$(openssl rand -hex 32)"

cat > .env <<EOF
# -----------------------------------------------------------------------------
# ネットワーク設定
# -----------------------------------------------------------------------------

# docker-compose.ymlでホスト側に公開するポートです。
# Langfuseコンテナ内部の待受ポートは引き続き3000です。
LANGFUSE_HOST_PORT=${LANGFUSE_HOST_PORT}

# Batch ExportのダウンロードおよびMedia Uploadで使用する
# MinIO S3 APIのホスト側公開ポートです。
MINIO_HOST_PORT=${MINIO_HOST_PORT}

# Langfuseの認証およびリダイレクトに使用する公開URLです。
NEXTAUTH_URL=http://${LANGFUSE_HOST}:${LANGFUSE_HOST_PORT}


# -----------------------------------------------------------------------------
# Langfuse設定
# -----------------------------------------------------------------------------

NEXTAUTH_SECRET=$(openssl rand -hex 32)
SALT=$(openssl rand -hex 32)

# 256-bitの暗号化キーを64文字の16進数で指定します。
ENCRYPTION_KEY=$(openssl rand -hex 32)

# Langfuse自身の匿名テレメトリを無効化します。
# 外部クライアントからのOpenTelemetry受信には影響しません。
TELEMETRY_ENABLED=false

# 初回管理ユーザーを作成するまではサインアップを有効にします。
AUTH_DISABLE_SIGNUP=false


# -----------------------------------------------------------------------------
# PostgreSQL設定
# -----------------------------------------------------------------------------

POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/postgres


# -----------------------------------------------------------------------------
# ClickHouse設定
# -----------------------------------------------------------------------------

CLICKHOUSE_PASSWORD=${CLICKHOUSE_PASSWORD}


# -----------------------------------------------------------------------------
# Redis設定
# -----------------------------------------------------------------------------

REDIS_AUTH=${REDIS_AUTH}


# -----------------------------------------------------------------------------
# MinIO / S3設定
# -----------------------------------------------------------------------------

MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}

LANGFUSE_S3_EVENT_UPLOAD_ACCESS_KEY_ID=minio
LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}

LANGFUSE_S3_BATCH_EXPORT_ENABLED=true
LANGFUSE_S3_BATCH_EXPORT_ACCESS_KEY_ID=minio
LANGFUSE_S3_BATCH_EXPORT_SECRET_ACCESS_KEY=${MINIO_ROOT_PASSWORD}
LANGFUSE_S3_BATCH_EXPORT_EXTERNAL_ENDPOINT=http://${LANGFUSE_HOST}:${MINIO_HOST_PORT}
EOF

chmod 600 .env
```

実行前に、次の値を環境に合わせて変更してください。

```bash
LANGFUSE_HOST_PORT=16300
MINIO_HOST_PORT=16900
LANGFUSE_HOST=192.168.1.20
```

### 2.2 `docker-compose.yml` の修正

このリポジトリの [docker-compose.yml](./docker-compose.yml) は、[公式の docker-compose.yml](https://github.com/langfuse/langfuse/blob/main/docker-compose.yml) をベースに以下の修正を加えています。  
すでに修正済みなので、何らかの意図がない限りは追加の修正は不要です。

#### Langfuse Web と MinIO S3 API のホスト側ポートを環境変数化

```yaml
services:
  langfuse-web:
    ports:
      - "0.0.0.0:${LANGFUSE_HOST_PORT:-16300}:3000"

  minio:
    ports:
      - "0.0.0.0:${MINIO_HOST_PORT:-16900}:9000"
```

`langfuse-web` に加えて `minio` を `0.0.0.0` で公開することで、同一 LAN 上の別ホストから Langfuse Web UI と Langfuse での Batch Export のダウンロードを利用できます。

LAN 内の特定インターフェースだけで公開したい場合は、ホストのファイアウォールでアクセス元を制限してください。

#### Batch Export の外部 Endpoint を設定

```env
LANGFUSE_S3_BATCH_EXPORT_ENABLED=true
LANGFUSE_S3_BATCH_EXPORT_EXTERNAL_ENDPOINT=http://192.168.1.20:16900
```

Batch Export は、一時ファイルを MinIO に保存し、署名付き URL を使ってブラウザへダウンロードさせます。

そのため、External Endpoint には Docker 内部の `minio:9000` ではなく、利用するブラウザから到達できるホストのアドレスを設定します。別ホストから利用する場合、`localhost:16900` はその別ホスト自身を指すため使用できません。

Langfuse コンテナから MinIO への内部通信は、引き続き Docker Compose ネットワーク上の `http://minio:9000` を使用します。

詳細は [Langfuse の Blob Storage 公式ドキュメント](https://langfuse.com/self-hosting/deployment/infrastructure/blobstorage) を参照してください。

#### 内部サービスと MinIO Console のポートを非公開化

Langfuse 自身のサービス間通信は Docker Compose の内部ネットワークを使うので、PostgreSQL や Redis などをホストへ publish する必要はありません。

- langfuse-worker
- clickhouse
- redis
- postgres
- MinIO Console (`9001/tcp`)

> [!NOTE]
> MinIO のうち、クライアントからのアクセスが必要な S3 API (`9000/tcp`) だけをホストへ公開します。管理用の MinIO Console (`9001/tcp`) は公開しません。

### 2.3 Compose 設定を確認

`.env` が正しく読み込まれていることを確認します。

```bash
docker compose config --environment
```

展開後の Compose 設定は次で確認できます。

```bash
docker compose config
```

> [!CAUTION]
> `docker compose config` には展開後のパスワードやシークレットが表示される場合があります。出力をログや Issue へ貼り付けないでください。

### 2.4 起動

```bash
docker compose up -d
```

起動状態を確認します。

```bash
docker compose ps
```

以下の URL で Langfuse Web UI へアクセスします。

```text
http://192.168.1.20:16300
```

## 3. 初回 Langfuse 設定

### 3.1 管理ユーザーと Project を作成

初回起動後、Langfuse Web UI で次を実施します。

1. 最初の管理ユーザーを作成
2. Organization を作成または選択
3. Project を作成
4. Project の Public Key と Secret Key を作成

OpenTelemetry 送信には、Project 単位の Public Key と Secret Key を使用します。

### 3.2 サインアップを無効化

初回起動時は次の設定です。

```env
AUTH_DISABLE_SIGNUP=false
```

最初の管理ユーザーと Project の作成が完了したら、`.env` を変更します。

```env
AUTH_DISABLE_SIGNUP=true
```

変更後、コンテナを再作成します。

```bash
docker compose up -d
```

既存ユーザーのログインは維持したまま、新規ユーザーのサインアップを禁止できます。

## 4. OpenTelemetry 直接送信

Langfuse の OTLP endpoint は次です。

```text
http://192.168.1.20:16300/api/public/otel
```

Langfuse は OTLP/HTTP の JSON または protobuf を受け付けます。gRPC は使用しません。

### 4.1 共通認証環境ファイル

Langfuse の Public Key と Secret Key を設定した共通認証環境ファイルを作成します。

```bash
mkdir -p ~/.config/langfuse
chmod 700 ~/.config/langfuse

umask 077

cat > ~/.config/langfuse/copilot-otel.env <<'EOF'
export LANGFUSE_HOST="192.168.1.20"
export LANGFUSE_HOST_PORT="16300"
export LANGFUSE_SECRET_KEY="sk-lf-..."
export LANGFUSE_PUBLIC_KEY="pk-lf-..."
export LANGFUSE_BASE_URL="http://${LANGFUSE_HOST}:${LANGFUSE_HOST_PORT}"

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

`sk-lf-...` と `pk-lf-...` は、Langfuse Web UI で作成した実際の Project Key へ置き換えてください。

> [!CAUTION]
> Secret Key を Git リポジトリへ保存しないでください。

> [!TIP]
> `x-langfuse-ingestion-version: 4` を指定すると、Langfuse v4 のデータモデルや Observations API への反映遅延を抑えられます。
>
> 詳細は [Langfuse OpenTelemetry 公式ドキュメント](https://langfuse.com/integrations/native/opentelemetry) を参照してください。

`~/.zshrc` に次のコードを追加します。

```bash
# GitHub Copilotの起動コマンドをLangfuse用の環境変数込みで上書き
copilot() (
  source "$HOME/.config/langfuse/copilot-otel.env"

  export COPILOT_OTEL_ENABLED="true"
  export OTEL_RESOURCE_ATTRIBUTES="langfuse.trace.metadata.execution_origin=manual-tui"

  # commandで本物の実行ファイルを呼び、関数の再帰呼び出しを防ぐ
  command copilot "$@"
)

# VS Codeの起動コマンドをLangfuse用の環境変数込みで上書き
code() (
  source "$HOME/.config/langfuse/copilot-otel.env"

  export OTEL_RESOURCE_ATTRIBUTES="langfuse.trace.metadata.execution_origin=vscode-chat"

  # code関数ではなく実際のVS Code CLIを呼ぶ
  command code "$@"
)
```

設定後、新しいターミナルを開くか、次を実行して設定を再読み込みします。

```bash
source ~/.zshrc
```

## 5. クライアント設定

### 5.1 Copilot CLI/TUI

VS Code の統合ターミナルなどから Copilot CLI/TUI を起動します。

```bash
copilot
```

`~/.zshrc` のラッパーにより、次の属性が自動的に付与されます。

```text
langfuse.trace.metadata.execution_origin=manual-tui
```

### 5.2 VS Code 組み込みチャット

VS Code の `settings.json` に設定します。

```json
{
  "github.copilot.chat.otel.enabled": true,
  "github.copilot.chat.otel.exporterType": "otlp-http",
  "github.copilot.chat.otel.otlpEndpoint": "http://192.168.1.20:16300/api/public/otel",
  "github.copilot.chat.otel.captureContent": false
}
```

> [!TIP]
> `github.copilot.chat.otel.captureContent` は、プロンプト、応答、ツール引数などの本文を送信するかどうかの設定です。

> [!WARNING]
> `github.copilot.chat.otel.captureContent` を `true` にすると、センシティブな情報が永続化されてしまい Langfuse 上で権限のあるユーザーに覗き見られてしまう可能性があります。特に共有環境などでは `false` を設定することを推奨します。

認証 header は VS Code の設定ファイルへ書かず、VS Code を起動するプロセスの環境変数へ設定します。

```bash
code .
```

`code` コマンドをラップしている場合は、起動時に次の設定が自動的に適用されます。

```text
langfuse.trace.metadata.execution_origin=vscode-chat
```

> [!WARNING]
> VS Code を完全に終了してから、OpenTelemetry 環境変数を設定したターミナルで `code .` を実行してください。
>
> Dock などから直接起動した場合や、すでに起動中の VS Code へフォルダを追加した場合は、起動元プロセスの環境変数が引き継がれないことがあります。

VS Code Remote Tunnel をサービスとして起動し、別ホストのブラウザから VS Code Chat を利用する場合は、[VS Code Remote Tunnel 経由の設定](./docs/vscode-remote-tunnel.md) を参照してください。

## 6. 経路識別

| 経路                     | `execution_origin` |
| ------------------------ | ------------------ |
| Copilot CLI/TUI          | `manual-tui`       |
| VS Code 組み込みチャット | `vscode-chat`      |

`langfuse.trace.metadata.*` を使用すると、Langfuse の metadata として検索できます。

## 7. LAN 公開時の注意

この構成は LAN 内での利用を想定しています。

`0.0.0.0:${LANGFUSE_HOST_PORT}:3000` および `0.0.0.0:${MINIO_HOST_PORT}:9000` で公開すると、ホストの全ネットワークインターフェースで待ち受けます。

そのため、次の対策を推奨します。

- ホストのファイアウォールで LAN 内からのアクセスだけを許可する
- ルーターでインターネット側からのポート転送を設定しない
- Public Key や Secret Key を共有しない
- MinIO のアクセスキーやシークレットキーを共有しない
- 信頼できないネットワークから直接アクセスしない
- インターネット公開が必要な場合は、TLS、VPN、リバースプロキシ、アクセス制御を別途構成する

ポート番号を `16300` へ変更しても、サービスが安全になるわけではありません。

> [!WARNING]
> この構成は HTTP を使用するため、Basic 認証情報および OpenTelemetry データは暗号化されません。
>
> 信頼できる LAN 内での開発・検証用途に限定してください。
> 異なるネットワークからアクセスする場合は、HTTPS、VPN、または TLS 終端するリバースプロキシを使用してください。

## 8. 運用

### 8.1 状態確認

```bash
docker compose ps
```

### 8.2 ログ確認

```bash
docker compose logs -f
```

特定サービスのログだけを確認する場合は、サービス名を指定します。

```bash
docker compose logs -f langfuse-web
```

### 8.3 設定変更の反映

`.env` や `docker-compose.yml` を変更した場合は、次を実行します。

```bash
docker compose up -d
```

### 8.4 停止

停止する場合は次を実行します。データボリュームは通常保持されます。

```bash
docker compose down
```

> [!CAUTION]
> すべてのデータを削除する場合は次を実行します。データが失われるため注意してください。
>
> ```bash
> docker compose down -v
> ```

## 9. セキュリティ

- `.env` は Git へ commit しない
- `.env` を `.gitignore` へ追加する
- Public Key、Secret Key、Basic 認証値をログや Issue へ貼り付けない
- `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=false` を維持する
- prompt、response、tool 引数などの本文を送信しない
- `docker compose config` の出力を共有しない
- Secret Key をシェル履歴へ残さない
- 本番用途では公開ポート、認証方式、バックアップ、データ保持期間を別途確認する

## 10. 参考資料

- [Langfuse OpenTelemetry](https://langfuse.com/integrations/native/opentelemetry)
- [Langfuse Blob Storage](https://langfuse.com/self-hosting/deployment/infrastructure/blobstorage)
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
