---
title: "Claude Codeのシークレット管理を1Passwordで完全統一した話"
emoji: "🔐"
type: "tech"
topics: ["ClaudeCode", "1Password", "セキュリティ", "MCP", "Kubernetes"]
published: true
---

Claude Codeは強力なAI開発ツールですが、素の状態ではAPIキーやシークレットの管理に課題があります。

本記事では、1Passwordを唯一のシークレット基盤として、Claude Codeの開発環境からKubernetesデプロイまでを一貫してセキュアに運用する方法を紹介します。

---

## 1. Claude Codeのセキュリティ課題

Claude Codeを普通にセットアップすると、シークレット管理はこんな状態になりがちです。

### APIキーが平文で散在する

```bash
# .envファイルに直書き
GEMINI_API_KEY=AIzaSy...
XAI_API_KEY=xai-...

# ~/.claude.jsonにも直書き
{
  "mcpServers": {
    "gemini": {
      "env": { "GEMINI_API_KEY": "AIzaSy..." }
    }
  }
}
```

これらのファイルは `.gitignore` で除外していたとしても、ローカルディスク上に平文で存在し続けます。

### MCPサーバーへの環境変数直渡し

Claude CodeのMCPサーバー設定（`~/.claude.json`）では、`env` フィールドにAPIキーを記述するのが標準的な方法です。しかしこれは：

- **設定ファイルにシークレットが埋まる** — `~/.claude.json`はClaude Code自身が頻繁に更新するファイル
- **バックアップや同期で漏洩する** — dotfilesをgit管理している場合にリスクが高い
- **複数マシン間の同期が手動** — 各マシンで個別に設定が必要

### Kubernetesデプロイ時のシークレット管理

開発環境のシークレットとKubernetesのSecretが別管理になりがちです。

```yaml
# secret.yaml を直接 kubectl apply する運用...
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
stringData:
  API_KEY: "sk-..."  # これがgitに入ったら事故
```

### まとめると

| 課題 | リスク |
|------|--------|
| `.env`や設定ファイルに平文APIキー | ディスク上に常時露出 |
| dotfilesのgit管理で漏洩 | パブリックリポジトリに誤push |
| マシン間でシークレットの手動コピー | 不整合・管理漏れ |
| K8s Secretと開発環境が別管理 | 二重管理による更新忘れ |

---

## 2. 設計方針：「1Passwordを唯一のシークレット基盤にする」

解決策として、以下の方針を立てました。

### 原則

**「シークレットの実体は1Passwordにだけ存在する。それ以外の場所には `op://` 参照のみ許可する。」**

具体的には：

1. **平文シークレットをファイルに書かない** — `~/.zshrc`、`.env`、`~/.claude.json` すべて
2. **1Password CLIの `op run` でランタイム注入** — プロセス起動時に1Passwordから取得して環境変数に渡す
3. **開発環境もK8sも同じ1Passwordアイテムを参照** — Single Source of Truth

### なぜ1Passwordか

- **CLIが優秀**: `op run` で任意のコマンドに環境変数を注入できる
- **Touch ID / 生体認証**: macOSアプリ連携で認証がシームレス
- **`op://` URI**: 設定ファイルに参照だけ書ける標準的な仕組み
- **個人プランでも十分**: $2.99/月で全機能利用可能（Service AccountやConnectは使えないが、個人開発には不要）

### アーキテクチャ

```
┌─────────────────────────────────────────────────┐
│                  1Password (Dev Vault)            │
│   ┌───────────┐ ┌───────────┐ ┌───────────────┐  │
│   │Google AI  │ │ XAI       │ │ K8s-MyApp     │  │
│   │Studio     │ │           │ │               │  │
│   │GEMINI_KEY │ │ XAI_KEY   │ │ DB_PASS, etc  │  │
│   └─────┬─────┘ └─────┬─────┘ └───────┬───────┘  │
│         │              │               │          │
└─────────┼──────────────┼───────────────┼──────────┘
          │              │               │
    ┌─────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐
    │ op run     │ │ op run     │ │op-k8s-sync │
    │ (MCP起動)  │ │ (MCP起動)  │ │ (デプロイ)  │
    └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
          │              │               │
    ┌─────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐
    │ Gemini MCP │ │ Grok MCP   │ │ K8s Secret │
    └────────────┘ └────────────┘ └────────────┘
```

---

## 3. 実装

### 3.1 MCPサーバーのop runラップ

Claude CodeのMCPサーバー設定で、`command` を直接のNode.jsやPythonではなく、`op run` でラップします。

**Before（危険）：**
```json
{
  "mcpServers": {
    "gemini": {
      "command": "npx",
      "args": ["-y", "@rlabs-inc/gemini-mcp@latest"],
      "env": { "GEMINI_API_KEY": "AIzaSy..." }
    }
  }
}
```

**After（安全）：**
```json
{
  "mcpServers": {
    "gemini": {
      "command": "/opt/homebrew/bin/op",
      "args": [
        "run", "--no-masking", "--",
        "npx", "-y", "@rlabs-inc/gemini-mcp@latest"
      ],
      "env": {
        "GEMINI_API_KEY": "op://Dev/Google-AI-Studio/GEMINI_API_KEY"
      }
    }
  }
}
```

ポイント：

- **`command` を `op` に変更**: 元のコマンドは `args` の `--` 以降に移動
- **`env` に `op://` 参照**: 1Passwordの Vault名/アイテム名/フィールド名 で指定
- **`--no-masking`**: MCPサーバーの出力がマスクされるのを防止（デバッグ時に重要）
- **Claude Code本体の起動は高速**: `op run` はMCPサーバーの起動時にだけ走るため、Claude Code自体の起動速度には影響しない

### 3.2 1Passwordアイテムの構成

1Passwordの `Dev` Vaultに、用途ごとにアイテムを作成します。

```bash
# Gemini API Key
op item create --vault=Dev --title="Google-AI-Studio" \
  "GEMINI_API_KEY=AIzaSy..."

# Grok (xAI) API Key
op item create --vault=Dev --title="XAI" \
  "XAI_API_KEY=xai-..."
```

**命名規則の注意**: 1Passwordのアイテム名にコロン `:` は使えません（`op://` URIの区切り文字と衝突）。ハイフン `-` を使いましょう。

### 3.3 Claude Code起動の設計

以前は `op run` でClaude Code自体をラップしていました：

```bash
# 旧方式（遅い）
op run --env-file=~/.config/op/env -- claude
```

この方式だと、Claude Code起動時に毎回1Password認証が走ります。日に何度も起動するツールなので、これは体験が悪い。

**現在の方式**では、Claude Code本体は素で起動し、MCPサーバー個別に `op run` ラップしています。

```bash
# 現在の方式（高速）
claude  # 直接起動。APIキーはMCPサーバー利用時にだけ解決される
```

MCPサーバーを初めて使うタイミングでTouch ID認証が1回だけ走り、以降はセッション中キャッシュされます。

### 3.4 Kubernetesシークレットの同期

K8sデプロイでも同じ1Passwordアイテムを参照します。

**マッピングファイル（`k8s/op-secrets.yaml`）：**
```yaml
secret_name: my-app-secrets
namespace: my-app
vault: Dev
item: K8s-MyApp
fields:
  - name: DATABASE_URL
    field: DATABASE_URL
  - name: API_KEY
    field: API_KEY
```

**同期スクリプト（`op-k8s-sync`）の動作：**
```bash
# 1Passwordから値を取得してK8s Secretを作成/更新
op read "op://Dev/K8s-MyApp/DATABASE_URL"
# → kubectl create secret で反映
```

**デプロイ：**
```bash
k8s-deploy              # シークレット同期 + kubectl apply（一括実行）
k8s-deploy --dry-run    # 変更確認のみ
k8s-deploy --skip-secrets  # マニフェストだけ適用
```

開発環境のAPIキーもK8sのシークレットも、すべて1Passwordの `Dev` Vaultが唯一の情報源です。値を変更したいときは1Passwordで編集して `k8s-deploy` を実行するだけ。

---

## 4. 運用してみて：ハマりどころとTips

### `op whoami` が失敗しても慌てない

```bash
$ op whoami
[ERROR] account is not signed in
```

1Passwordアプリ連携モードでは、`op whoami` が失敗しても `op run` は正常に動作する場合があります。これはCLIのセッション管理とアプリ連携が独立した認証経路を持っているためです。

MCPサーバーが動くかどうかは、`op whoami` ではなく、実際にMCPツールを呼んで確認しましょう。

### `~/.claude.json` の編集はjqで

Claude Codeは `~/.claude.json` を頻繁に自動更新します。手動でエディタで編集すると競合する可能性があるため、`jq` での編集が安全です。

```bash
# MCPサーバーの追加
jq '.mcpServers.newserver = {
  "command": "/opt/homebrew/bin/op",
  "args": ["run", "--no-masking", "--", "npx", "-y", "new-mcp-server"],
  "env": { "API_KEY": "op://Dev/NewService/API_KEY" }
}' ~/.claude.json > /tmp/claude.json.tmp && mv /tmp/claude.json.tmp ~/.claude.json
```

### MCP以外のシークレット（Codexプラグイン等）

Claude Codeのプラグイン（Codex等）は独自の認証を持つため、1Passwordラップの対象外です。これらはプラグイン自身の認証フロー（OAuth等）に任せます。

すべてを1Passwordで統一する必要はなく、**ファイルに平文で書かれるシークレットを排除する**ことが目的です。

### 禁止事項チェックリスト

運用中に守っているルールです：

- [ ] `~/.zshrc` に平文のAPIキーを書いていないか
- [ ] `.env` ファイルに実際の値を書いていないか（`op://` 参照のみ許可）
- [ ] `~/.claude.json` の `env` に平文の値がないか
- [ ] `secret.yaml`（平文）をgitにコミットしていないか
- [ ] 1Passwordアイテム名にコロン `:` を使っていないか

---

## 5. 展望

### チーム利用への拡張

現在の構成は1Password Individualプラン（個人利用）が前提です。チームで運用する場合は：

- **1Password Teams/Business**: 共有Vaultでチーム内のシークレットを管理
- **Service Account**: CI/CDパイプラインからの自動アクセス（Individualプランでは利用不可）
- **Connect Server**: セルフホスト型のシークレットサーバー

### CI/CDパイプラインとの統合

GitHub ActionsやGitLab CIでも同じ1Passwordアイテムを参照できれば、開発・ステージング・本番で完全に統一されたシークレット管理が実現します。

```yaml
# GitHub Actions での例（1Password Connect利用）
- uses: 1password/load-secrets-action@v2
  with:
    export-env: true
  env:
    API_KEY: op://Dev/MyApp/API_KEY
```

### Claude Code側の改善に期待すること

- **`op://` 参照のネイティブサポート**: `env` フィールドで `op://` を直接解釈してくれれば、`op run` ラップが不要になる
- **シークレットプロバイダーのプラグイン機構**: 1Password以外（Vault、AWS Secrets Manager等）にも対応可能な汎用的な仕組み
- **MCPサーバー設定の暗号化**: `~/.claude.json` 自体を保護する仕組み

---

## まとめ

| 項目 | 対策 |
|------|------|
| MCPサーバーのAPIキー | `op run` ラップ + `op://` 参照 |
| Claude Code起動速度 | 本体は素で起動、MCPサーバー個別にラップ |
| K8sシークレット | `op-k8s-sync` で1Passwordから同期 |
| 設定ファイル管理 | `jq` で安全に編集 |
| シークレットの唯一の情報源 | 1Password Dev Vault |

「シークレットの実体はどこにあるか？」を常に意識すること。答えが「1Password」以外になった瞬間にリスクが生まれます。

多少の初期設定コストはかかりますが、一度構築してしまえば、セキュアな開発環境が低い運用コストで維持できます。AIツールの利便性とセキュリティの両立を目指す方の参考になれば幸いです。
