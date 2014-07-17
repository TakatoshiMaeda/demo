Vyatta起動時に独自のスクリプトを実行して設定を変更する方法について説明する。

### スクリプトの作成
設定変更用のスクリプトを作成する。

スクリプトからVyattaの設定を変更するには`/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper`を利用する。
以下のようなコンフィグレーションモードを使った設定の変更は、

```txt
$ configure
# set system host-name router001
# commit
# exit
```

`/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper`を使用して以下のように書ける。

```bash
$ /opt/vyatta/sbin/vyatta-cfg-cmd-wrapper begin
$ /opt/vyatta/sbin/vyatta-cfg-cmd-wrapper set system host-name rounter001
$ /opt/vyatta/sbin/vyatta-cfg-cmd-wrapper commit
$ /opt/vyatta/sbin/vyatta-cfg-cmd-wrapper end
```

ここでは、以下のようなホスト名を変更するスクリプトを`/opt/vyatta/etc/config/scripts/change-hostname.sh`として作成したとする。

```bash
#!/bin/bash

CFG=/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper

$CFG begin

$CFG set system host-name router001

$CFG commit
$CFG end
```

### スクリプトの登録
起動時に実行するスクリプトを登録する。

起動時に`/etc/rc.local`から呼び出されるスクリプトとして`/opt/vyatta/etc/config/scripts/vyatta-postconfig-bootup.script`が用意されているので、起動したいスクリプトをこのスクリプトから呼び出せばよい。

先ほど作成したスクリプトを起動時に実行させるためには、以下を`/opt/vyatta/etc/config/scripts/vyatta-postconfig-bootup.script`に追記する。

```bash
sg vyattacfg -c /opt/vyatta/etc/config/scripts/change-hostname.sh
```

**注意** `/opt/vyatta/sbin/vyatta-cfg-cmd-wrapper`は`vyattacfg`グループとして実行する必要があるので、`sg`コマンドで`vyattacfg`グループを指定する必要がある