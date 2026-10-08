# VS Code Remote Tunnel 経由の設定

[English](vscode-remote-tunnel.en.md) | 日本語

VS Code Remote Tunnel を利用して、別ホストのブラウザから Mac の VS Code に接続する場合の設定です。

```mermaid
flowchart LR
  subgraph MBA[クライアント]
    A[ブラウザ / vscode.dev]
  end
  subgraph MINI[macOS Tunnel ホスト]
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

## 1. 前提

README の「OpenTelemetry 直接送信」にある共通認証環境ファイルを、接続先ホスト上に作成済みであることを前提にします。

```text
~/.config/langfuse/copilot-otel.env
```

接続先の Mac に VS Code を導入し、ターミナルで `code` コマンドを使えるようにします。Tunnel サービスのインストール、plist の生成、LaunchAgents の symlink 作成は、次のスクリプトで実行します。共通 env の冒頭にある2つの capture 変数は、構築者が `true` / `false` を選択してから進めてください。

VS Code Remote Tunnel の詳細は、[VS Code 公式ドキュメント](https://code.visualstudio.com/docs/remote/tunnels) を参照してください。

## 2. Tunnel サービスのインストールと設定

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

接続先 Mac のログインユーザーで以下を実行します。冒頭の `TUNNEL_NAME` を指定してください。共通 env の読み込み、plist がない場合のサービス導入（初回は認証が必要な場合があります）、OTel 設定、plist の lint、LaunchAgents の symlink 作成、サービスのロードまで実行します。両 Host の設定には plist の process env を使用するため、Remote OTel User Settings の設定作業はありません。

```bash
(
set -eu
umask 077

TUNNEL_NAME="macmini"
ENV_FILE="$HOME/.config/langfuse/copilot-otel.env"

# VS Code が生成した正本を使用します。
PLIST="$HOME/com.visualstudio.code.tunnel.plist"
LAUNCH_AGENT="$HOME/Library/LaunchAgents/com.visualstudio.code.tunnel.plist"

if [[ ! -r "$ENV_FILE" ]]; then
  echo "環境ファイルが見つかりません: $ENV_FILE" >&2
  exit 1
fi

unset LANGFUSE_BASE_URL OTEL_EXPORTER_OTLP_HEADERS \
  COPILOT_OTEL_CAPTURE_CONTENT OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT
source "$ENV_FILE"
: "${LANGFUSE_BASE_URL:?}"
: "${OTEL_EXPORTER_OTLP_HEADERS:?}"
LANGFUSE_BASE_URL="${LANGFUSE_BASE_URL%/}"
for capture_value in "$COPILOT_OTEL_CAPTURE_CONTENT" "$OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT"; do
  case "$capture_value" in
    true|false) ;;
    *) echo "Capture settings must be true or false" >&2; exit 1 ;;
  esac
done

OTEL_RESOURCE_ATTRIBUTES_VALUE="langfuse.trace.metadata.execution_origin=vscode-chat"

# LaunchAgents の既存の通常ファイルを上書きしません。
mkdir -p "$(dirname "$LAUNCH_AGENT")"
if [[ -e "$LAUNCH_AGENT" && ! -L "$LAUNCH_AGENT" ]]; then
  echo "既存ファイルを確認・退避してください: $LAUNCH_AGENT" >&2
  exit 1
fi
if [[ ! -f "$PLIST" ]]; then
  command code tunnel service install --name "$TUNNEL_NAME"
fi
if [[ ! -r "$PLIST" ]]; then
  echo "Tunnelサービスのplistが見つかりません: $PLIST" >&2
  exit 1
fi
LABEL="$(/usr/libexec/PlistBuddy -c 'Print :Label' "$PLIST")"
chmod 600 "$PLIST"
DOMAIN="gui/$(id -u)"

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
set_plist_env OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT "$OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT"
set_plist_env COPILOT_OTEL_ENABLED "true"
set_plist_env COPILOT_OTEL_CAPTURE_CONTENT "$COPILOT_OTEL_CAPTURE_CONTENT"

chmod 600 "$PLIST"
plutil -lint "$PLIST"

if [[ ! -L "$LAUNCH_AGENT" ]] || [[ "$(readlink "$LAUNCH_AGENT")" != "$PLIST" ]]; then
  ln -sfn "$PLIST" "$LAUNCH_AGENT"
fi
if launchctl print "$DOMAIN/$LABEL" >/dev/null 2>&1; then
  launchctl bootout "$DOMAIN/$LABEL"
fi
launchctl bootstrap "$DOMAIN" "$PLIST"
launchctl kickstart "$DOMAIN/$LABEL"
)
```

この例は、macOS で VS Code が `$HOME/com.visualstudio.code.tunnel.plist` を生成する場合の設定です。このファイルを正本とし、`~/Library/LaunchAgents` には symlink を配置してログイン後に自動ロードさせます。リンク先の配置場所に通常ファイルがある場合は、内容を確認・退避してから実行してください。配布形態によって場所が異なる場合は、実際の plist を使用し、別のサービス定義を作成しないでください。

共通 env の変更を反映する場合は、このスクリプトを再実行します。読み込んだ認証情報は subshell 内に限定します。稼働中の Tunnel は、設定反映時に一時切断されます。

| プロセス | 環境変数 | endpoint |
| --- | --- | --- |
| Extension Host | `COPILOT_OTEL_ENDPOINT` | `/api/public/otel`（`/v1/traces` を付加） |
| Agent Host | `OTEL_EXPORTER_OTLP_ENDPOINT` | `/api/public/otel/v1/traces`（そのまま使用） |

> [!TIP]
> `COPILOT_OTEL_CAPTURE_CONTENT` は Extension Host の本文送信を制御し、`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` は Agent Host の本文送信を制御します。後者は Copilot CLI/TUI でも使用されます。

> [!WARNING]
> いずれかを `true` にすると、対応する pipeline からプロンプト、応答、ツール引数などのセンシティブな本文が送信・永続化される可能性があります。両方を `false` に設定することを推奨します。

> [!CAUTION]
> plist には Langfuse の Basic 認証情報が保存されます。plist の内容を公開リポジトリ、Issue、ログへ貼り付けないでください。

## 3. 接続先と認証の確認

Remote Tunnel の接続先ホスト上で、Langfuse のエンドポイントへ到達できることを確認します。

```bash
curl -i "http://192.168.1.20:16300"
```

Langfuse Web の応答が返れば、ネットワーク経路は確認できています。

ブラウザで `https://vscode.dev/tunnel/` を開き、Tunnel の導入時に使用したアカウントでログインして、設定した Tunnel 名を選択します。VS Code Chat で Copilot へログインし、テストメッセージを送信します。

## 4. 動作確認

次を確認します。

1. VS Code Chat でメッセージに応答が返る
2. Langfuse Web UI に Trace が作成される
3. Trace の metadata に次が含まれる

```text
langfuse.trace.metadata.execution_origin=vscode-chat
```

VS Code のログで endpoint と capture 値が選択した設定と一致することを確認します。以下は Extension Host の capture を `false` にした場合の例です。

```text
[OTel] Instrumentation enabled
exporter=otlp-http
endpoint=http://192.168.1.20:16300/api/public/otel
captureContent=false
```

macOS の再起動・ログイン後にも、Tunnel が自動起動し、Copilot Chat のメッセージから Langfuse に Trace が届くことを確認してください。

## 5. CLI/TUI との違い

| 利用経路        | 実行場所                              | 設定方法                           | `execution_origin` |
| --------------- | ------------------------------------- | ---------------------------------- | ------------------ |
| Copilot CLI/TUI | 統合ターミナル                        | `~/.zshrc` の `copilot()` ラッパー | `manual-tui`       |
| VS Code Chat    | Remote Tunnel 接続先の VS Code Server | launchd plist process env | `vscode-chat`      |

Remote Tunnel 経由の VS Code Chat では、ブラウザ側のシェル設定や、ブラウザを開いたホストの `~/.zshrc` は接続先の VS Code Server へ伝搬しません。

また、VS Code Chat の GitHub 認証と、ターミナルで実行する Copilot CLI の認証は別経路です。片方のサインイン状態だけで、もう片方が自動的に有効になるとは限りません。

## 6. セキュリティ上の注意

この構成では、VS Code Server から Langfuse へ HTTP で送信します。LAN 外から利用する場合は、HTTPS、VPN、または TLS 終端するリバースプロキシを使用してください。

Extension Host は `COPILOT_OTEL_CAPTURE_CONTENT=false`、Agent Host は `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=false` に設定し、両 pipeline からプロンプト、応答、ツール引数などの本文を送信しない構成を推奨します。

OTLP 認証 header は Agent Host から起動する subprocess に継承され得ます。環境変数全体をログへ出力しないでください。
