---
title: "ISUCON学習ログ：練習で使うコマンド集"
emoji: "🧰"
type: "idea"
topics: ["isucon", "学習記録", "nginx", "alp", "mysql"]
published: true
---

private-isuの練習で使うコマンドを、導入・設定と実行の順にまとめた。

## 最初に導入・設定するもの

ツールの導入とNginx・MySQLの設定を先に済ませる。
Ubuntu/DebianのBash環境をまとめて整えたい場合は、[Bashの補完・履歴の設定手順](https://github.com/sorafujitani/dotfiles/tree/main/dot_config/bash)も参照する（任意）。

### VS CodeのRemote-SSH拡張機能

VS CodeにMicrosoftのRemote - SSH拡張機能をインストールする。

### alp

競技用サーバーでアーキテクチャを確認し、`/tmp`へ移動する。

```bash
uname -m
cd /tmp
```

alpのamd64版をダウンロードして展開し、実行ファイルを配置する。

```bash
curl -fLO https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_linux_amd64.tar.gz
tar -xzf alp_linux_amd64.tar.gz
sudo install -m 0755 alp /usr/local/bin/alp
alp --version
```

### pt-query-digest

```bash
sudo apt update
sudo apt install percona-toolkit
pt-query-digest --version
```

### NginxのアクセスログをJSON形式にする

競技用サーバーでNginxの設定ファイルを開く。

```bash
sudo nano /etc/nginx/nginx.conf
```

`http { ... }`ブロック内にJSON形式のログ設定を追記、または確認する。

```nginx
log_format json escape=json '{"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"request_method":"$request_method",'
    '"request_uri":"$request_uri",'
    '"status":"$status",'
    '"body_bytes_sent":"$body_bytes_sent",'
    '"referer":"$http_referer",'
    '"user_agent":"$http_user_agent",'
    '"request_time":"$request_time",'
    '"response_time":"$upstream_response_time"}';

access_log /var/log/nginx/access.log json;
```

JSON形式を指定するときは、既存の`access_log`設定をコメントアウトする。

### Nginxから静的ファイルを配信する

競技用サーバーで`sites-enabled`を確認し、`sites-available/isucon.conf`を編集する。

```bash
ls -la /etc/nginx/sites-enabled/
sudo nano /etc/nginx/sites-available/isucon.conf
```

```nginx
location ~ ^/(favicon\.ico|css/|js/|img/) {
  root /home/isucon/private_isu/webapp/public/;
  expires 1d;
}
```

### 投稿画像のNginx設定

```nginx
location /image/ {
  root /home/isucon/private_isu/webapp/public/;
  expires 1d;
  try_files $uri @app;
}

location @app {
  internal;
  proxy_pass http://localhost:8080;
}
```

設定を検査し、問題がなければ反映する。

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### MySQLのスロークエリ設定

ベンチマークを回すときは、スロークエリログの設定で`long_query_time = 0`にする。

## JSONログの出力を確認してalpで集計する

古いログやテキスト形式のログを空にしてから、テストリクエストを送る。

```bash
sudo truncate -s 0 /var/log/nginx/access.log
curl -I http://localhost/
```

ログの先頭行がJSON形式であることを確認する。

```bash
sudo head -n 1 /var/log/nginx/access.log
```

`alp json`でアクセスログを解析、集計する。

```bash
sudo alp json \
  --file /var/log/nginx/access.log \
  --method-key request_method \
  --uri-key request_uri \
  --body-bytes-key body_bytes_sent \
  --sort sum -r \
  -m '/posts/[0-9]+,/@\w+,/image/\d+' \
  -o count,method,uri,min,avg,max,sum
```

## ベンチマーク前にログを空にする

ベンチマークを回す直前に、アクセスログとスロークエリログを空にし、ログを開き直す。

```bash
sudo truncate -s 0 /var/log/nginx/access.log
sudo truncate -s 0 /var/log/mysql/mysql-slow.log
sudo mysqladmin flush-logs
sudo systemctl reload nginx
```

## VS Codeでサーバーに接続する

1. `Cmd + Shift + P`でコマンドパレットを開く。
2. `Remote-SSH: Connect to Host...`を選ぶ。
3. 普段ターミナルからSSH接続するときと同じ接続先を選ぶ。例えば、普段`ssh isucon-01`なら`isucon-01`を選ぶ。
4. 接続後、「ファイル → フォルダーを開く」で次のパスを指定する。

```text
/home/isucon/private_isu.git/webapp
```

## Macへコードと設定をコピーする

Mac側で作業ディレクトリを作り、サーバーから`webapp`を取得する。

```bash
mkdir -p ~/work/private-isu-local/server-config/system

rsync -avz \
  isucon-01:/home/isucon/private_isu.git/webapp \
  ~/work/private-isu-local/
```

NginxとMySQLの設定をコピーする。

```bash
rsync -avzL \
  isucon-01:/etc/nginx \
  isucon-01:/etc/mysql \
  ~/work/private-isu-local/server-config/
```

systemdの設定をコピーする。

```bash
rsync -avzL \
  isucon-01:/etc/systemd/system/isu-ruby.service \
  isucon-01:/etc/systemd/system/multi-user.target.wants/nginx.service \
  isucon-01:/etc/systemd/system/multi-user.target.wants/mysql.service \
  isucon-01:/etc/systemd/system/multi-user.target.wants/memcached.service \
  ~/work/private-isu-local/server-config/system/
```

```bash
codex
```

## Rubyの配置先を確認してファイルを反映する

実行中のRubyの配置先を確認する。

```bash
sudo systemctl show isu-ruby -p WorkingDirectory -p ExecStart
```

サーバー上で、変更前の`app.rb`を退避する。

```bash
cp -p /home/isucon/private_isu.git/webapp/ruby/app.rb \
  "/home/isucon/app.rb.before-$(date +%Y%m%d-%H%M%S)"
```

Mac側から確認したファイルをサーバーへ転送する。

```bash
rsync -avz \
  ~/work/private-isu-local/webapp/ruby/app.rb \
  isucon-01:/home/isucon/private_isu.git/webapp/ruby/app.rb
```

## MySQLのプロセスを確認する

```sql
SHOW PROCESSLIST;
```

## pt-query-digestでスロークエリを集計する

スロークエリログを表示する。

```bash
sudo pt-query-digest /var/log/mysql/mysql-slow.log | less
```

解析結果をファイルへ保存する。

```bash
sudo pt-query-digest /var/log/mysql/mysql-slow.log | tee "digest_$(date +%Y%m%d%H%M).txt"
```

## 投稿一覧のSQLを確認する

```sql
SELECT `id`, `user_id`, `body`, `created_at`, `mime`
FROM `posts`
ORDER BY `created_at` DESC;
```
