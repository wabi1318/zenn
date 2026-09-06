---
title: "ISUCON学習ログ：第3章「基礎的な負荷試験」のアクセスログ集計まで"
emoji: "📝"
type: "idea"
topics: ["isucon", "読書メモ", "nginx", "alp"]
published: false
---

『達人が教えるWebパフォーマンスチューニング』の第3章を読み進めている。
今回は、nginxのアクセスログをJSON形式に変え、alpで集計するところまでのメモ。

## 負荷試験で性能を数値化する

- 負荷試験 → 問題発見 → 修正 → 再度確認、という流れで改善する。
- 性能を数値化するために、リクエストを送って負荷試験を行うソフトウェアがベンチマーカー。
- レイテンシは、リクエストに対する応答を得るまでにかかる時間。
- スループットは、一定時間あたりに処理できるリクエスト数。

## Webサービス側でも計測する

- Webサービス側でも性能を計測できるようにしておくと、負荷試験中だけでなく運用中も確認できる。
- ベンチマーカー自身もCPUやネットワークを使う。
- 本の最初の実験では同じサーバーで動かすが、高速化が進んだらベンチマーカーを別サーバーへ移す。アプリと資源を取り合う影響が大きくなるため。

## nginxのアクセスログをJSON形式にする

- 最初の指標として、URLごとのレイテンシを集計する。
- 標準の`combined`形式には処理時間が入っていないため、記録する設定を追加する。
- `log_format`でJSON形式を定義し、`access_log`でその形式を指定する。

SSMのSession ManagerでEC2に接続して作業した。
Mac側ではなく、EC2内のnginx設定を変更する。

1. 現在の設定を読む。`less`は閲覧用で、終了は`q`。

```bash
sudo less /etc/nginx/nginx.conf
```

2. 編集前の設定をバックアップする。同名のバックアップがある場合は別名にする。

```bash
sudo cp -p /etc/nginx/nginx.conf /etc/nginx/nginx.conf.before-json
```

3. 設定ファイルを編集する。

```bash
sudo nano /etc/nginx/nginx.conf
```

`http {`の中に次を追加する。

```nginx
log_format json escape=json '{"time":"$time_iso8601",'
    '"host":"$remote_addr",'
    '"port":$remote_port,'
    '"method":"$request_method",'
    '"uri":"$request_uri",'
    '"status":"$status",'
    '"body_bytes":$body_bytes_sent,'
    '"referer":"$http_referer",'
    '"ua":"$http_user_agent",'
    '"request_time":"$request_time",'
    '"response_time":"$upstream_response_time"}';
```

既存の`access_log`行は次の形に変更する。

```nginx
access_log /var/log/nginx/access.log json;
```

- `escape=json`は、変数の値に含まれる引用符などをJSONとして扱えるようにする指定。
- 保存はCtrl+O → Enter、終了はCtrl+X。

4. 設定を検査し、成功した場合だけ反映する。

```bash
sudo nginx -t
```

```bash
sudo systemctl reload nginx
```

5. ブラウザでアプリを再読み込みして、新しいログを確認する。

```bash
sudo tail -n 3 /var/log/nginx/access.log
```

- 変更後のログがJSON形式になったことを確認できた。
- 設定を変えても、過去に書かれたログの形式は変わらない。

## 二つの処理時間を見る

- `$upstream_response_time`は、nginxが転送先のWebアプリから応答を受け取るまでの時間。
- `$request_time`は、nginxがリクエストを受信し始めてから、クライアントへの応答送信を終えるまでの時間。
- どちらも秒単位。二つの時間に差があるかも見る。

## alpでアクセスログを集計する

- [alp](https://github.com/tkuchiki/alp)は、JSON形式やLTSV形式などのアクセスログを解析するツール。
- アクセスが多いURL、処理に時間がかかるURL、返却サイズが大きいURLなどを調べられる。詳しく調べる対象を絞れるのが便利。
- 今回はログがEC2の`/var/log/nginx/access.log`にあるため、alpもEC2にインストールした。

ダウンロードと解凍には、一時ディレクトリを使う。

```bash
mkdir -p /tmp/alp-install
cd /tmp/alp-install
```

今回はLinuxのamd64版を使った。
OSとCPUの種類に対応する配布ファイルを選ぶ。

```bash
curl -fL -o alp.tar.gz \
  https://github.com/tkuchiki/alp/releases/download/v1.0.21/alp_linux_amd64.tar.gz
```

展開して、コマンドとして使う場所へ配置する。
`/tmp`は作業用で、実際のインストール先は`/usr/local/bin/alp`。

```bash
tar -xzf alp.tar.gz
sudo install -m 755 alp /usr/local/bin/alp
alp --version
```

## 集計時につまずいたこと

- `cat access.log`では、今いるディレクトリにファイルがなくてエラーになった。実際のパスを指定する必要がある。
- パスを直しても、変更前の通常形式のログが混ざっていて、`alp json`が解析に失敗した。
- 今回は`{`で始まるJSON形式の行だけを渡して集計した。

```bash
sudo grep '^{' /var/log/nginx/access.log | alp json
```

- `cat`が使えないわけではない。すべてJSON形式なら、`cat`で全行を渡せる。
- `alp json --file ファイル名`で、alpから直接ファイルを読むこともできる。ただし、形式が混ざっている問題は同じ。
- ログの項目名を読み替えたり、並べ替えたり、クエリ文字列を含めたり、IDが異なるURLをまとめたりするオプションもある。

## ロードマップとのつながり

- 第1週の「リクエストを送ってアクセスログを確認する」学習につながった。
- 第4週の「負荷試験中にCPUやログを観察する」ために、処理時間を記録して集計する準備ができた。
- 次はベンチマーカーで負荷をかけ、計測結果とアクセスログの集計を比較したい。

## 参照

- [『達人が教えるWebパフォーマンスチューニング』](https://gihyo.jp/book/2022/978-4-297-12846-3) 第3章
- [alp](https://github.com/tkuchiki/alp)
- [学習ロードマップ](https://github.com/wabi1318/isucon-learning-log/blob/main/roadmap/isucon-learning-roadmap-2026.md)
