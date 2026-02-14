# F2A.dev Personal System

Blog, Portfolio, File Management を備えたパーソナルサイトシステム。

このリポジトリをクローンして `.env` を設定するだけですぐに使えます。

## Requirements

- Docker & Docker Compose
- Reverse proxy (Nginx, Caddy, etc.) for HTTPS termination

## Setup

### 0. クローン & GHCR にログイン

```bash
git clone https://github.com/fltuna/f2a.dev.personal-system.git
cd f2a.dev.personal-system
```

イメージは GitHub Container Registry にホストされています。GitHub の Personal Access Token (PAT) が必要です。

```bash
# PAT を作成: https://github.com/settings/tokens
# 必要なスコープ: read:packages
echo "YOUR_GITHUB_PAT" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

### 1. 設定ファイルを準備

```bash
cp .env.example .env
```

`.env` を編集:

```bash
SITE_URL=https://example.com
PUBLIC_API_BASE_URL=https://api.example.com
DB_PASSWORD=your_secure_database_password
JWT_SECRET=your-random-string-at-least-32-characters-long
```

### 2. 起動

```bash
docker compose up -d
```

### 3. 初期セットアップ

```bash
# データベースマイグレーション
docker compose exec api f2a-cli db migrate

# デフォルト権限グループを作成 (SuperUser, Admin, Editor, Author, Viewer)
docker compose exec api f2a-cli db seed-permissions

# 管理者ユーザーを作成
docker compose exec api f2a-cli user add your@email.com --password YourSecurePassword123! --group SuperUser
```

### 4. リバースプロキシ設定

Frontend (port 3000) と API (port 5000) をリバースプロキシで公開してください。

Nginx の例:

```nginx
# Frontend
server {
    listen 443 ssl;
    server_name example.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# API
server {
    listen 443 ssl;
    server_name api.example.com;

    client_max_body_size 0;  # アプリ側で制限を管理

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 設定

### 環境変数

| 変数 | 必須 | 説明 |
|---|---|---|
| `SITE_URL` | Yes | サイトの公開 URL (e.g. `https://example.com`) |
| `PUBLIC_API_BASE_URL` | Yes | API の公開 URL (e.g. `https://api.example.com`) |
| `DB_PASSWORD` | Yes | PostgreSQL パスワード |
| `JWT_SECRET` | Yes | JWT 署名キー (32文字以上) |
| `GOOGLE_CLIENT_ID` | No | Google OAuth クライアント ID |
| `GOOGLE_CLIENT_SECRET` | No | Google OAuth クライアントシークレット |

### Google OAuth

Google ログインを有効にする場合:

1. [Google Cloud Console](https://console.cloud.google.com/) で OAuth 2.0 クライアント ID を作成
2. 承認済みの JavaScript 生成元に `SITE_URL` のドメインを追加
3. `.env` に `GOOGLE_CLIENT_ID` と `GOOGLE_CLIENT_SECRET` を設定
4. コンテナを再起動: `docker compose restart api`

## CLI ツール

コンテナ内で `f2a-cli` コマンドが使用できます:

```bash
# ユーザー管理
docker compose exec api f2a-cli user list
docker compose exec api f2a-cli user add <email> --password <pw> --group <group>
docker compose exec api f2a-cli user delete <email> --force
docker compose exec api f2a-cli user reset-password <email> --password <new-pw>

# 権限グループ管理
docker compose exec api f2a-cli group list
docker compose exec api f2a-cli group show <name>

# データベース
docker compose exec api f2a-cli db migrate
```

## 更新

※ Migrationはサイト上からも行えます

```bash
docker compose pull
docker compose up -d
docker compose exec api f2a-cli db migrate
```

## データ

永続データは `./data/` 以下にマウントされます:

```
data/
├── postgres/   # PostgreSQL データベース
├── files/      # アップロードファイル
└── system/     # システムファイル (favicon, OGP画像等)
```

### バックアップ

```bash
# データベース
docker compose exec db pg_dump -U f2a f2a > backup.sql

# ファイル (data/ ディレクトリごとコピー)
tar czf backup-files.tar.gz data/files data/system
```

