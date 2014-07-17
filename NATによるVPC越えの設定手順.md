### 概要

下図のような3つのVPCからなる構成を考える。
VPC1とVPC2間、VPC2とVPC3間はそれぞれ疎通しているとする。
この文書では、VPC2内のHost-BにNATを設定して、VPC1内のHost-AからVPC3内のHost-Cへの通信を可能にする方法を記す。

具体的には、Host-AからHost-Bの特定のポートへのパケットを、Host-Cの特定のポートへのパケットに変換する。

```
 VPC1              VPC2              VPC3
+--------+        +--------+        +--------+
| Host-A |--------| Host-B |--------| Host-C |
+--------+        +--------+        +--------+
```

### 環境例
以降では下記のような環境を仮定して説明する。
- Host-A：10.0.1.100/24
- Host-B：10.0.2.100/24
- Host-Bのポート：10080/TCP
- Host-C：10.0.3.100/24
- Host-Cのポート：80/TCP

### 設定
#### VyattaにNATを設定する方法
Host-BがVyattaの場合は以下のようなコマンドで設定できる。
```txt
$ configure
# set nat destination rule 100 destination address 10.0.2.100
# set nat destination rule 100 destination port 10080
# set nat destination rule 100 inbound-interface eth0
# set nat destination rule 100 protocol tcp
# set nat destination rule 100 source
# set nat destination rule 100 translation address 10.0.3.100
# set nat destination rule 100 translation port 80
# set nat source rule 200 destination address 10.0.3.100
# set nat source rule 200 outbound-interface eth0
# set nat source rule 200 source address 10.0.1.0/24
# set nat source rule 200 translation address masquerade
# commit
# exit
```

##### 動作確認
Host-CでHTTPサーバを起動して、Host-Aから以下のコマンドを実行するとレスポンスが返る。
```
$ curl 10.0.2.100:10080
```
