---
title: "ISUCON学習ログ：pproteinで計測する"
emoji: "🧰"
type: "idea"
topics: ["isucon", "pprotein", "go", "nginx", "mysql"]
published: true
---

pproteinで、同じベンチマーク中のCPU使用状況、HTTPアクセス、SQL実行時間をまとめて確認する。
初回にツールとログを設定し、以後は画面から計測を開始する。
遅い処理が分かったら、実行計画やロック待ちを調べ、変更後に同じ条件で測り直す。

## 対象環境

[参照元の導入記録](https://scrapbox.io/sorafujitani/pprotein_integ)に合わせ、Goアプリを使う練習環境を対象とする。
private-isuのRuby向けコマンドは、この手順に含めない。
HTTPログとSQLログの集計はアプリの言語によらないが、CPU計測にはGoのpprofを使う。
pprofは、GoアプリのCPU使用状況などを取得する仕組み。

Ubuntu/Debianでsystemdを使い、同じサーバーにGoアプリ、Nginx、MySQL 8.0が導入されていることを前提とする。
サーバーではsudo権限のある`isucon`ユーザーで作業する。
以下の値は例なので、配置先とサービス名を実環境に合わせて置き換える。

| 対象 | 例 |
| --- | --- |
| SSH接続先 | `isucon-01` |
| Goアプリ | `/home/isucon/webapp/go` |
| アプリのGitリポジトリ | `/home/isucon/webapp` |
| Goサービス | `isuconquest.go` |
| pproteinの作業先 | `/home/isucon/pprotein` |

ベンチマークの実行コマンドは、練習環境のものを用意する。
設定ファイルは変更前に退避し、コマンドが失敗したら原因を確認してから続ける。
導入済みなら「画面を開いて計測する」から始める。

## 必要なツールを導入する

実行場所：サーバー。
pproteinはHTTPログの集計にalp、SQLログの集計にslp、CPUのグラフ表示にGraphvizを使う。
ツールの導入はここで一度だけ行う。

```bash
sudo apt-get update && sudo apt-get install -y curl ca-certificates graphviz
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

## Goアプリにpprofを追加する

実行場所：サーバー。
`/home/isucon/webapp/go/profiling.go`へ次を追加し、`main.go`の`main()`から`startProfiling()`を呼ぶ。
既にpprofがある場合は追加せず、その待受先を計測先に指定する。

```go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof"
)

func startProfiling() {
    go func() {
        log.Print(http.ListenAndServe("127.0.0.1:6060", nil))
    }()
}
```

ビルド後、Goサービスを再起動する。

```bash
cd /home/isucon/webapp/go &&
  go build -buildvcs=false -o isuconquest.new . &&
  mv isuconquest.new isuconquest &&
  sudo systemctl restart isuconquest.go
```

確認：`sudo systemctl status isuconquest.go --no-pager`で稼働状態を確認する。
ビルドや起動に失敗した場合は、計測へ進まずサービスのログを確認する。

## 計測するログを設定する

実行場所：サーバー。
HTTPログはLTSV形式、SQLログはMySQLのスロークエリログを使う。
LTSVは、項目名と値をタブで区切るログ形式。

### NginxのLTSVログ

`/home/isucon/pprotein/config/nginx-pprotein.conf`へ次を保存する。
Ubuntu/Debian標準構成では`conf.d/*.conf`を`http`内で読み込む。
異なる構成では、この`log_format`を`http`内へ配置する。

```nginx
log_format pprotein_ltsv 'time:$time_iso8601\tmethod:$request_method\turi:$request_uri\tstatus:$status\tsize:$body_bytes_sent\treqtime:$request_time';
```

計測対象のサイト設定を開き、対象の`server`ブロックへ次を追加する。
同じログ出力が既にある場合は、その設定を使う。
対象の`location`で`access_log`を上書きしている場合は、その設定も確認する。

```nginx
access_log /var/log/nginx/pprotein-access.log pprotein_ltsv;
```

ログ形式を配置し、構文検査に成功した場合だけ反映する。

```bash
sudo install -D -m 644 /home/isucon/pprotein/config/nginx-pprotein.conf \
  /etc/nginx/conf.d/pprotein.conf &&
  sudo nginx -t &&
  sudo systemctl reload nginx
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

`long_query_time=0`は速いSQLも記録するため、ログ量が増える。
練習中の計測用として使い、空き容量を確認する。
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
sudo -u isucon -g adm test -r /var/log/nginx/pprotein-access.log &&
  sudo -u isucon -g adm test -r /var/log/mysql/mysql-slow.log
```

確認：最後の読み取り検査がエラーなく終了すること。
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
WorkingDirectory=/home/isucon/webapp/go
Environment=PORT=19000
Environment=PPROTEIN_HTTPLOG=/var/log/nginx/pprotein-access.log
Environment=PPROTEIN_SLOWLOG=/var/log/mysql/mysql-slow.log
Environment=PPROTEIN_GIT_REPOSITORY=/home/isucon/webapp
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
初期値の9000番はpprotein自身の計測先なので、アプリの6060番とエージェントの19000番へ変更する。

```json
[
  {
    "Type": "pprof",
    "Label": "isuconquest-cpu",
    "URL": "http://127.0.0.1:6060/debug/pprof/profile",
    "Duration": 5
  },
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
3種類のレポートが開けたら、すべての`Duration`を60へ変更する。
レポートが欠ける場合は、本計測へ進まず「結果が出ないとき」を確認する。
HTTPログのURLのまとめ方は、httplog設定の`matching_groups`をアプリのパスに合わせる。

### ベンチマークと同時に計測する

1. グループ計測の`Collect`を押す。
2. すぐに練習環境のベンチマークを実行する。
3. 設定した60秒後、同じグループのレポートを開く。
4. スコアと結果を保存し、コード変更後に同じ条件で測り直す。

| 結果 | 確認すること |
| --- | --- |
| `isuconquest-cpu` | CPUを使っているGoの関数 |
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
sudo mysql -u root -e 'SHOW DATABASES;'
DB_NAME=''
sudo mysql -u root "${DB_NAME:?DB_NAMEを設定してください}"
```

接続後に`SHOW TABLES;`で対象テーブルを確認する。
テーブル定義は`SHOW CREATE TABLE テーブル名;`で調べる。
遅いSQLの先頭に`EXPLAIN`を付けて実行し、使われるインデックスと読み取り行数を確認する。

ロック待ちが疑わしい場合は、次を実行する。

```sql
SELECT
  wait_started, wait_age, locked_table, locked_index,
  waiting_pid, blocking_pid, waiting_query, blocking_query
FROM sys.innodb_lock_waits;
```

結果が空なら、その時点ではロック待ちがない。
インデックスやSQLを変更したら、同じSQLの実行計画とpproteinの計測結果を比較する。

## 結果が出ないとき

実行場所：サーバー。
サービスが停止していないか、ログを読めない状態になっていないかを確認する。

```bash
sudo systemctl status pprotein pprotein-agent isuconquest.go --no-pager
sudo journalctl -u pprotein -u pprotein-agent -u isuconquest.go -n 50 --no-pager
sudo journalctl -u nginx -u mysql -n 50 --no-pager
```

HTTPだけ失敗する場合はLTSVログの出力先、SQLだけ失敗する場合はスロークエリログの設定を確認する。
CPUだけ失敗する場合は、Goアプリのpprof待受先とtargetsのURLを照合する。
エラーが解消するまで計測を止め、変更後に異常が出た場合は退避した設定へ戻す。

## 補足：手動集計が必要な場合

pproteinの画面を使えないときは、導入済みのalpとslpで同じログを集計できる。
pproteinの計測区間と異なり、次のコマンドはログファイル全体を対象にする。
必要な計測区間を切り出したファイルを使う場合は、`--file`のパスを置き換える。

```bash
sudo env PATH=/home/isucon/.local/bin:/usr/local/bin:/usr/bin:/bin \
  alp ltsv --file /var/log/nginx/pprotein-access.log --sort sum -r
sudo env PATH=/home/isucon/.local/bin:/usr/local/bin:/usr/bin:/bin \
  slp my --file /var/log/mysql/mysql-slow.log
```

通常の計測では`pt-query-digest`や`mysqldumpslow`を追加で実行する必要はない。
これらを使う別の調査方法や、Bash補完の導入は任意とする。

## 参考

- [pproteinの導入記録](https://scrapbox.io/sorafujitani/pprotein_integ)
- [pprotein v1.2.4](https://github.com/kaz/pprotein/tree/v1.2.4)
- [alpを呼び出す処理](https://github.com/kaz/pprotein/blob/v1.2.4/internal/extproc/alp/processor.go)
- [slpを呼び出す処理](https://github.com/kaz/pprotein/blob/v1.2.4/internal/extproc/slp/processor.go)
- [計測区間のログを収集する処理](https://github.com/kaz/pprotein/blob/v1.2.4/internal/tail/tail.go)
