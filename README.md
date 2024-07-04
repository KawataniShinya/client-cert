# client-cert

## Overview
mTLS学習用プロジェクト。<br>
mTLS通信実現方法を確認することを目的に、下記ステップで環境構築を実施できるもの。
1. HTTP通信を可能とする
2. サーバー証明書を組み込み、HTTPS通信を可能とする。
3. クライアント証明書を組み込み、mTLS通信を可能とする。

サーバーに見立てた下記コンテナ間でmTLS通信を確認する。
- ca : 認証局
- app : webサーバー
- client : クライアント

## composition
- Docker
- Nginx
- openssl
- curl

## install
### 1. 全コンテナ起動
```shell
docker compose build
docker compose up -d
```

### 2. コンテナ確認
```shell
docker ps
docker compose ps
```

### 3. 初期設定(webバックエンド)
```shell
docker compose exec app bash
composer install
cp -p /var/www/app/.env.example /var/www/app/.env
php artisan key:generate
exit
```

### 4. 初期設定(webフロント)
```shell
docker compose exec node bash
npm install
exit
```

### 5. コンテナ停止の場合
```shell
docker compose down
```

## usage
### 1. HTTP通信
証明書を利用しない通常のHTTP通信。
#### 1-1. 通信確認
正常応答を確認
```shell
docker compose exec client bash
curl http://localhost.app.sample.jp
```
