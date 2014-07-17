# HAProxy の各種設定方法

## リファレンス
- [HAProxy Configuration Manual version 1.4.24](http://cbonte.github.io/haproxy-dconv/configuration-1.4.html)

## ロギング
HTTP リクエストのログを rsyslog 経由で`/var/log/haproxy.log`に出力する。
###### /etc/haproxy/haproxy.cfg
```txt
global
    log 127.0.0.1 local0

defaults
    log global
    option httplog
```

###### /etc/rsyslog.d/haproxy.conf
```txt
$ModLoad imudp
$UDPServerRun 514
$UDPServerAddress 127.0.0.1
local0.* /var/log/haproxy.log
```

## スティッキーセッション
### アプリケーションのクッキーを利用する方法
アプリケーションが`JSESSIONID`という名前のクッキーを使ってセッションを管理している場合は下記のように設定する。
###### /etc/haproxy/haproxy.cfg
```txt
backend servers
    appsession JSESSIONID len 32 timeout 3h
```

HAProxy はバックエンドサーバからのレスポンスを調べて、指定した名前のクッキー（上記の設定では`JSESSIONID`）の設定要求（おそらく`Set-Cookie`ヘッダ）が含まれていれば、そのクッキーの値とレスポンスを返したサーバを紐付ける。`timeout`パラメータに指定した期間使用されないと、紐付けは解除される。

上記の設定では、クライアントが`JSESSIONID`という名前のクッキーを使ってリクエストした際に、そのクッキーの値に紐付くサーバが存在しない場合には、バランシングのアルゴリズムによって適当なサーバが選択される。
このリクエストのレスポンスに`JSESSIONID`という名前のクッキーの設定要求が含まれていれば紐付けが追加されるが、そうでなければ紐付けは追加されない。
この振る舞いを変更して、後者の場合にリクエストで指定された`JSESSIONID`の値とレスポンスを返したサーバの名前の紐付けを追加するには、以下のように`request-learn`パラメータを指定する。

###### /etc/haproxy/haproxy.cfg
```txt
backend servers
    appsession JSESSIONID len 32 timeout 3h request-learn
```

### 独自のクッキーを利用する方法
`SERVERID`という名前のクッキーに、リクエストを`server1`へ振り分けた場合は`s1`を、`server2`へ振り分けた場合は`s2`を格納してセッションを維持するには下記のように設定する。

###### /etc/haproxy/haproxy.cfg
```txt
backend servers
    cookie SERVERID insert nocache indirect
    server server1 server1:80 cookie s1
    server server2 server2:80 cookie s2
```

`nocache`パラメータを指定すると、レスポンスヘッダに`Cache-Control: private`が追加される。
これにより、クッキーがキャッシュサーバなどにってキャッシュされることを防ぐ。

`insert`パラメータと`indirect`パラメータを同時に指定すると、クッキーがバックエンドサーバに送られなくなる。

## X-Forwarded-For
本来の送信元アドレスをリクエストの`X-Forwarded-For`ヘッダに格納してバックエンドサーバに送信する。
###### /etc/haproxy/haproxy.cfg
```txt
frontend balancer
    option forwardfor
```

### stunnel
stunnel は `X-Forwarded-For`ヘッダの追加をサポートしていない。
メーリングリストには stunnel に`X-Forwarded-For`ヘッダの追加をサポートするパッチが[投稿されている](http://www.stunnel.org/pipermail/stunnel-users/2013-May/004203.html)が、keep-alive 時に正しく動作しないらしい。
