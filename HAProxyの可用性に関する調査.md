# HAProxy の可用性に関する調査

## リファレンス
- [HAProxy Configuration Manual version 1.4.24](http://cbonte.github.io/haproxy-dconv/configuration-1.4.html)
- [HAProxy Configuration Manual version 1.5-dev19-9](http://cbonte.github.io/haproxy-dconv/configuration-1.5.html)

## 状態の確認方法
バックエンドサーバの状態を含めた HAProxy の状態を確認する3つの方法について説明する。

### 1. ブラウザから確認する方法
指定したURLで状態のレポートを確認することができる。

#### 設定
```txt
backend servers
    stats enable
```
上記の設定によって、`/haproxy?stats`で状態のレポートを確認できるようになる。
URI は `stats uri` オプションで変更できるが、指定できるのは URI のプレフィックスであることに注意。

### 2. UNIXドメインソケット経由で確認する方法
UNIXドメインソケットを経由してコマンドを送ることで、状態を取得したり変更したりすることができる。

#### 設定
```txt
global
    stats socket /var/run/haproxy level admin
```
上記の設定によって、`/var/run/haproxy`というHAProxyと通信するためのソケットが作成される。
`level`パラメータには`user`、`operator`、`admin`を指定することができ、後者になるほど実行可能なコマンドが増える。
また、`user`、`group`、`mode`などのパラメータを指定することで、ソケットのアクセス権限などを設定できる。

その他にも、タイムアウト（`stats timeout <timeout, in milliseconds>`）と同時接続数（`stats maxconn <connections>`）を設定するためのオプションが存在する。

ソケットは HAProxy のいずれか1つのプロセスに関する状態しか扱えない。
そのため、SSLを有効にする場合などで `nbproc` オプションに2以上の値を設定する場合は、`stats bind-process`オプションで対象のプロセスを指定する必要がある。
なお、このオプションは（2013-07-11現在の）開発版であるバージョン1.5で追加されたものであることに注意。
```txt
global
    nbproc 2
    stats bind-process 1

frontend balancer
    bind-process 1
```

#### 使用例
実行できるコマンドの一覧は[マニュアル](http://cbonte.github.io/haproxy-dconv/configuration-1.4.html#9.2)および`help`コマンドで確認できる。

##### バックエンドサーバの状態を確認する
###### コマンド
```txt
# echo 'show stat -1 4 -1' | socat stdio unix-connect:/var/run/haproxy | grep -v -e '^#\|^$' | cut -d ',' -f 2,18 --output-delimiter=':'
```

###### 出力例
```txt
server1:UP
server2:DOWN
```

##### バックエンドサーバをメンテナンスモードにする
###### コマンド
```txt
# echo 'disable server servers/server2' | socat stdio unix-connect:/var/run/haproxy
```

### 3. 状態確認リクエストを利用して確認する方法
特定のURIへのリクエストもしくは特定のアドレスからのリクエストをHAProxyに対する状態確認のためのリクエストとして、HAProxy自身に処理させることができる。

#### 特定のURIへのリクエストを状態確認リクエストとして処理する
```txt
frontend balancer
    monitor-uri /alive
```
上記の設定によって、`/alive` にリクエストを投げると状態を表すステータスコードを持ったレスポンスが返る。

状態は`200`と`503`のいずれかのステータスコードで表され、`monitor fail` オプションで指定した条件によって決まる。
条件の指定には[ACL](http://cbonte.github.io/haproxy-dconv/configuration-1.4.html#7)を使用する。
```txt
frontend balancer
    acl dead nbsrv(servers) eq 0
    monitor-uri /alive
    monitor fail if dead
```
上記の設定では、正常に稼動しているバックエンドサーバが1台でもあれば`200`を返却し、1台もない場合には`503`を返却する。

#### 特定のアドレスからのリクエストを状態確認リクエストとして処理する
```txt
frontend balancer
    monitor-net 192.168.0.252/31
```
上記の設定によって、`192.168.0.252` ~ `192.168.0.255` からのリクエストに対して、ただちに`200`が返却される。

ただし、（2012-07-11現在の）開発版であるバージョン1.5-dev19でSSLを有効にした条件のもとでは、
このオプションが正しく動作することは確認できなかった。

## バックエンドサーバの状態チェック
HAProxy はバックエンドサーバの稼働状態をチェックする機能を有している。

デフォルトでは、HAProxyはバックエンドサーバの状態をチェックせず、常に正常に稼働していると仮定する。
この挙動を変更して、HAProxy にバックエンドサーバの状態をチェックさせるには以下のように`server`オプションに`check`パラメータを指定する。

```txt
backend servers
    server server1 server1:80 check
```
上記の設定によって、`server1`の80番ポートに対してTCPの接続チェックが2秒間隔で実行されるようになる。
接続チェックが2回連続で成功すると`server1`は正常に稼働していると判断され、
接続チェックが3回連続で失敗すると`server1`は正常に稼働していないと判断される。

これらの挙動は下記のようにパラメータで変更できる。
```txt
backend servers
    server server1 server1:80 check addr server2 port 8080 inter 10s rise 5 fall 2
```

それぞれのパラメータの意味は下記の通り。

|パラメータ名|意味|例|
|---|---|---|
|`addr`|チェック対象のアドレス|server2|
|`port`|チェック対象のポート番号|8080|
|`inter`|チェックの実行間隔|10s|
|`rise`|稼働している（UP）と判断するのに必要な連続で成功するチェック回数|5|
|`fall`|稼働していない（DOWN）と判断するのに必要な連続で失敗するチェック回数|2|

デフォルトのTCP接続チェック以外の手段も用意されており、それぞれ下記のオプションで設定する。

|オプション|意味|
|---|---|
|`option httpchk`|HTTPリクエストを送信してレスポンスを調べて判断する|
|`option smtpchk`|SMTPサーバまたはリレーにSMTPコマンドを送信して応答コードが2xxであることを確認する|
|`option mysql-check`|MySQLサーバに Client Authentication パケットと QUIT パケットを送信してレスポンスを調べて判断する|
|`option ssl-hello-chk`|SSLv3の client hello を送信して server hello が返却されることを確認する|

### HTTPリクエストによるバックエンドサーバの状態チェック
`option httpchk`を指定して、HTTPリクエストによる状態チェックを実施する場合の設定について説明する。

```txt
backend servers
    option httpchk
    server server1 server1:80 check
```
上記の設定では、`OPTIONS / HTTP/1.0`というHTTPリクエストが`server1:80`に送信され、
レスポンスコードが`2xx`または`3xx`であれば成功、そうでなければ失敗と判断される。

HTTPリクエストのメソッド、URI、バージョンは`option httpchk`のパラメータで変更できる。
以下に例を示す。

|オプション例|送信されるHTTPリクエスト|
|---|---|
|`option httpchk *`|`OPTIONS * HTTP/1.0`|
|`option httpchk GET /index.html`|`GET /index.html HTTP/1.0`|
|`option httpchk GET /check HTTP/1.1`|`GET /check HTTP/1.1`|

その他の関連するオプション群について以下に簡単にまとめる。

|オプション|意味|
|---|---|
|`http-check disable-on-404`|404を返却したサーバをメンテナンス中であると認識してバランシングの対象から除外する|
|`http-check expect`|成功/失敗の条件を指定する。ステータスコードやリクエストボディから判断することができる|
|`http-check send-state`|リクエストのヘッダにHAProxyサーバの状態を格納して、バックエンドサーバに状態を伝える|

### バックエンドサーバ停止時の動作
バックエンドサーバが停止した際の HAProxy の動作と、それに影響を与えるオプションについて説明する。

#### シナリオ
バックエンドサーバとして設定されている`server1`と`server2`の2台のAPサーバの内の`server1`が停止。
`server1`へのTCPによる接続が不可能になった。

#### バックエンドサーバの状態をチェックしていない場合
##### デフォルトの挙動
`server1`の停止を検出できないため、`server1`に対してリクエストを送信しようとしてTCPの接続に失敗する。
デフォルトでは1秒間隔で3回まで接続をリトライする。リトライが全て失敗するとクライアントに`503 Service Unavailable`を返却する。

##### 挙動を変更する
リトライ回数は`retries`オプションで変更できる。リトライの間隔は変更できない模様。
```txt
defaults
    retries 2
```
`option redispatch`を指定すると、リトライの最後の一回を他のバックエンドサーバに対して実行する。
```txt
defaults
    retries 2
    option redispatch
```
上記の設定により、リトライの1回目を`server1`に対して実行して失敗した場合、2回目のリトライを`server2`に対して実行する。
スティッキーセッションの設定によって振り分け先として`server1`が指定されている場合でも、最後のリトライは`server2`に対して実行される。
`server2`が正常に稼働していれば最後のリトライのレスポンスがクライアントに返却される。

リトライ回数を1回に設定して`option redispatch`を設定しても、
`server1`へリクエストが振り分けられた場合には、`server1`、`server2`という順番で2回のリクエストが送られることになる。
1回目と2回目のリクエストの間には1秒間のインターバルがとられるため、レスポンスの返却にかかる時間が長くなる。

#### バックエンドサーバの状態をチェックしている場合
##### デフォルトの挙動
デフォルトでは2秒間隔で接続チェックが実行され、3回連続で失敗すると正常に稼動していない（DOWN）と判断される。
`server1`が DOWN 状態であると判定されるまで間は、`server1`もバランシングの対象になる。
したがって、この間の挙動は上で述べた状態をチェックしていない場合のそれと同様になり、挙動の変更についても同じである。

`server1`が DOWN 状態であると判定されると、`server1`はバランシングの対象から除外される。
そのため、それ以降のリクエストはすべて`server2`に振り分けられる。

`server1`が再起動などによって再度正常に稼動するようになると、チェックが2回連続で成功した後に正常に稼動している（UP）と判断される。
その結果、`server1`は再びバランシングの対象になる。

##### 挙動を変更する
すでに述べたように、状態チェックの実行間隔や手段などは各種パラメータやオプションによって変更できる。