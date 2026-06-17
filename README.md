# edge-proxy-stack

Nginx Proxy Manager と Portainer を Docker Compose で構成した、リバースプロキシ／エッジゲートウェイ基盤。
外部からの HTTP/HTTPS トラフィックを単一の入口で受け、SSL を終端して各バックエンドアプリへ振り分けます。

## 概要

複数の Web アプリケーションを 1 台のサーバーで運用する際の「入口（エッジ）」を担う構成です。

- **Nginx Proxy Manager (NPM)**: Web UI でリバースプロキシと Let's Encrypt の SSL 証明書を管理
- **Portainer**: Docker コンテナを GUI で監視・管理

両コンテナを共有ネットワーク `nginx-proxy-manager-network` に接続し、Portainer は外部へ直接公開せず NPM 経由でアクセスする設計です。

## アーキテクチャ

```
                    ┌──────────────────────────────────┐
  Internet ─▶ 80/443 ─▶   Nginx Proxy Manager           │ ─▶ 各バックエンドアプリ
                    │   (リバースプロキシ / SSL 終端)        │     (同一ネットワーク上の他コンテナ)
                    │            :81 管理画面               │
                    └────────────────┬─────────────────┘
                                     │ nginx-proxy-manager-network (external)
                                     ▼
                            Portainer (Docker 管理 UI)
                            ※ :9000 は内部公開のみ。NPM 経由でアクセスする
```

## 技術スタック

| 種別 | 使用技術 |
|------|----------|
| コンテナ基盤 | Docker / Docker Compose |
| リバースプロキシ・SSL | Nginx Proxy Manager `jc21/nginx-proxy-manager:2.12.6` |
| SSL 証明書 | Let's Encrypt（自動発行・更新） |
| コンテナ管理 | Portainer CE (LTS) |

## 主な機能

- **リバースプロキシ**: 複数の Web アプリケーションを単一のドメイン／入口で集約
- **SSL 証明書管理**: Let's Encrypt による証明書の自動発行・更新
- **Web 管理画面**: NPM の GUI でプロキシ設定を直感的に管理
- **データ永続化**: 設定と SSL 証明書をホストにマウントして永続化
- **Portainer 統合**: Docker コンテナの監視・管理（NPM 経由でアクセス）
- **IPv6 無効化**: IPv4 のみでの動作を保証
- **権限昇格の抑止**: Portainer に `no-new-privileges` を付与、Docker ソケットは読み取り専用でマウント

## 前提条件

- Docker と Docker Compose がインストールされていること
- ホスト側でポート 80・443・81（またはカスタムポート）が利用可能であること
- 外部ネットワーク `nginx-proxy-manager-network` が作成されていること（手順 3 で作成）

## セットアップ手順

### 1. リポジトリのクローン

```bash
git clone <repository-url>
cd edge-proxy-stack
```

### 2. 環境変数ファイルの設定

```bash
# 環境変数ファイルの例をコピー
cp env.example .env

# .env を編集して必要に応じてポート番号を変更
# デフォルトでは管理画面はポート 81 で動作
```

### 3. 外部ネットワークの作成

```bash
# スクリプトを使用して作成
chmod +x create-network.sh
./create-network.sh

# または手動で作成
docker network create nginx-proxy-manager-network
```

### 4. 起動

```bash
docker compose up -d
```

### 5. 初期設定

1. **Nginx Proxy Manager 管理画面**
   - ブラウザで `http://your-server-ip:81` にアクセス
   - デフォルトログイン情報（**初回ログイン後すぐに変更すること**）:
     - Email: `admin@example.com`
     - Password: `changeme`

2. **Portainer 管理画面**
   - Portainer はホストへ直接公開していないため、NPM でプロキシホストを 1 つ作成してアクセスする
   - Forward Hostname: `portainer-ce-prod` / Forward Port: `9000`（同一ネットワーク上のコンテナ名で解決）
   - 初回アクセス時に管理者アカウントの作成を求められる

## 使用方法

### 基本的なリバースプロキシの設定

1. **管理画面にログイン** — `http://your-server-ip:81`
2. **プロキシホストの追加** — "Proxy Hosts" → "Add Proxy Host"
   - Domain Names: 公開したいドメイン名
   - Scheme: `http` または `https`
   - Forward Hostname/IP: 転送先のホスト名／IP
   - Forward Port: 転送先のポート番号
3. **SSL 証明書の設定** — "SSL" タブで "Request a new SSL Certificate"（Let's Encrypt）を選択

### ポート一覧

| ポート | 用途 | 公開範囲 |
|--------|------|----------|
| 80 | HTTP トラフィック | ホストへ公開 |
| 443 | HTTPS トラフィック | ホストへ公開 |
| 81 | NPM 管理画面（`ADMIN_PORT` で変更可） | ホストへ公開 |
| 9000 | Portainer 管理画面 | 内部のみ（NPM 経由） |

## セキュリティの考慮事項

- 管理画面（ポート 81 / Portainer）へのアクセスは送信元 IP やファイアウォールで制限する
- デフォルトの認証情報は初回ログイン時に必ず変更する
- 不要なポートへのアクセスはファイアウォールでブロックする
- SSL 証明書の有効期限を監視する
- `.env` などの機密情報は Git 管理対象外とする（`.gitignore` 済み）
- Portainer は Docker ソケットを読み取り専用でマウントしているが、それでも強い権限を持つため公開範囲に注意する

## 環境変数

| 変数名 | デフォルト値 | 説明 |
|--------|-------------|------|
| `ADMIN_PORT` | `81` | Nginx Proxy Manager 管理画面のポート番号 |

## ファイル構成

```
edge-proxy-stack/
├── docker-compose.yml          # Docker Compose 設定ファイル
├── env.example                 # 環境変数設定例
├── .env                        # 環境変数設定ファイル（要作成・Git 管理外）
├── create-network.sh           # 外部ネットワーク作成スクリプト
├── nginx-proxy-manager-data/   # NPM のデータ（永続化）
├── nginx-proxy-manager-ssl/    # SSL 証明書（永続化）
├── portainer-data/             # Portainer のデータ（永続化）
└── README.md                   # 本ファイル
```
