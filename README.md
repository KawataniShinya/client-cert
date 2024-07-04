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
対象タグ : [http](https://github.com/KawataniShinya/client-cert/tree/http)

#### 1-1. 通信確認
正常応答を確認
```shell
docker compose exec client bash
curl http://localhost.app.sample.jp
```

### 2. HTTPS通信
CA証明書、サーバー証明書を発行し、HTTPS通信可能にする。<br>
対象タグ : [https](https://github.com/KawataniShinya/client-cert/tree/https)

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

### 3. mTLS
クライアント証明書を発行し、mTLS通信を可能とする。<br>
対象タグ : [mTLS](https://github.com/KawataniShinya/client-cert/tree/mTLS)

#### 3-1. クライアント用秘密鍵作成
```shell
docker compose exec ca bash
cd /mnt/share/ca
```
```shell
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:4096 -out client.key
```

#### 3-2. クライアント用証明書署名要求(CSR)作成
```shell
openssl req -new -key client.key -out client.csr -subj "/C=JP/ST=Osaka/L=Chuou-ku/O=Org/OU=Sample/CN=Client"
```

#### 3-3. クライアント用証明書署名
```shell
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365
```

#### 3-4. クライアント証明書のPKCS#12形式への変換
パスワードは`password123`とする。
```shell
openssl pkcs12 -export -out client.p12 -inkey client.key -in client.crt -passout pass:password123
```

#### 3-5. 証明書送信
共有ディレクトリへのコピーをサーバーへのファイル連携とみなす。<br>
クライアント送付対象ファイルは下記の通り。
- クライアント証明書(PKCS#12形式)

```shell
cp -p client.p12 /mnt/share/client
exit
```

#### 3-6. クライアント証明書の展開
展開に必要パスワードは先に設定した`password123`とする。

##### 3-6-1. クライアント証明書取得
```shell
docker compose exec client bash
cd /mnt/share/client
```
```shell
openssl pkcs12 -in client.p12 -out client.crt -clcerts -nokeys
# (パスワード入力)
```

##### 3-6-2. クライアント秘密鍵取得
```shell
openssl pkcs12 -in client.p12 -out client.key -nocerts -nodes
# (パスワード入力)
```
```shell
exit
```

#### 3-7. 設定反映(コンテナ再起動)
```shell
docker compose down
docker compose build
docker compose up -d
```

#### 3-8. 通信確認
```shell
docker compose exec client bash
````

HTTP通信、クライアント証明書なしのHTTPS通信はエラー。
```shell
curl http://localhost.app.sample.jp
curl --cacert /mnt/share/client/ca.crt https://localhost.app.sample.jp
```

クライアント証明書指定のHTTPS通信は正常応答を確認
```shell
curl --cert /mnt/share/client/client.crt --key /mnt/share/client/client.key --cacert /mnt/share/client/ca.crt https://localhost.app.sample.jp
```

### 4. クライアント証明書の失効
著名されたクライアント証明書を失効させ、アクセス不可にする。

#### 4-1. CA環境セットアップ
CAで失効リストを作成する上で不足している環境をセットアップ。<br>
必要ディレクトリの作成、シリアルナンバーの初期化、CA秘密鍵と証明書の配置を実施。

```shell
docker compose exec ca bash
cd /mnt/share/ca
```
```shell
mkdir -p /etc/pki/CA/{certs,crl,newcerts,private}
touch /etc/pki/CA/index.txt
echo 1000 > /etc/pki/CA/serial
echo 1000 > /etc/pki/CA/crlnumber
cp -p /mnt/share/ca/ca.key /etc/pki/CA/private/cakey.pem
cp -p /mnt/share/ca/ca.crt /etc/pki/CA/cacert.pem
```

#### 4-2. 失効リスト登録
失効させるクライアント証明書のシリアル番号を登録
```shell
openssl ca -revoke client.crt -keyfile ca.key -cert ca.crt -config /etc/ssl/openssl.cnf
```
```shell
cat /etc/pki/CA/index.txt
```

#### 4-3. 証明書失効リスト(CRL)を生成
```shell
openssl ca -gencrl -out /mnt/share/ca/crl.pem -crldays 365 -config /etc/ssl/openssl.cnf
```

#### 4-4. 証明書送信
共有ディレクトリへのコピーをサーバーへのファイル連携とみなす。<br>
サーバー送付対象ファイルは下記の通り。
- CRL(証明書失効リスト)

```shell
cp crl.pem /mnt/share/server/
```
```shell
exit
```

#### 4-5. 設定反映(コンテナ再起動)
```shell
docker compose down
docker compose build
docker compose up -d
```

#### 4-6. 通信確認
認証されていたクライアント証明書を指定したHTTPS通信でもエラー
```shell
curl --cert /mnt/share/client/client.crt --key /mnt/share/client/client.key --cacert /mnt/share/client/ca.crt https://localhost.app.sample.jp
```