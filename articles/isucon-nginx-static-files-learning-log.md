---
title: "ISUCON学習ログ：ログ集計とnginxでの静的ファイル配信"
emoji: "🛠️"
type: "idea"
topics: ["isucon", "nginx", "alp", "学習記録"]
published: false
---

private-isuの練習で使うコマンドと、静的ファイル配信について学んだことのメモ。
ログを集計する準備、サーバー上のコードを編集する方法、nginxに配信を任せる考え方を整理した。

nginxのログ設定とalpの導入は[前の学習ログ](https://zenn.dev/zhenyou620/scraps/209fbb2a1dc3ca)に書いた。
MySQLの設定は[スロークエリログとインデックスの学習ログ](https://zenn.dev/zhenyou620/scraps/bc6f200bb25694)を参照する。
ここでは、その後の練習で使うコマンドをまとめる。

## alpでURLごとに集計する

以下は競技用サーバーで実行する。
まずログがJSON形式になっているかを確認する。

```bash
sudo head -n 1 /var/log/nginx/access.log
```

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

ベンチマークを実行する直前に、サーバー側でログを空にする。

```bash
sudo truncate -s 0 /var/log/nginx/access.log
sudo truncate -s 0 /var/log/mysql/mysql-slow.log
sudo mysqladmin flush-logs
sudo systemctl reload nginx
```

## VS Codeでサーバー上のコードを開く

サーバー上のファイルを直接編集する方法として、VS CodeのRemote - SSHを使う。

1. MicrosoftのRemote - SSH拡張機能をインストールする。
2. MacでCmd+Shift+Pを押し、コマンドパレットを開く。
3. `Remote-SSH: Connect to Host...`を選ぶ。
4. 普段SSHで接続しているホストを選ぶ。以下の例では`isucon-01`。
5. 接続後、「ファイル → フォルダーを開く」でサーバー上の作業ディレクトリを指定する。

```text
/home/isucon/private_isu.git/webapp
```

サーバー上のフォルダーを直接開く機能。

## Macへコードをコピーして編集する

手元でコードを読む場合は、`rsync`でサーバーから取得する。
以下はMac側で実行する。

```bash
mkdir -p ~/work/private-isu-local

rsync -avz \
  isucon-01:/home/isucon/private_isu.git/webapp \
  ~/work/private-isu-local/
```

### 確認したファイルだけサーバーへ戻す

Rubyの`app.rb`を修正する場合、最初にサーバー側で元のファイルを退避する。

```bash
cp -p /home/isucon/private_isu.git/webapp/ruby/app.rb \
  "/home/isucon/app.rb.before-$(date +%Y%m%d-%H%M%S)"
```

次に、Mac側から編集したファイルを転送する。

```bash
rsync -avz \
  ~/work/private-isu-local/webapp/ruby/app.rb \
  isucon-01:/home/isucon/private_isu.git/webapp/ruby/app.rb
```

## 静的ファイルをnginxから配信する

CSSやJavaScriptなど、保存済みのファイルを返すだけのリクエストは、nginxから直接配信する。

サーバー側で、有効な設定ファイルを確認する。

```bash
ls -la /etc/nginx/sites-enabled/
```

コマンド集では`/etc/nginx/sites-available/isucon.conf`を編集していた。

```bash
sudo nano /etc/nginx/sites-available/isucon.conf
```

静的ファイル配信の設定。

```nginx
location ~ ^/(favicon\.ico|css/|js/|img/) {
    root /home/isucon/private_isu/webapp/public/;
    expires 1d;
}
```

`location`は、リクエストされたURLのパスに応じて処理を分けるブロック。

設定の検査に成功したら反映する。

```bash
sudo nginx -t
```

```bash
sudo systemctl reload nginx
```

## 投稿画像をファイルとして配信するには

投稿画像もnginxから返すには、アプリ側で画像をファイルに保存する処理が必要になる。

学んだ流れは次のとおり。

1. アップロードされた画像を、アプリがサーバー上のファイルとして保存する。
2. 画像へのリクエストを受けたnginxは、ファイルがあれば直接返す。
3. ファイルがなければ、nginxがアプリへリクエストを転送する。
4. アプリはMySQLから画像を取得し、ファイルとして保存してから応答する。

`try_files`は、指定したパスにファイルがあるかを確認するための設定。

## 参照

- [前の学習ログ：第3章のアクセスログ集計まで](https://zenn.dev/zhenyou620/scraps/209fbb2a1dc3ca)
- [前の学習ログ：付録Aのスロークエリログとインデックス](https://zenn.dev/zhenyou620/scraps/bc6f200bb25694)
- [学習ロードマップ](https://github.com/wabi1318/isucon-learning-log/blob/main/roadmap/isucon-learning-roadmap-2026.md)
