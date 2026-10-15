---
title: "ISUCON学習ログ：練習で使うコマンド集"
emoji: "🧰"
type: "idea"
topics: ["isucon", "学習記録", "nginx", "pprotein", "mysql"]
published: true
---

private-isuの準備、計測、最適化で使うコマンドをまとめる。
初回にツールとログを設定し、pproteinでHTTPアクセスとSQL実行時間を計測する。
結果から遅い処理を選び、SQLやUnicorn、Nginxを調整して同じ条件で測り直す。

## 対象環境

サーバーのコマンドはSSH接続後、sudo権限のある`isucon`ユーザーで実行する。

| 対象 | 例 |
| --- | --- |
| SSH接続先 | `isucon-01` |
| private-isu | `/home/isucon/private_isu.git` |
| Rubyアプリ | `/home/isucon/private_isu.git/webapp/ruby` |
| Rubyサービス | `isu-ruby.service` |
| pproteinの作業先 | `/home/isucon/pprotein` |

配置先とサービス名は実環境に合わせて置き換える。
最初にサーバーで、Rubyサービスの起動先を確認する。

```bash
sudo systemctl show isu-ruby.service -p WorkingDirectory -p ExecStart
```

## 必要なツールを導入する

実行場所：サーバー。
pproteinはHTTPログの集計にalp、SQLログの集計にslpを使う。
下の例では、編集用のVimと、補足のGoのCPUグラフ表示に使うGraphvizも導入する。
ツールの導入はここで一度だけ行う。

```bash
sudo apt-get update && sudo apt-get install -y vim curl ca-certificates graphviz
mkdir -p /home/isucon/bin /home/isucon/.local/bin \
  /home/isucon/pprotein/{config,systemd,data}

(
  set -eu
  case "$(uname -m)" in
    x86_64) arch=amd64 ;;
    aarch64|arm64) arch=arm64 ;;
    *) echo '未対応のアーキテクチャ' >&2; exit 1 ;;
  esac
  work=$(mktemp -d)
  trap 'rm -rf "$work"' EXIT
  cd "$work"
  curl -fL -o pprotein.tar.gz \
    "https://github.com/kaz/pprotein/releases/download/v1.2.4/pprotein_1.2.4_linux_${arch}.tar.gz"
  tar -xzf pprotein.tar.gz
  install -m 755 pprotein pprotein-agent /home/isucon/bin/

  # 既存のalp・slpは上書きしない。
  if ! command -v alp >/dev/null && ! test -x /home/isucon/.local/bin/alp; then
    curl -fL -o alp.tar.gz \
      "https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_linux_${arch}.tar.gz"
    tar -xzf alp.tar.gz
    install -m 755 alp /home/isucon/.local/bin/
  fi
  if ! command -v slp >/dev/null && ! test -x /home/isucon/.local/bin/slp; then
    curl -fL -o slp.tar.gz \
      "https://github.com/tkuchiki/slp/releases/download/v0.2.3/slp_linux_${arch}.tar.gz"
    tar -xzf slp.tar.gz
    install -m 755 slp /home/isucon/.local/bin/
  fi
)

PATH="/home/isucon/.local/bin:$PATH" alp --version
PATH="/home/isucon/.local/bin:$PATH" slp --version
dot -V
```

確認：alp、slp、Graphvizのバージョンが表示されること。
既存のalpやslpが別の場所にある場合は、後述するサービスの`PATH`にもその配置先を含める。

## 計測するログを設定する

実行場所：サーバー。
HTTPログはLTSV形式、SQLログはMySQLのスロークエリログを使う。
LTSVは、項目名と値をタブで区切るログ形式。

### NginxのLTSVログ

SSH接続したサーバーで、次の2ファイルを編集する。

| 編集するファイル | 書く内容 |
| --- | --- |
| `/home/isucon/pprotein/config/nginx-pprotein.conf` | ログの書式を定義する`log_format` |
| `/etc/nginx/sites-available/isucon.conf` | ログの出力先を指定する`access_log` |

2つ目の`isucon.conf`は例なので、以下の手順で実際に使われるファイルを確認する。

#### 1. ログ形式のファイルを作る

```bash
vim /home/isucon/pprotein/config/nginx-pprotein.conf
```

次の内容を保存する。
このファイルは設定の控えで、手順3で`/etc/nginx/conf.d/pprotein.conf`へコピーして反映する。

```nginx
log_format pprotein_ltsv 'time:$time_iso8601\tmethod:$request_method\turi:$request_uri\tstatus:$status\tsize:$body_bytes_sent\treqtime:$request_time';
```

以下は、Nginxが`/etc/nginx/conf.d/*.conf`を`http`内で読み込む構成を前提とする。
読み込みの指定は`/etc/nginx/nginx.conf`で確認できる。
異なる構成では、`log_format`を`http`内から読み込まれる場所へ配置する。

#### 2. private-isuのサイト設定を編集する

有効なサイト設定のリンク先を確認する。

```bash
ls -l /etc/nginx/sites-enabled/
```

private-isuの設定が`/etc/nginx/sites-available/isucon.conf`を指していれば、次のコマンドで開く。
別のファイル名なら、表示されたリンク先へ置き換える。
複数の設定がある場合は、private-isuのホスト名やアプリへの転送先に対応するものを選ぶ。

```bash
sudo vim /etc/nginx/sites-available/isucon.conf
```

既存の`server { ... }`の中へ、次の1行を追加する。
`server`ブロック全体を置き換えず、既存の待受ポートやアプリへの転送設定は残す。

```nginx
access_log /var/log/nginx/pprotein-access.log pprotein_ltsv;
```

同じログ出力が既にあれば追加しない。
対象の`location`で`access_log`を上書きしている場合は、その設定にも同じ出力先と書式を指定する。

#### 3. 配置して反映する

手順1のファイルをNginxの読み込み先へコピーし、構文検査に成功した場合だけ再読み込みする。

```bash
sudo install -D -m 644 /home/isucon/pprotein/config/nginx-pprotein.conf \
  /etc/nginx/conf.d/pprotein.conf &&
  sudo nginx -t &&
  sudo systemctl reload nginx
```

反映に成功したらリクエストを送り、ログを確認する。
`/var/log/nginx/pprotein-access.log`は設定ファイルではなく、Nginxが書き出す計測用ログ。

```bash
curl -I http://localhost/
sudo tail -n 1 /var/log/nginx/pprotein-access.log
```

確認：新しい行に`method:`、`uri:`、`reqtime:`があること。
この手順ではJSON形式のログを追加する必要はない。
既存のログ出力を停止する場合は、他の用途で使われていないことを確認する。

### MySQLのスロークエリログ

`/etc/mysql/mysql.conf.d/mysqld.cnf`の既存の`[mysqld]`内へ設定する。
同じ項目は重複させず、既存の値を変更する。

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 0
log_output = FILE
```
構文検査に成功した場合だけMySQLを再起動する。
再起動中はDB接続が切れる。

```bash
sudo mysqld --validate-config && sudo systemctl restart mysql
sudo systemctl status mysql --no-pager
sudo mysql -e "SHOW VARIABLES WHERE Variable_name IN (
  'slow_query_log', 'slow_query_log_file', 'long_query_time', 'log_output'
);"
```

確認：`slow_query_log=ON`、`long_query_time=0`、`log_output=FILE`と指定したログパスが表示されること。
`--validate-config`が未対応なら、利用中のMySQLで使える検査方法を確認してから続ける。

### エージェントの読み取り権限

pprotein-agentは、アプリが出力したログをpproteinへ渡すプロセス。
サービスの補助グループ`adm`で両方のログを読めるようにする。

```bash
sudo touch /var/log/nginx/pprotein-access.log /var/log/mysql/mysql-slow.log
sudo chown www-data:adm /var/log/nginx/pprotein-access.log
sudo chown mysql:adm /var/log/mysql/mysql-slow.log
sudo chmod 640 /var/log/nginx/pprotein-access.log /var/log/mysql/mysql-slow.log
sudo -n runuser -u isucon -g adm -- test -r /var/log/nginx/pprotein-access.log && echo 'Nginx: OK'
sudo -n runuser -u isucon -g adm -- test -r /var/log/mysql/mysql-slow.log && echo 'MySQL: OK'
```

確認：`Nginx: OK`と`MySQL: OK`の両方が表示されること。
エラーが出る、または`OK`が表示されない場合は、解決するまでサービスの起動へ進まない。
Nginxの実行ユーザーが`www-data`でない場合は、所有者を置き換える。
ログローテーション後も同じ所有者と権限を維持する。

## pproteinを起動する

実行場所：サーバー。
次を`/home/isucon/pprotein/systemd/pprotein.service`へ保存する。

```ini
[Unit]
Description=pprotein
After=network.target

[Service]
User=isucon
WorkingDirectory=/home/isucon/pprotein
Environment="PATH=/home/isucon/.local/bin:/usr/local/bin:/usr/bin:/bin"
Environment=PORT=9000
ExecStart=/home/isucon/bin/pprotein
Restart=on-failure
IPAddressDeny=any
IPAddressAllow=localhost

[Install]
WantedBy=multi-user.target
```

次を`/home/isucon/pprotein/systemd/pprotein-agent.service`へ保存する。
`WorkingDirectory`と`PPROTEIN_GIT_REPOSITORY`は、アプリの配置先に合わせる。

```ini
[Unit]
Description=pprotein-agent
After=network.target

[Service]
User=isucon
SupplementaryGroups=adm
WorkingDirectory=/home/isucon/private_isu.git/webapp/ruby
Environment=PORT=19000
Environment=PPROTEIN_HTTPLOG=/var/log/nginx/pprotein-access.log
Environment=PPROTEIN_SLOWLOG=/var/log/mysql/mysql-slow.log
Environment=PPROTEIN_GIT_REPOSITORY=/home/isucon/private_isu.git
ExecStart=/home/isucon/bin/pprotein-agent
Restart=on-failure
IPAddressDeny=any
IPAddressAllow=localhost

[Install]
WantedBy=multi-user.target
```

公式バイナリは9000番と19000番を全アドレスで待ち受ける。
上記の`IPAddressDeny`と`IPAddressAllow`は、通信をlocalhostへ限定する設定。
この制限が働くには、カーネルのcgroup BPF対応が必要となる。
起動前に、ファイアウォールでも両ポートへの外部アクセスを遮断する。
SSH転送だけではサーバー側の公開ポートを制限できない。

設定を配置し、サービスを起動する。

```bash
sudo install -m 644 /home/isucon/pprotein/systemd/pprotein.service \
  /home/isucon/pprotein/systemd/pprotein-agent.service /etc/systemd/system/ &&
  sudo systemctl daemon-reload &&
  sudo systemctl enable --now pprotein pprotein-agent
sudo systemctl status pprotein pprotein-agent --no-pager
```

確認：両サービスが稼働していること。
既存のサービス定義を変更した場合は、`daemon-reload`後に対象サービスを再起動する。
pproteinの設定と計測結果は`/home/isucon/pprotein/data/`へ保存される。

## 画面を開いて計測する

### 手元のPCから接続する

実行場所：手元のMac。
普段のSSH接続先に9000番を転送する。
専用のSSH設定ファイルを使う場合は、普段使う`-F`の指定も加える。

```bash
ssh -N -L 9000:127.0.0.1:9000 -o ExitOnForwardFailure=yes isucon-01
```

接続したままブラウザで<http://localhost:9000/>を開く。
終了するときはターミナルで`Ctrl+C`を押す。

### 計測先を設定する

画面のグループ計測のtargets設定を次の内容にする。
URLの`127.0.0.1`はpproteinが動くサーバー自身を指す。
初期値のpprof対象を外し、エージェントの19000番からHTTPログとSQLログを取得する。

```json
[
  {
    "Type": "httplog",
    "Label": "nginx",
    "URL": "http://127.0.0.1:19000/debug/log/httplog",
    "Duration": 5
  },
  {
    "Type": "slowlog",
    "Label": "mysql",
    "URL": "http://127.0.0.1:19000/debug/log/slowlog",
    "Duration": 5
  }
]
```

初回は`Duration=5`でCollectを押し、ベンチマークなどで負荷をかける。
HTTPとSQLの両レポートが開けたら、すべての`Duration`を60へ変更する。
レポートが欠ける場合は、本計測へ進まず「結果が出ないとき」を確認する。
HTTPログのURLのまとめ方は、httplog設定の`matching_groups`をアプリのパスに合わせる。

### ベンチマークと同時に計測する

1. グループ計測の`Collect`を押す。
2. すぐに練習環境のベンチマークを実行する。
3. 設定した60秒後、同じグループのレポートを開く。
4. スコアと結果を保存し、コード変更後に同じ条件で測り直す。

| 結果 | 確認すること |
| --- | --- |
| `nginx` | 時間がかかるAPI |
| `mysql` | 時間がかかるSQL |

エージェントは計測開始時のログ末尾から、終了時までに追記された部分を収集する。
ベンチマークごとのログ消去や、ログを開き直すための再読み込みは不要。
計測中はログを消去したり、ローテーションしたりしない。
ログの容量管理は計測とは別に行う。

## 遅いSQLの原因を調べる

pproteinで見つけた遅いSQLを、MySQLクライアントで調べる。
実行場所：サーバー。
一覧から対象DBを選び、`DB_NAME`に設定して接続する。

```bash
sudo mysql -u root

SHOW DATABASES;
USE isucon;
SELECT DATABASE();
```

### EXPLAINで実行計画を確認する

SQLは接続したMySQLクライアント内で実行する。
以下はprivate-isuの`comments`テーブルから、投稿に紐づく最新のコメントを1件取得する例。
`SHOW CREATE TABLE`で定義を、`EXPLAIN`で使われるインデックスと読み取り行数を確認する。

```sql
SHOW TABLES;
SHOW CREATE TABLE comments;
EXPLAIN SELECT * FROM comments
WHERE post_id = 1
ORDER BY created_at DESC
LIMIT 1;
```

### インデックスを追加して比較する

先に`SHOW INDEX`で既存のインデックスを確認する。
次の例は投稿IDによる絞り込みと作成日時による並べ替えに使う候補で、無条件に追加するものではない。
同名や同じ列構成のインデックスがないことを確かめ、必要な場合だけ実行する。

```sql
SHOW INDEX FROM comments;
ALTER TABLE comments
ADD INDEX post_id_idx (post_id, created_at DESC);
```

追加後に同じ`EXPLAIN`とベンチマークを実行し、実行計画、SQLの集計結果、スコアを比較する。
インデックスの作成中に、負荷の増加や処理待ちが起こる場合もある。
効果がなく戻す場合は、今回追加したインデックスだけを削除する。

```sql
ALTER TABLE comments DROP INDEX post_id_idx;
```

### ロック待ちを確認する

ロック待ちが疑わしい場合は、次を実行する。

```sql
SELECT
  wait_started, wait_age, locked_table, locked_index,
  waiting_pid, blocking_pid, waiting_query, blocking_query
FROM sys.innodb_lock_waits;
```

結果が空なら、その時点ではロック待ちがない。
インデックスやSQLを変更したら、同じSQLの実行計画とpproteinの計測結果を比較する。

## アプリを編集してサーバーへ反映する

### VS Codeでサーバーに接続する

実行場所：手元のMac。
VS CodeにMicrosoftのRemote - SSH拡張機能をインストールする。

1. `Cmd + Shift + P`でコマンドパレットを開く。
2. `Remote-SSH: Connect to Host...`を選ぶ。
3. 普段SSH接続するときと同じ`isucon-01`を選ぶ。
4. 「ファイル → フォルダーを開く」で、確認したアプリの配置先を開く。

```text
/home/isucon/private_isu.git/webapp
```

ここで編集するのはサーバー上のファイル。
Nginxの`/etc/nginx/`とMySQLの`/etc/mysql/`は、アプリとは別のフォルダにある。
VS Codeの「ファイルを開く」でパスを指定すれば確認できる。
保存には管理者権限を求められる場合がある。

### Rubyの変更を反映する

サーバーでRubyサービスを再起動し、状態とHTTP応答を確認する。

```bas
sudo systemctl restart isu-ruby.service &&
  sudo systemctl status isu-ruby.service --no-pager
curl -I http://localhost/
```

正常に応答したら、pproteinとベンチマークで変更前後を比較する。
起動に失敗した場合は、退避したファイルへ戻して再起動する。

## Unicornのプロセス数を調整する

実行場所：サーバー。
UnicornはRubyアプリのリクエストを複数のプロセスで処理する。
設定ファイルを探し、`ExecStart`の起動引数などから実際に使われるファイルを確認する。

```bash
sudo find /home/isucon -name unicorn_config.rb 2>/dev/null
sudo systemctl show isu-ruby.service -p WorkingDirectory -p ExecStart
```

たとえば現在の設定が`worker_processes 4`なら、元のファイルを退避したうえで8へ変更して比較する。
次は変更箇所だけの例で、他の設定は残す。

```ruby
worker_processes 8
```

```bash
sudo systemctl restart isu-ruby.service &&
  sudo systemctl status isu-ruby.service --no-pager
```

プロセスを増やすとメモリやDB接続への負荷も変わるため、メモリ使用量、エラー、ベンチマークのスコアを確認する。
改善しなければ元の値へ戻して再起動する。

## Nginxから静的ファイルと画像を配信する

実行場所：サーバー。
Nginxでファイルを直接返すと、そのリクエストをRubyアプリで処理する必要がなくなる。
以下は公開ディレクトリが`/home/isucon/private_isu.git/webapp/public`、アプリの待受先が8080番の例。

```bash
ls -la /etc/nginx/sites-enabled/
ls -ld /home/isucon/private_isu.git/webapp/public
sudo cp -p /etc/nginx/sites-available/isucon.conf \
  "/etc/nginx/sites-available/isucon.conf.before-$(date +%Y%m%d-%H%M%S)"
sudo vim /etc/nginx/sites-available/isucon.conf
```

対象の`server`内へ設定する。
同じ`location`がある場合は既存の定義を変更する。
認証が必要なファイルは静的配信の対象に含めない。

```nginx
location ~ ^/(favicon\.ico|css/|js/|img/) {
  root /home/isucon/private_isu.git/webapp/public/;
  expires 1d;
}

location /image/ {
  root /home/isucon/private_isu.git/webapp/public/;
  expires 1d;
  try_files $uri @app;
}

location @app {
  internal;
  proxy_pass http://localhost:8080;
}
```

`/image/`はファイルが存在すればNginxが返し、存在しなければアプリへ渡す。

```bash
sudo nginx -t && sudo systemctl reload nginx
```

確認：静的ファイルと画像を取得でき、ファイルがない画像のリクエストはアプリへ渡ること。
同じベンチマークで、スコアとHTTPログの集計結果を比較する。

## 結果が出ないとき

実行場所：サーバー。
サービスが停止していないか、ログを読めない状態になっていないかを確認する。

```bash
sudo systemctl status pprotein pprotein-agent isu-ruby.service --no-pager
sudo journalctl -u pprotein -u pprotein-agent -u isu-ruby.service -n 50 --no-pager
sudo journalctl -u nginx -u mysql -n 50 --no-pager
```

HTTPだけ失敗する場合はLTSVログの出力先、SQLだけ失敗する場合はスロークエリログの設定を確認する。
エラーが解消するまで計測を止め、変更後に異常が出た場合は退避した設定へ戻す。