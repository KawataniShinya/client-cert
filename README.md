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

ここではサーバー証明書およびクライアント証明書の作成をCAで全て実施する。

## composition
- Docker
- Nginx
- Laravel 
- MySQL
- openssl
- curl

## install
### 1. コンテナビルド
```shell
docker compose build
```

### 2. 初期設定のためにコンテナ起動
```shell
docker compose up -d app node db
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

### 5. 初期設定(データベース設定)
```shell
docker compose exec db bash
mysql -u root -proot -e "CREATE USER 'laravelUser' IDENTIFIED BY 'password000'"
mysql -u root -proot -e "GRANT all ON *.* TO 'laravelUser'"
mysql -u root -proot -e "FLUSH PRIVILEGES"
mysql -u root -proot -e "CREATE DATABASE laravel_sample"
exit
```
```shell
docker compose exec app bash
php artisan migrate:fresh --seed
exit
```

### 6. コンテナ停止
```shell
docker compose down
```

### 7. 全コンテナ起動
```shell
docker compose up -d
```

### 8. コンテナ確認
```shell
docker ps
docker compose ps
```

## usage
### 1. HTTP通信
証明書を利用しない通常のHTTP通信。<br>
対象タグ : [https](https://github.com/KawataniShinya/client-cert/tree/http)

#### 1-1. 通信確認
正常応答を確認
```shell
docker compose exec client bash
curl http://localhost.app.sample.jp
```

### 2. HTTPS通信
CA証明書、サーバー証明書を発行し、HTTPS通信可能にする。

#### 2-1. CAの自己署名証明書の作成
##### 2-1-1. CA秘密鍵作成
```shell
docker compose exec ca bash
cd /mnt/share/ca
```
```shell
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out ca.key
```

##### 2-1-2. CA自己署名証明書作成
```shell
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/C=JP/ST=Osaka/L=Chuou-ku/O=Org/OU=Sample/CN=Org CA"
```

#### 2-2. サーバー証明書作成
##### 2-2-1. サーバー用秘密鍵作成
```shell
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out server.key
```

##### 2-2-2. サーバー用証明書署名要求(CSR)作成
```shell
openssl req -new -key server.key -out server.csr -subj "/C=JP/ST=Osaka/L=Chuou-ku/O=Org/OU=Sample/CN=localhost.app.sample.jp"
```

##### 2-2-3. サーバー用証明書署名
```shell
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
```

#### 2-3. 証明書送信
共有ディレクトリへのコピーをサーバーへのファイル連携とみなす。<br>
サーバー送付対象ファイルは下記の通り。
- CA証明書
- サーバー証明書
- サーバー秘密鍵
```shell
cp -p ca.crt server.crt server.key /mnt/share/server
exit
```

クライアント送付対象ファイルは下記の通り。
- CA証明書

```shell
cp -p ca.crt /mnt/share/client
exit
```

#### 2-4. 設定反映(コンテナ再起動)
```shell
docker compose down
docker compose build
docker compose up -d
```

#### 2-5. 通信確認
```shell
docker compose exec client bash
````

HTTP通信はエラー
```shell
curl http://localhost.app.sample.jp
```

HTTPS通信は正常応答を確認
```shell
curl --cacert /mnt/share/client/ca.crt https://localhost.app.sample.jp
```