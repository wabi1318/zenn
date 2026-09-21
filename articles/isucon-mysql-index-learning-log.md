---
title: "ISUCON学習ログ：付録Aのスロークエリログとインデックス"
emoji: "🔍"
type: "idea"
topics: ["isucon", "読書メモ", "mysql", "alp"]
published: true
---

『達人が教えるWebパフォーマンスチューニング』の付録A「private-isuの攻略実践」を進めている。
今回は、スロークエリログから調べるSQLを見つけ、スキーマと実行計画を読んでインデックスを考えるところのメモ。
学習中に出てきた計算量、N+1問題、CPUの使われ方も整理した。

## 調べる順番

1. ベンチマークを実行し、スコアとエラーを記録する。
2. 実行中に`htop`でCPUやプロセスの様子を見る。
3. スロークエリログを集計し、時間がかかっているSQLを選ぶ。
4. 対象テーブルのスキーマと、そのSQLの実行計画を読む。
5. 一つ変更して、同じ条件でベンチマークを実行する。

本に載っているスコアは本の環境での結果なので、自分の結果とは分けて読む。
今回のメモには変更前後の実測値を残せていないので、改善幅は次の計測で記録したい。

## htopで負荷を見る

競技用サーバーで、ベンチマーク中に実行する。

```bash
htop
```

- コアごとのCPU使用率と、`mysqld`やアプリのプロセスのCPU使用率を見る。
- 全体のCPU使用率だけでなく、一部のコアに処理が偏っていないかを見る。
- CPUが空いていても、DBの応答やディスクなどを待っている場合がある。空いている理由まではこの画面だけでは決められない。

## MySQLのスロークエリログを有効にする

どのSQLに時間がかかっているかを記録するための設定。
以下は、競技用サーバーのMySQLで作業する。

```bash
sudo mkdir -p /etc/mysql/mysql.conf.d
sudo touch /etc/mysql/mysql.conf.d/mysqld.cnf
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

`mkdir`はディレクトリの用意、`touch`はファイルがなければ作る操作。
設定を編集するのは`nano`。
既存の設定を残し、`[mysqld]`の項目として次を設定する。
同じ項目があれば、重ねて書かずに値を変更する。

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 0
```

- `slow_query_log`：スロークエリログを有効にする。
- `slow_query_log_file`：ログの保存先。
- `long_query_time`：記録対象にする実行時間のしきい値。`0`にして、速いSQLも含めて調べる。
- nanoの保存はCtrl+O → Enter、終了はCtrl+X。

設定ファイルを作るだけでは、MySQLに読み込まれるとは限らない。
`/etc/mysql/my.cnf`の`!includedir`などで、対象ディレクトリが読み込まれる構成かも確認する。
`[mysqld]`はサーバー用の設定なので、クライアント用の節へ入れない。

設定を検査し、エラーがなければ再起動する。

```bash
sudo mysqld --validate-config
```

```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

動いていないときは、起動時のログを見る。

```bash
sudo journalctl -xeu mysql --no-pager
```

再起動後はMySQLに接続して、実際の設定値を確認する。

```bash
sudo mysql -u root
```

```sql
SHOW GLOBAL VARIABLES WHERE Variable_name IN (
  'slow_query_log', 'slow_query_log_file', 'long_query_time', 'log_output'
);
```

`slow_query_log`が`ON`、保存先が指定したパス、`long_query_time`が`0`であることを見る。
ファイルを集計するので、`log_output`に`FILE`が含まれていることも確認する。
ログの記録条件には読み取り行数などの設定も関係する。[MySQL公式のスロークエリログの説明](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)

## スロークエリログは何を読むか

設定後にベンチマークを実行してから、サーバーのシェルで集計する。
MySQLの画面から戻るには`exit`を使う。

```bash
sudo mysqldumpslow -s t -t 10 /var/log/mysql/mysql-slow.log | less
```

`-s t`は合計時間順、`-t 10`は上位10件。
`less`の終了は`q`。

| 見るもの | 読み方 |
| --- | --- |
| `Count` | 同じ形のSQLが何回実行されたか |
| `Time` | 1回あたりの平均時間と、括弧内の合計時間 |
| `Lock` | ロック取得にかかった時間 |
| `Rows` | クライアントへ返した行数。調べた行数とは別 |
| SQL本文 | どのテーブルを、どんな条件で検索しているか |

1回が遅いSQLだけでなく、短いSQLが何度も実行されて合計時間が大きくなっている場合も調べる。
まずは集計上位のSQLを読み、全件の生ログを最初から読むのは後回しにする。
実際の値や読み取り行数が必要になったら、生ログの`Rows_examined`などへ戻る。

`mysqldumpslow`は、似たSQLをまとめるために数値を`N`、文字列を`'S'`へ置き換える。[mysqldumpslowの公式説明](https://dev.mysql.com/doc/refman/8.0/en/mysqldumpslow.html)

```sql
SELECT * FROM comments WHERE post_id = N ORDER BY created_at DESC LIMIT N;
SELECT COUNT(*) AS count FROM comments WHERE post_id = N;
```

- 1本目は、ある投稿のコメントを新しい順に指定件数だけ取得するSQL。
- 2本目は、ある投稿のコメント数を数えるSQL。
- 二つの`N`が同じ数を表すとは限らない。`N`のまま実行せず、調べたい投稿IDや件数へ置き換える。

## スキーマも読む

SQLだけでは、すでにどんな索引があるか分からない。
テーブルの列や型、制約、索引を定義したものが**スキーマ**。
MySQLに接続して、対象の定義を確認する。

```sql
SHOW DATABASES;
USE データベース名;
SHOW CREATE TABLE comments\G
```

`データベース名`は一覧に出た対象の名前へ置き換える。
`\G`は結果を縦に表示する指定で、長いテーブル定義を読みやすくできる。

- `id`、`post_id`、`created_at`がどんな型の列か。
- `PRIMARY KEY`がどの列に付いているか。
- `KEY`や`UNIQUE KEY`にどの列が、どの順番で並んでいるか。
- `NULL`を許すか、一意である必要があるか。
- 投稿やユーザーなど、ほかのテーブルとどのIDで対応するか。外部キー制約の有無と、アプリ上の関係は分けて読む。

**主キー（Primary Key）**は、行を一意に識別するための列、または列の組み合わせ。
列名が`id`だから速いのではなく、その列に主キーなどの索引があるかを確認する。
今回はまず検索に関わる列と索引を読み、無関係な列の詳細は必要になってから調べる。

## インデックスは、探すために整列した情報

「何かでソートされた簡略化したテーブルみたいなもの」と教わった。
最初のイメージとしては、検索に使う値と、元の行を見つけるための情報を並べた索引、と考えると分かりやすかった。

たとえば`post_id`の索引があれば、投稿IDが同じコメントのまとまりを探せる。
コメント全体を先頭から一つずつ確認する**線形探索**に比べて、無関係な行を調べる量を減らせる。
InnoDBの通常の索引はB+木という木構造で、二つに分ける単純な二分探索そのものではない。[^btree]

[^btree]: InnoDBの主キーの索引には行データが入り、主キー以外の索引には主キーの値も入る。「簡略化した別テーブル」は理解のためのイメージ。[MySQL公式の索引構造の説明](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)

### 二分探索と計算量

整列したデータの中央を調べ、目的の値がある範囲を半分ずつに絞るのが**二分探索**。
データが増えたときに、処理の手数がどう増えるかを表すのが**オーダー記法（ビッグオー）**。

| 表記 | 大まかな意味 |
| --- | --- |
| `O(1)` | データ数が増えても、手数が一定。「必ず一瞬」という意味ではない |
| `O(N)` | データ数に比例して手数が増える。全件を調べる処理など |
| `O(log N)` | データ数が増えても、手数は緩やかに増える。二分探索など |

2の累乗も覚えておくと、規模を考えやすい。

```text
2⁵  =     32
2⁸  =    256
2¹⁰ =  1,024
2¹⁶ = 65,536
2¹⁷ = 131,072
```

10万個は`2¹⁶`より大きく、`2¹⁷`より小さい。
二分探索で1件を探す比較回数は、最悪でもおよそ17回と考える。
「近い2の累乗を選ぶ」というより、半分にし続けて残りが1個程度になるまでの回数を見る。

16bitで表せるのは`2¹⁶`通りで、符号なし整数なら`0〜65,535`。
`2¹⁶`そのものを「16bit」と呼ぶこととは区別する。

索引があっても、該当するコメントを大量に返すなら、その分を読む必要がある。
探す部分が`O(log N)`でも、K件を取り出す処理は概念的には`O(log N + K)`と考える。

また、`rows = 100000`は「SQLを10万回実行する」という意味ではない。
1クエリが1秒だったからといって、さらに10万を掛けて10万秒にはしない。
行の読み取り回数と、SQLの実行回数を分ける。

## EXPLAINで実行計画を読む

MySQLがSQLをどんな方法で処理する予定かを示すのが**実行計画**。
まず、投稿IDを具体的な値にして確認する。

```sql
EXPLAIN SELECT * FROM comments
WHERE post_id = 1
ORDER BY created_at DESC
LIMIT 1;
```

| 項目 | 最初に見ること |
| --- | --- |
| `table` | どのテーブルの処理か |
| `type` | 行の探し方。全件を調べるのか、索引で絞るのか |
| `possible_keys` | 候補になった索引 |
| `key` | 実際に選ばれた索引。候補があっても使うとは限らない |
| `rows` | 読み取ると見積もった行数。InnoDBでは推定値 |
| `Extra` | 追加の絞り込みや並べ替えなど |
| `ref` | 索引の値と比較する定数や、別テーブルの列 |

`type`の代表例も覚えておく。

- `const`：主キーや一意索引の全列を定数と比較するなどして、最大1行に絞れる。
- `ref`：一意ではない索引などで、値が一致する行を探す。複数行になり得る。
- `range`：索引のある範囲を読む。
- `index`：索引全体を走査する。索引で少数に絞れたとは限らない。
- `ALL`：テーブル全体を走査する。

**`type`欄の`const`と、`ref`欄の`const`は別の意味。**
`ref = const`は固定の値と比較するという意味で、結果が1行とは限らない。
「constを見たら優勝」ではなく、どの欄なのか、何行読むのかまで見る。

`Extra`は次の違いを確認する。

- `Using index`：必要な列を索引から取得できる。
- `Using where`：条件を使って行を絞り込む。それだけで問題とは判断しない。
- `Using filesort`：索引の並びだけでは済まず、別途並べ替える。必ずディスクを使うという意味ではない。

最初は`key`、`type`、`rows`、`Extra`を優先し、単純な1テーブルのSQLでは`id`や`select_type`などの細部は後回しにする。
複数の列を組み合わせた索引がどこまで使われたか調べるときには、`key_len`なども読む。[EXPLAIN出力の公式説明](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)

## post_idとcreated_atを組み合わせる

`WHERE`にある列を何でも索引に入れる、という覚え方では足りなかった。
今回のSQLは「投稿IDで絞り、その中で新しい順に少数だけ取得する」という処理。
この順番に合うように、複数列の**複合インデックス**を考える。

```sql
ALTER TABLE comments
ADD INDEX post_id_idx (post_id, created_at DESC);
```

- `comments`：索引を追加するテーブル。
- `post_id_idx`：索引の名前。
- `post_id, created_at DESC`：まず投稿IDごとに並び、その中で作成日時の新しい順に並ぶ。
- `DESC`は降順。日時なら新しいものが先になる。

`(post_id)`だけでも投稿の絞り込みに使えるが、`(post_id, created_at DESC)`なら並び順にも対応できる。
一般に複合インデックスは左端の列からの条件で使えるので、列の順番も考える。[複合インデックスの公式説明](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html)

同じ名前で`ADD INDEX`を何度も実行しても、索引の更新にはならない。
`(post_id)`版と複合版は比較する候補であり、先に`SHOW CREATE TABLE`で現在の定義を確認する。

なお、今回のように`post_id`が固定なら、`(post_id, created_at)`の昇順索引を逆向きに読む方法もある。
降順索引が必須と決めつけず、MySQLのバージョンと実行計画で確認する。[降順索引の公式説明](https://dev.mysql.com/doc/refman/8.0/en/descending-indexes.html)

索引を増やすと保存容量が増え、書き込み時にも更新が必要になる。
追加後は同じSQLの`EXPLAIN`とベンチマークを取り直す。
件数を数えるSQLも別に確認する。索引で対象を絞れても、該当するコメントを数える処理は残る。

## N+1問題と、ボトルネックの移動

投稿一覧を1回取得したあと、N件の投稿それぞれに対してコメント取得のSQLを実行すると、合計で1+N回の問い合わせになる。
このように一覧の件数に応じて問い合わせが増えるのが**N+1問題**。
1回のSQLを索引で速くしても、呼び出す回数は減らないので、SQLを実行するアプリ側のループも読む。

Webサービスの処理の流れは、今回なら次のようになる。

```text
リクエスト → nginx → Rubyアプリ（Unicorn）→ MySQL
```

MySQLを速くすると、アプリがDBの応答を待つ時間が減り、アプリ側の処理量が増えることがある。
今度はアプリ側の処理や同時に処理できる数が制限になり、そこを直すと再びDBの負荷が増えることもある。
一度見つけたボトルネックを直した後も、同じ測定を繰り返す。

ただし、MySQLのCPU使用率が下がっただけで「アプリのN+1が原因」とは断定できない。
アプリのCPU使用率、SQLの実行回数、同時に処理できる数、ベンチマーカー側の負荷も確認する。

### Unicornのプロセス数

付録ではRuby実装を使い、Unicornの`worker_processes`を4にする例があった。
リクエストを処理するプロセスを増やし、複数のCPUコアを利用するための設定。

設定ファイルの場所が分からないときに使うコマンドもメモした。

```bash
sudo find / -name 'unicorn_config.rb' 2>/dev/null
```

見つかったファイルが稼働中のサービスで使われているかを確認してから編集する。
Ruby版の設定変更を反映するコマンドは次のとおり。

```bash
sudo systemctl restart isu-ruby.service
```

Node.jsやGoでも、CPUコアの使われ方は確認する。
Unicornの設定をそのまま当てはめられるわけではなく、各実装のプロセス数や並行処理の仕組みを調べる。
CGIという言葉だけで、コアを使えるかどうかは判断しない。

## 計測後のスロークエリログ

次の計測に前回のログが混ざると、変更前後を比べにくい。
一方、書き込み中のログを`rm`するだけでは、MySQLが削除済みのファイルへ書き続けることがある。

計測が終わったら、別名へ移してMySQLに開き直してもらう。
以下の退避先は、まだ存在しない名前を使う。

```bash
sudo mv /var/log/mysql/mysql-slow.log /var/log/mysql/mysql-slow.before-index.log
sudo mysqladmin -u root flush-logs slow
```

`flush-logs`はログの削除ではなく、ファイルを開き直す操作。
`slow`を指定してスロークエリログを対象にする。
MySQLが新しいファイルを作れるディレクトリ権限も必要になる。[MySQL公式のログ管理の説明](https://dev.mysql.com/doc/refman/8.0/en/log-file-maintenance.html)

新しいログへ記録されることを確認し、退避したログは比較が終わってから削除する。
学習を終えるときは、`long_query_time = 0`によるログ出力を続けるかも見直す。

## alpのインストールメモ

SQL単位は`mysqldumpslow`、URL単位はalpで調べる。
前回の記事に続き、今回はチェックサムでダウンロードしたファイルを確認する手順もメモした。

競技用サーバーでCPUの種類を確認する。

```bash
uname -m
```

以下はLinuxで`x86_64`だった場合のamd64版の手順。
`aarch64`など別の結果なら、対応する配布ファイルを選ぶ。
バージョンは学習メモに合わせて[v1.0.21](https://github.com/tkuchiki/alp/releases/tag/v1.0.21)に固定している。

```bash
mkdir -p /tmp/alp-install
cd /tmp/alp-install
curl -fLO https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_linux_amd64.tar.gz
curl -fLO https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_1.0.21_checksums.txt

grep 'alp_linux_amd64.tar.gz$' alp_1.0.21_checksums.txt |
  sha256sum -c -
```

対象ファイルが`OK`と表示されたら展開する。

```bash
tar -xzf alp_linux_amd64.tar.gz
sudo install -m 0755 alp /usr/local/bin/alp
alp --version
```

アクセスログが前回設定したJSON形式なら、ファイルを直接指定して集計できる。

```bash
sudo alp json --file /var/log/nginx/access.log --sort sum -r
```

## 次に記録したいこと

- 索引追加前後の`SHOW CREATE TABLE`と`EXPLAIN`。
- ベンチマークのスコア、エラー、MySQLとアプリのCPU使用率。
- スロークエリログの合計時間と実行回数。
- 件数の多い投稿でも、同じ索引が有効か。
- アプリのどこでコメント取得や件数取得を繰り返しているか。

## 参照

- [『達人が教えるWebパフォーマンスチューニング』](https://gihyo.jp/book/2022/978-4-297-12846-3) 付録A、特にA-3冒頭（本文268〜271ページ）
- [前回の学習ログ：第3章のアクセスログ集計まで](/wabi1318/articles/isucon-load-testing-learning-log)
