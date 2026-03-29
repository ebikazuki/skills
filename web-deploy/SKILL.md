---
name: web-deploy
description: Webアプリの新規作成からVPSへのデプロイまでを一貫して行うスキル。React + TypeScript + Vite でプロジェクトを作成し、Docker でコンテナ化、nginx リバースプロキシ経由で ebikazuki.com のサブパスに公開する。「Webアプリを作りたい」「新しいアプリを公開したい」「サイトにページを追加したい」「アプリをデプロイしたい」といったリクエストで使用すること。アプリ開発だけでなく、Docker化やnginx設定、ポート割当てなどインフラ周りも含めて対応する。
---

# Web Deploy スキル

Webアプリを新規作成し、VPS（ebikazuki.com）にデプロイするための手順書。
ユーザーの VPS は メモリ 2GB の軽量環境なので、省リソースを意識する。

## 全体フロー

1. プロジェクトのひな形を作成
2. アプリのコードを実装
3. Docker 設定を作成
4. nginx 設定を追加
5. デプロイ（docker compose up → nginx reload）

## 1. プロジェクト作成

### ディレクトリ

`/home/ebikazuki/<app-name>/` に作成する。`<app-name>` はURLパスにもなるので、短く英小文字・ハイフンのみで命名する。

### 技術スタック（デフォルト）

ユーザーから特に指定がなければ以下を使う：

- **フレームワーク**: React 19 + TypeScript
- **ビルドツール**: Vite
- **スタイリング**: CSS Modules
- **テスト**: Vitest + React Testing Library
- **リンター**: ESLint
- **パッケージマネージャー**: npm

### セットアップ手順

```bash
cd /home/ebikazuki
npm create vite@latest <app-name> -- --template react-ts
cd <app-name>
npm install
```

### vite.config.ts

サブパスで配信するため `base` を設定する。これを忘れるとアセットの読み込みが壊れる。

```ts
/// <reference types="vitest/config" />
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  base: '/<app-name>/',
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    maxWorkers: 1,
  },
})
```

### index.html

`lang="ja"` を設定し、タイトルとメタディスクリプションをアプリに合わせて書く。

### CLAUDE.md

プロジェクトルートに CLAUDE.md を作成し、以下を記載する：
- プロジェクト概要
- 技術スタック
- ディレクトリ構造
- 開発コマンド（Docker / ローカル）
- コーディング規約
- デプロイ手順
- アプリ仕様

既存プロジェクト `/home/ebikazuki/chord/CLAUDE.md` を参考にするとよい。

## 2. Docker 設定

VPS のメモリが 2GB なので、alpine ベースの軽量イメージを使い、リソース制限を設ける。

### ポート割当て

新しいアプリのホスト側ポートは、既存の compose.yaml ファイルを調べて決める。

```bash
grep -r '"[0-9]*:80"' /home/ebikazuki/*/compose.yaml
```

現在使用中のポートを確認し、空いている次の番号を使う。Docker アプリは 3000 番台を使用する慣例。

### Dockerfile

```dockerfile
# ---- Build stage ----
FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- Production stage ----
FROM nginx:alpine AS production
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# ---- Development stage ----
FROM node:24-alpine AS development
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]
```

### compose.yaml（本番）

```yaml
services:
  web:
    build:
      context: .
      target: production
    ports:
      - "<PORT>:80"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 128M
```

### compose.dev.yaml（開発オーバーライド）

```yaml
services:
  web:
    build:
      context: .
      target: development
    ports:
      - "5174:5173"
    volumes:
      - .:/app
      - /app/node_modules
    deploy:
      resources:
        limits:
          memory: 512M
```

### .dockerignore

```
node_modules
dist
.git
```

## 3. コンテナ内 nginx 設定

プロジェクトルートに `nginx.conf` を置く。コンテナ内の nginx はアプリ単体を配信する役割。

```nginx
worker_processes 1;
worker_rlimit_nofile 1024;

events {
    worker_connections 256;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 30;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1024;

    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;

        location /<app-name>/ {
            alias /usr/share/nginx/html/;
            index index.html;

            location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
            }

            try_files $uri $uri/ /<app-name>/index.html;
        }

        location = /<app-name> {
            return 301 /<app-name>/;
        }

        location / {
            return 301 /<app-name>/;
        }
    }
}
```

## 4. システム nginx 設定（リバースプロキシ）

### 構成の概要

ホスト側の nginx はリバースプロキシとして、各アプリへのルーティングを担当する。
アプリごとに個別の設定ファイルを管理し、メインの設定ファイルから `include` で読み込む構成にする。

### ディレクトリ構成

```
/etc/nginx/
├── sites-available/
│   └── ebikazuki.conf          # メインのサーバーブロック（SSL・include）
├── sites-enabled/
│   └── ebikazuki.conf -> ../sites-available/ebikazuki.conf
└── app-locations/              # アプリごとのlocation設定
    ├── poke.conf
    ├── typing.conf
    ├── chord.conf
    ├── nenkin.conf
    └── <new-app>.conf
```

### 初回セットアップ（まだ分割されていない場合）

**`/etc/nginx/app-locations/` が既に存在する場合、この初回セットアップはスキップしてよい。** そのまま「新しいアプリを追加するとき」の手順に進む。

既存の `poke-gallery.conf` を分割する。この作業は一度だけ行えばよい。

1. `/etc/nginx/app-locations/` ディレクトリを作成
2. `poke-gallery.conf` から各 `location` ブロックを個別ファイルに切り出す
3. メインの conf を `ebikazuki.conf` としてリネームし、`include /etc/nginx/app-locations/*.conf;` を追加
4. `sites-enabled` のシンボリックリンクを更新
5. `nginx -t` でテスト → `systemctl reload nginx`

**ebikazuki.conf のテンプレート:**

```nginx
server {
    server_name ebikazuki.com www.ebikazuki.com;
    root /home/ebikazuki/poke-gallery;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # アプリごとの location を読み込み
    include /etc/nginx/app-locations/*.conf;

    # --- SSL (Certbot managed) ---
    listen [::]:443 ssl ipv6only=on;
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/ebikazuki.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ebikazuki.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    if ($host = www.ebikazuki.com) {
        return 301 https://$host$request_uri;
    }

    if ($host = ebikazuki.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    listen [::]:80;
    server_name ebikazuki.com www.ebikazuki.com;
    return 404;
}
```

**アプリ個別 conf の例（app-locations/chord.conf）:**

```nginx
location /chord/ {
    proxy_pass http://127.0.0.1:3001/chord/;
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
}
```

### 新しいアプリを追加するとき

`/etc/nginx/app-locations/<app-name>.conf` を作成するだけでよい：

```nginx
location /<app-name>/ {
    proxy_pass http://127.0.0.1:<PORT>/<app-name>/;
    proxy_http_version 1.1;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
}
```

その後：

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## 5. デプロイ

### 手順

```bash
cd /home/ebikazuki/<app-name>

# ビルド＆起動
docker compose up -d --build

# nginx 設定追加（初回のみ）
# /etc/nginx/app-locations/<app-name>.conf を作成済みであること
sudo nginx -t && sudo systemctl reload nginx
```

### 動作確認

```bash
# コンテナの状態確認
docker compose ps

# ローカルでの疎通確認
curl -I http://localhost:<PORT>/<app-name>/

# 公開URLでの確認
curl -I https://ebikazuki.com/<app-name>/
```

### 開発サーバー

```bash
docker compose -f compose.yaml -f compose.dev.yaml up
```

ホスト側 5174 ポートでアクセスできる（ホットリロード有効）。

## チェックリスト

新しいアプリをデプロイする際の確認事項：

- [ ] `vite.config.ts` の `base` がアプリ名と一致しているか
- [ ] `nginx.conf`（コンテナ内）の `location` パスがアプリ名と一致しているか
- [ ] `compose.yaml` のポートが他のアプリと衝突していないか
- [ ] `/etc/nginx/app-locations/<app-name>.conf` のポートが `compose.yaml` と一致しているか
- [ ] `nginx -t` が通るか
- [ ] `https://ebikazuki.com/<app-name>/` でアクセスできるか
