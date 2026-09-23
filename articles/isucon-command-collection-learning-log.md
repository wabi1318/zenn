---
title: "ISUCON学習ログ：練習で使うコマンドと調査メモ"
emoji: "🧰"
type: "idea"
topics: ["isucon", "学習記録", "nginx", "alp", "mysql"]
published: true
---

private-isuの練習で使うコマンドと、アクセスログやデータベースの負荷を調べるメモを整理した。
NginxのJSONログとalp、サーバー上のコード操作、静的ファイル配信、MySQLの状態確認を記録する。

## NginxのアクセスログをJSON形式にする

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
構文を確認し、問題がなければ設定を反映する。

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## alpをインストールする

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

VS Codeの拡張機能でMicrosoftのRemote - SSHをインストールする。

1. `Cmd + Shift + P`でコマンドパレットを開く。
2. `Remote-SSH: Connect to Host...`を選ぶ。
3. 普段ターミナルからSSH接続するときと同じ接続先を選ぶ。例えば、普段`ssh isucon-01`なら`isucon-01`を選ぶ。
4. 接続後、「ファイル → フォルダーを開く」で次のパスを指定する。

```text
/home/isucon/private_isu.git/webapp
```

これはサーバー上のフォルダーを直接開く機能。

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

## 静的ファイルをNginxから配信する

静的ファイルの配信は、アプリケーションを経由せずにNginxから直接行う。
サーバー上で`sites-enabled`の設定を確認し、`sites-available/isucon.conf`を編集する。

```bash
ls -la /etc/nginx/sites-enabled/
sudo nano /etc/nginx/sites-available/isucon.conf
```

次の設定を追加する。

```nginx
location ~ ^/(favicon\.ico|css/|js/|img/) {
  root /home/isucon/private_isu/webapp/public/;
  expires 1d;
}
```

`location`は、クライアントからリクエストされたURLのパスに応じて処理の分岐ルールを定義するブロック。
設定を検査してから反映する。

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 投稿画像をファイルから配信する

投稿画像をNginxから配信するため、アプリケーションとNginxを次のように動かす。

1. アプリケーションサーバーはアップロードされた画像を、インスタンス上のファイルとして保存する。
2. 画像のリクエストを最初に受けるNginxは、ファイルがあればそのまま配信する。
3. ファイルがなければ、アプリケーションサーバーへリバースプロキシする。
4. アプリケーションサーバーはMySQLから画像を取得し、ファイルとして保存した上でレスポンスを返す。

`try_files`は、Nginxがリクエストを受けたとき、指定されたパスに物理ファイルがあるかを判定する設定。

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

## プロセスとスレッド

- プロセスはプログラムの単位で、メモリは独立して動く。
- スレッドは処理の実行単位で、メモリを共有する場合がある。
- C10K問題は、クライアント数が1万を超えた辺りでパフォーマンスが極端に落ちる問題。
- マルチプロセス・シングルスレッドでは、クライアントからの1リクエストを1プロセスが処理する。プロセスは処理中にほかのリクエストを処理できず、1リクエストごとに独立したプロセスを生成する。
- シングルプロセス・マルチスレッドでは、1つのプロセスで複数のスレッドを立ち上げる。

## NoSQL

NoSQLは、伝統的なRDBMSとは異なるデータ構造や設計思想を持つデータベースの総称。
固定スキーマを持たず、強い一貫性を持つ代わりに高速で、複数サーバーに分散できる。
RDBMSは強い一貫性を持つため複数サーバーへのデータ分散が難しく、NoSQLは分散できるアーキテクチャになっている。

## MySQLのプロセスを確認する

MySQL上でどのプロセス（スレッド）が動き、どの程度のCPUを使っているかを調べるには、`SHOW PROCESSLIST`を使う。
OSの`top`コマンドのように、MySQL上の処理を確認する。

```sql
SHOW PROCESSLIST;
```

## pt-query-digestでスロークエリを集計する

pt-query-digestは、データベースに負荷をかけている重いクエリを特定、集計するためのパフォーマンス解析ツール。

```bash
sudo apt update
sudo apt install percona-toolkit
pt-query-digest --version
```

スロークエリログを表示する。

```bash
sudo pt-query-digest /var/log/mysql/mysql-slow.log | less
```

解析結果をファイルへ保存する。

```bash
sudo pt-query-digest /var/log/mysql/mysql-slow.log | tee "digest_$(date +%Y%m%d%H%M).txt"
```

解析結果には、次の内容がある。

1. `Overall`：全体統計。
2. `Profile`：サマリとランキング。
3. `Query Report`：個別の詳細レポート。

pt-query-digestは、似たクエリをまとめ、負荷への寄与が大きかった順に表示する。

- `Query ID`：クエリのハッシュ値。
- `Response time`：実行時間の合計と全体に占める割合（秒）。
- `Calls`：実行された回数。
- `R/Call`：1回あたりの時間。

`Response time`の割合が高いクエリは、ボトルネックになる。
`Calls`が多い場合は、N+1問題が発生している典型的なサイン。

## ベンチマーク後に調べる

ベンチマークを回すときは、スロークエリログの設定で`long_query_time = 0`にする。
pt-query-digestで解析し、`Rank 1`から`Rank 3`を確認する。

- `Calls`が多すぎる場合は、アプリのコードを直してN+1問題を解消する。
- `Rows examined`が大きすぎる場合は、`EXPLAIN`を確認してデータベースにインデックスを追加する。
- スローログをリセットしてから再度ベンチマークを回し、効果を測定する。

投稿一覧で使うSQL。

```sql
SELECT `id`, `user_id`, `body`, `created_at`, `mime`
FROM `posts`
ORDER BY `created_at` DESC;
```

N+1問題は、親データを1回取得した後、取得したレコードに関連するデータを個別に問い合わせるクエリがN回発行され、合計N+1回のクエリになって性能が悪くなる問題。
