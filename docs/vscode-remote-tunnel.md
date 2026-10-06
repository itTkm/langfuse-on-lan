# VS Code Remote Tunnel 経由の設定

[English](vscode-remote-tunnel.en.md) | 日本語

VS Code Remote Tunnel を利用して、別ホストのブラウザから Mac の VS Code に接続する場合の設定です。

```mermaid
flowchart LR
  subgraph MBA[開発作業用 MacBook Air]
    A[ブラウザ / vscode.dev]
  end
  subgraph MINI[開発用 Mac mini]
    C[VS Code Server]
  end
  subgraph LF[Langfuse サーバー]
    subgraph DOCKER[Docker]
      D[Langfuse]
    end
  end
  A -->|VS Code Remote Tunnel| C
  C -->|OTLP/HTTP| D
```

OpenTelemetry の送信元はブラウザではなく、Remote Tunnel 接続先で動作する VS Code Server です。
そのため、Langfuse のエンドポイントへ接続できるようにする設定は、接続先ホスト側で行います。

私の検証環境は VS Code Server と Langfuse サーバーが同じ Mac mini に同居しているので、実態は以下のような構成になっています。

```mermaid
flowchart LR
  subgraph MBA[開発作業用 MacBook Air]
    A[ブラウザ / vscode.dev]
  end
  subgraph MINI[開発用 Mac mini]
    C[VS Code Server]
    subgraph DOCKER[Docker]
      D[Langfuse]
    end
    C -->|OTLP/HTTP| D
  end
  A -->|VS Code Remote Tunnel| C
```

## 1. 前提

README の「OpenTelemetry 直接送信」にある共通認証環境ファイルを、接続先ホスト上に作成済みであることを前提にします。

```text
~/.config/langfuse/copilot-otel.env
```

Remote Tunnel 自体は、次のコマンドでサービスとしてインストール済みとします。

```bash
code tunnel service install --name macmini
```

VS Code Remote Tunnel の詳細は、[VS Code 公式ドキュメント](https://code.visualstudio.com/docs/remote/tunnels) を参照してください。

## 2. トンネルサービスへ OpenTelemetry 環境変数を設定

`code tunnel service` は macOS の `launchd` で起動します。`launchd` は、サービスをインストールしたターミナルの環境変数や `~/.zshrc` を読み込みません。

そのため、次の設定は `~/.zshrc` の `code()` ラッパーだけでは反映されません。

- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_PROTOCOL`
- `OTEL_EXPORTER_OTLP_HEADERS`
- `OTEL_RESOURCE_ATTRIBUTES`
- `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`
- `COPILOT_OTEL_ENDPOINT`
- `COPILOT_OTEL_ENABLED`
- `COPILOT_OTEL_CAPTURE_CONTENT`

サービスが使用する plist へ環境変数を設定します。以下は、既存の共通認証環境ファイルから値を読み込み、VS Code Tunnel のサービス plist へ反映する例です。

```bash
(
set -eu

ENV_FILE="$HOME/.config/langfuse/copilot-otel.env"

# VS Code が生成した正本を使用します。
PLIST="$HOME/com.visualstudio.code.tunnel.plist"
LAUNCH_AGENT="$HOME/Library/LaunchAgents/com.visualstudio.code.tunnel.plist"

if [[ ! -r "$ENV_FILE" ]]; then
  echo "環境ファイルが見つかりません: $ENV_FILE" >&2
  exit 1
fi

if [[ ! -r "$PLIST" ]]; then
  echo "Tunnelサービスのplistが見つかりません: $PLIST" >&2
  exit 1
fi

source "$ENV_FILE"
: "${LANGFUSE_BASE_URL:?}"
: "${OTEL_EXPORTER_OTLP_HEADERS:?}"
LANGFUSE_BASE_URL="${LANGFUSE_BASE_URL%/}"

OTEL_RESOURCE_ATTRIBUTES_VALUE="langfuse.trace.metadata.execution_origin=vscode-chat"
LABEL="$(/usr/libexec/PlistBuddy -c 'Print :Label' "$PLIST")"

# LaunchAgents の既存の通常ファイルを上書きしません。
mkdir -p "$(dirname "$LAUNCH_AGENT")"
if [[ -e "$LAUNCH_AGENT" && ! -L "$LAUNCH_AGENT" ]]; then
  echo "既存ファイルを確認・退避してください: $LAUNCH_AGENT" >&2
  exit 1
fi
chmod 600 "$PLIST"
DOMAIN="gui/$(id -u)"
if launchctl print "$DOMAIN/$LABEL" >/dev/null 2>&1; then
  launchctl bootout "$DOMAIN/$LABEL"
fi

if ! /usr/libexec/PlistBuddy -c 'Print :EnvironmentVariables' "$PLIST" >/dev/null 2>&1; then
  /usr/libexec/PlistBuddy -c 'Add :EnvironmentVariables dict' "$PLIST"
fi

set_plist_env() {
  local key="$1"
  local value="$2"
  plutil -remove "EnvironmentVariables.${key}" "$PLIST" 2>/dev/null || true
  plutil -insert "EnvironmentVariables.${key}" -string "$value" "$PLIST"
}

set_plist_env COPILOT_OTEL_ENDPOINT "${LANGFUSE_BASE_URL}/api/public/otel"
set_plist_env OTEL_EXPORTER_OTLP_ENDPOINT "${LANGFUSE_BASE_URL}/api/public/otel/v1/traces"
set_plist_env OTEL_EXPORTER_OTLP_PROTOCOL "http/json"
set_plist_env OTEL_EXPORTER_OTLP_HEADERS "$OTEL_EXPORTER_OTLP_HEADERS"
set_plist_env OTEL_RESOURCE_ATTRIBUTES "$OTEL_RESOURCE_ATTRIBUTES_VALUE"
set_plist_env OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT "false"
set_plist_env COPILOT_OTEL_ENABLED "true"
set_plist_env COPILOT_OTEL_CAPTURE_CONTENT "false"

chmod 600 "$PLIST"
plutil -lint "$PLIST"

if [[ ! -L "$LAUNCH_AGENT" ]] || [[ "$(readlink "$LAUNCH_AGENT")" != "$PLIST" ]]; then
  ln -sfn "$PLIST" "$LAUNCH_AGENT"
fi
launchctl bootstrap "$DOMAIN" "$PLIST"
launchctl kickstart "$DOMAIN/$LABEL"
)
```

この例は、macOS で VS Code が `$HOME/com.visualstudio.code.tunnel.plist` を生成する場合の設定です。このファイルを正本とし、`~/Library/LaunchAgents` には symlink を配置してログイン後に自動ロードさせます。リンク先の配置場所に通常ファイルがある場合は、内容を確認・退避してから実行してください。配布形態によって場所が異なる場合は、実際の plist を使用し、別のサービス定義を作成しないでください。

`code tunnel service install` で正本が再生成された場合や env ファイルを変更した場合は、環境変数設定を再実行してください。subshell により、読み込んだ認証情報はこの設定処理内に限定します。reload 時には Tunnel が一時切断されるため、接続し直してください。

| プロセス | 環境変数 | endpoint |
| --- | --- | --- |
| Extension Host | `COPILOT_OTEL_ENDPOINT` | `/api/public/otel`（`/v1/traces` を付加） |
| Agent Host | `OTEL_EXPORTER_OTLP_ENDPOINT` | `/api/public/otel/v1/traces`（そのまま使用） |

> [!TIP]
> `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` (Copilot CLI/TUI) と `COPILOT_OTEL_CAPTURE_CONTENT` (VS Code Copilot Chat) は、プロンプト、応答、ツール引数などの本文を送信するかどうかの設定です。

> [!WARNING]
> `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` と `COPILOT_OTEL_CAPTURE_CONTENT` を `true` にすると、それぞれ Copilot CLI/TUI と VS Code Copilot Chat において、センシティブな情報が永続化されてしまい Langfuse 上で権限のあるユーザーに覗き見られてしまう可能性があります。特に共有環境などでは `false` を設定することを推奨します。

> [!CAUTION]
> plist には Langfuse の Basic 認証情報が保存されます。plist の内容を公開リポジトリ、Issue、ログへ貼り付けないでください。

## 3. Remote User Settings

Remote Tunnel 側の OTel User Settings は不要です。以前追加した `github.copilot.chat.otel.*` と `chat.agentHost.otel.*` をリモート側の User Settings から削除します。Tunnel plist の process env を両 Host の正本とし、設定の競合を避けます。README に記載した Desktop VS Code のローカル User Settings は残してください。

## 4. 接続先と認証の確認

Remote Tunnel の接続先ホスト上で、Langfuse のエンドポイントへ到達できることを確認します。

```bash
curl -i "http://192.168.1.20:16300"
```

Langfuse Web の応答が返れば、ネットワーク経路は確認できています。

その後、VS Code Chat で Copilot へログインし、テストメッセージを送信します。認証状態が不安定な場合は、VS Code のアカウントメニューから GitHub アカウントを一度サインアウトしてから、再度サインインしてください。

## 5. 動作確認

次を確認します。

1. VS Code Chat でメッセージに応答が返る
2. Langfuse Web UI に Trace が作成される
3. Trace の metadata に次が含まれる

```text
langfuse.trace.metadata.execution_origin=vscode-chat
```

VS Code のログに、次のような OpenTelemetry 設定が出力されることも確認できます。

```text
[OTel] Instrumentation enabled
exporter=otlp-http
endpoint=http://192.168.1.20:16300/api/public/otel
captureContent=false
```

macOS の再起動・ログイン後にも、Tunnel が自動復旧し、Copilot Chat のメッセージから Langfuse に Trace が届くことを確認してください。

## 6. CLI/TUI との違い

| 利用経路        | 実行場所                              | 設定方法                           | `execution_origin` |
| --------------- | ------------------------------------- | ---------------------------------- | ------------------ |
| Copilot CLI/TUI | 統合ターミナル                        | `~/.zshrc` の `copilot()` ラッパー | `manual-tui`       |
| VS Code Chat    | Remote Tunnel 接続先の VS Code Server | launchd plist process env | `vscode-chat`      |

Remote Tunnel 経由の VS Code Chat では、ブラウザ側のシェル設定や、ブラウザを開いたホストの `~/.zshrc` は接続先の VS Code Server へ伝搬しません。

また、VS Code Chat の GitHub 認証と、ターミナルで実行する Copilot CLI の認証は別経路です。片方のサインイン状態だけで、もう片方が自動的に有効になるとは限りません。

## 7. セキュリティ上の注意

この構成では、VS Code Server から Langfuse へ HTTP で送信します。LAN 外から利用する場合は、HTTPS、VPN、または TLS 終端するリバースプロキシを使用してください。

`captureContent` および `COPILOT_OTEL_CAPTURE_CONTENT` は `false` に設定し、プロンプト、応答、ツール引数などの本文を送信しない構成を推奨します。

OTLP 認証 header は Agent Host から起動する subprocess に継承され得ます。環境変数全体をログへ出力しないでください。
