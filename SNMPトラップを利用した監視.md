SNMPトラップとZabbixを連携させる手順について説明する。

ZabbixでSNMPトラップを扱う方法は複数ある。
本文書で示す方法の概要は下記のとおり。

1. snmptrapdがSNMPトラップを受信する
2. snmptrapdはトラップをSNMPTTに渡す
3. SNMPTTはトラップをパーズしたのち、整形した結果をファイルに書き込む
4. ZabbixのSNMP trapperが3のファイルからトラップを読み込む
5. Zabbixに登録されているSNMPインタフェースの中から、トラップの発生元のIP（またはDNS名）に一致するインタフェースを探す。
6. インタフェースに登録されているアイテムの中に`snmptrap[regex]`キーを持つアイテムがあり、`regex`とトラップを比較してマッチするならば、トラップが`snmptrap[regex]`アイテムの値になる。マッチするアイテムがなく、`snmptrap.fallback`キーを持つアイテムがあるならば、トラップが`snmptrap.fallback`アイテムの値になる

### 前提条件
- SNMPのコミュニティ名: public
- AMI: ami-39b23d38 (Amazon Linux AMI - EBS-Backed 64-bit)
- Zabbix Server 2.0.6 (zabbix20-server-mysql-2.0.6-2.el6)

### 目的
SNMPトラップをZabbixのWeb UIで確認する

### 設定手順

#### snmptrapd
snmptrapdをZabbix Serverが動作しているサーバ上で動かして、SNMPトラップを受信する。

Amazon Linuxではsnmptrapdはnet-snmpパッケージに含まれている。

```txt
# yum -y install net-snmp
```

設定ファイル`/etc/snmp/snmptrapd.conf`に以下を追記する。

```txt
authCommunity execute public
traphandle default snmptt
```

`/etc/sysconfig/snmptrapd`にsnmptrapdの起動オプションを設定する。

```txt
OPTIONS="-m +ALL -Lsd -On -p /var/run/snmptrapd.pid"
```

snmptrapdを起動する。

```txt
# service snmptrapd start
```

#### SNMPTT
SNMPTT(SNMP Trap Translator)を使って、トラップメッセージを整形してファイルに出力する。

snmpttパッケージをインストールする。

```txt
# yum -y install snmptt --enablerepo=epel
```

設定ファイル`/etc/snmp/snmptt.ini`を以下のように修正する。

|項目|値|
|---|---|
|date_time_format|%H:%M:%S %Y/%m/%d|

`/etc/snmp/snmptt.conf`に整形ルールを記述する。
ここでは以下のように`linkDown`トラップおよび`linkUp`トラップに対する整形ルールを設定する。

```txt
EVENT linkDown .1.3.6.1.6.3.1.1.5.3 "Status Events" Normal
FORMAT ZBXTRAP $aA Link down on interface $1.  Admin state: $2.  Operational state: $3
EVENT linkUp .1.3.6.1.6.3.1.1.5.4 "Status Events" Normal
FORMAT ZBXTRAP $aA Link up on interface $1.  Admin state: $2.  Operational state: $3
```

#### Zabbix Server
`/etc/zabbix_server.conf`を以下のように修正する。

|項目|値|
|---|---|
|SNMPTrapperFile|/var/log/snmptt/snmptt.log|
|StartSNMPTrapper|1|

設定を反映するためにZabbix Serverを再起動する。

```txt
# service zabbix-server restart
```

##### 監視設定
監視対象のホストに、以下のようなアイテムを設定する。

|項目|値|
|---|---|
|Name|SNMP Trap Normal Status Event|
|Type|SNMP trap|
|Key|snmptrap[Normal "Status Events"]|
|Type of information|Log|
|Log time format|hh:mm:ss yyyy/MM/dd|

### 参考
- https://www.zabbix.com/documentation/2.0/manual/config/items/itemtypes/snmptrap
- http://www.slideshare.net/qryuu/zabbix20snm-ptt
- http://www.slideshare.net/HaruyoshiChiyoda/zabbixjp-study4-zabbix20rc1-snmp-traps
