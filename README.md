fluentdの練習
--------------------------------

# 環境

- VS CodeとDev Containerを使用する
- Ubuntu 22.04 (jammy) のコンテナ上で作業する

以下、コマンドはDev Containerとして起動したUbuntu 22.04のBashで実行するものとする。

# fluentdのインストール

ドキュメント:
<https://docs.treasuredata.com/display/public/PD/Installing+td-agent+on+Ubuntu+and+Debian>

Ubuntu 22.04の場合は以下を実行する:

```sh
curl -L https://toolbelt.treasuredata.com/sh/install-ubuntu-jammy-td-agent4.sh | sh
```

次のコマンドでインストールされていることを確認する

```sh
td-agent --version
#出力例> td-agent 4.4.2 fluentd 1.15.3 (e89092ce1132a933c12bb23fe8c9323c07ca81f5)
```

インストールが完了したらdebファイルを削除する

```sh
rm td-agent-apt-source.deb
```

# デフォルトの設定ファイルを確認する

設定ファイル `/etc/td-agent/td-agent.conf` が作成されているので内容を確認する:

```sh
cat /etc/td-agent/td-agent.conf
```

次のような内容になっている:

```conf
####
## Output descriptions:
##


# Treasure Data (http://www.treasure-data.com/) provides cloud based data
# analytics platform, which easily stores and processes data from td-agent.
# FREE plan is also provided.
# @see http://docs.fluentd.org/articles/http-to-td
#
# This section matches events whose tag is td.DATABASE.TABLE
<match td.*.*>
  @type tdlog
  @id output_td
  apikey YOUR_API_KEY

  auto_create_table
  <buffer>
    @type file
    path /var/log/td-agent/buffer/td
  </buffer>

  <secondary>
    @type file
    path /var/log/td-agent/failed_records
  </secondary>
</match>

## match tag=debug.** and dump to console
<match debug.**>
  @type stdout
  @id output_stdout
</match>

####
## Source descriptions:
##

## built-in TCP input
## @see http://docs.fluentd.org/articles/in_forward
<source>
  @type forward
  @id input_forward
</source>

## built-in UNIX socket input
#<source>
#  type unix
#</source>

# HTTP input
# POST http://localhost:8888/<tag>?json=<json>
# POST http://localhost:8888/td.myapp.login?json={"user"%3A"me"}
# @see http://docs.fluentd.org/articles/in_http
<source>
  @type http
  @id input_http
  port 8888
</source>

## live debugging agent
<source>
  @type debug_agent
  @id input_debug_agent
  bind 127.0.0.1
  port 24230
</source>

#以下略
```

# 設定ファイルを編集

設定ファイル `/etc/td-agent/td-agent.conf` をVS Codeで開く:

```sh
code /etc/td-agent/td-agent.conf
```

以下のように編集して保存する:

```conf
# httpによる入力を受け取る
<source>
  @type http
  @id input_http
  port 8888
</source>

# タグが **.stdout にマッチする場合、標準出力に出力
<match **.stdout>
  @type stdout
  @id output_stdout
</match>

# タグが **.local にマッチする場合、ローカルのログファイルに出力
<match **.local>
  @type file
  @id output_file
  path /var/log/td-agent/test
</match>
```

# 起動

ターミナルで `td-agent` コマンドを実行すると `td-agent` がフォアグラウンドで起動する

```sh
td-agent
```

# テスト

`td-agent` を起動したまま、もう１つのターミナルを起動して以下を実行する

(1) HTTPでタグ `test.stdout` をつけてデータを送信する

```sh
curl -v --data 'json={"foo":"bar"}' 'http://127.0.0.1:8888/test.stdout'
```

=> `td-agent` 起動中のターミナルにメッセージが表示される。

(2) HTTPでタグ `test.local` をつけてデータを送信する

```sh
curl -v --data 'json={"foo":"bar"}' 'http://127.0.0.1:8888/test.local'
```

=> データがログファイルに書き込まれる。

ログファイルの格納先フォルダを確認する:

```sh
ls /var/log/td-agent/test/
#出力例> buffer.b5f9cf524f4f2185d9f3fd07e9f88d190.log
#出力例> buffer.b5f9cf524f4f2185d9f3fd07e9f88d190.log.meta
```

書き込まれた内容を確認するには、次のコマンドを実行する:

```sh
cat /var/log/td-agent/test/*.log
#出力例> 2023-04-21T02:21:58+00:00       test.local      {"foo":"bar"}
```
