RabbitMQ
========

RabbitMQ は、AMQP (Advanced Message Queuing Protocol) の実装です。
見た目1つのメッセージキューシステムを複数のマシンに分散させて動かすことができるので、高い信頼性とスケーラビリティを備えています。

チュートリアル
--------------

### "Hello World!"

http://www.rabbitmq.com/tutorials/tutorial-one-python.html

- キューにメッセージを入れる(キューシステムにメッセージを送信する)プログラムを "Producer" と言います
- メッセージの列を "Queue" と言います
  - キューに入れられるメッセージ数に制限はありません
  - キューには名前を付けられます。つまりたくさんのキューを1つのシステムで使えます。
- キューからメッセージを受け取るプログラムを "Consumer" と言います


rabbitmq-server を Java クライアントのライブラリ使って操作してみます (Scala)。

`build.sbt` は以下のようにします。

```scala
scalaVersion := "2.11.2"

libraryDependencies ++= Seq(
  "com.rabbitmq" % "amqp-client" % "3.3.5"
)
```

Producer を作ります。これを実行すると、"Hello, World!" という文字列を hello という名前のキューに enqueue します。

```scala
package main

import com.rabbitmq.client.{ConnectionFactory, Connection, Channel}

object Send extends App {

  val queueName = "hello"

  val factory = new ConnectionFactory
  factory.setHost("localhost")
  val connection = factory.newConnection
  val channel = connection.createChannel

  channel.queueDeclare(queueName, false, false, false, null)
  channel.basicPublish("", queueName, null, "Hello, World!".getBytes)
  println(" [x] Sent message")

  channel.close
  connection.close
}
```

Consumer を作ります。これを実行すると、hello というキューから1つずつ dequeue して、内容を標準出力に表示します。キューが空になったら入ってくるまで待ちます。

```scala
package main

import com.rabbitmq.client.{ConnectionFactory, Connection, Channel, QueueingConsumer}

object Recv extends App {

  val queueName = "hello"

  val factory = new ConnectionFactory
  factory.setHost("localhost")
  val connection = factory.newConnection
  val channel = connection.createChannel

  channel.queueDeclare(queueName, false, false, false, null)
  println(" [*] Waiting for messages. To exit press Ctrl-C")

  val consumer = new QueueingConsumer(channel)
  channel.basicConsume(queueName, true, consumer)

  while (true) {
    val delivery = consumer.nextDelivery // キューが空の場合はここでブロック
    println(" [x] Received " ++ new String(delivery.getBody))
  }

  channel.close
  connection.close
}
```




#### 気づき

一意な名前をつけて複数のキューを作ることができる。
callback を指定して start_consuming() をすると、キューに値がたまり次第何度もタスクを実行する。一度だけではない。


### Work queues

http://www.rabbitmq.com/tutorials/tutorial-two-python.html

デフォルトはラウンドロビン。

メッセージを受け取ったあと処理の途中で死んだらどうするのか？ということが書かれています。





信頼性
------

http://www.rabbitmq.com/reliability.html

このページではどのように、信頼性を実現するための AMQP と RabbitMQ の特徴を使うかを説明します。

### 何が故障の原因か

ネットワーク障害が一番一般的な故障の原因かもしれません。
ネットワーク障害だけではなく、ファイヤーウォールがアイドル状態の接続を割り込みしたり、ネットワーク障害はいつも即座に発見できるわけではありません。

もちろんハードウェア障害やソフトウェアの障害もいつでも起こり得ます。

メッセージ欠落


その他
------

キューはメモリ上に持っているっぽいです。

```sh
$ sudo rabbitmq-server start &
$ send.py
$ sudo rabbitmqctl list_queues
Listing queues ...
hello   1
...done.
$ sudo rabbitmqctl stop
$ sudo rabbitmq-server start &
$ sudo rabbitmqctl list_queues
Listing queues ...
...done.
```

一回停止させるとキューの内容がなくなっています。
永続性をもたせるための方法がある？


ソースコード
------------

[rabbitmq-server](https://github.com/rabbitmq/rabbitmq-server/)


エントリーポイントは `rabbitmq:boot/0`。サーバーを実行するときは[rabbitmq-server](https://github.com/rabbitmq/rabbitmq-server/blob/master/scripts/rabbitmq-server)というスクリプトを実行するので、ここからわかる。

`src/rabbit.erl` に定義されている `?APPS` にある[アプリケーション](http://www.erlang.org/doc/man/application.html)がはじめにロードされる。`[os_mon, mnesia, rabbit]` の3つ。

- [os_mon](http://www.erlang.org/doc/man/os_mon_app.html) は OS のモニター。
- [mnesia](http://www.erlang.org/doc/man/mnesia.html) は Erlang の分散データベースマネージメントシステム。
- rabbit は自分自身。

`start_apps/1` 内で呼び出している `run_boot_steps/1` で、`-rabbit_boot_step` が依存関係の順にソートされ、mfa (Module Function Arguments の略だと思われる) が実行される。

この `-hogehoge` はユーザーが自由に使うことができる。[モジュール属性](http://erlang.org/doc/reference_manual/modules.html)という。<その宣言があるモジュール名>:module_info(<属性名>) でその内容を取得することができる。
rabbitmq で実際に module_info を使って属性を取得している関数は `rabbit_misc:module_attributes/1` と `rabbit_misc:all_module_attributes/1`。

`rabbit:run_step/3` で `rabbit_boot_step` のキーが mfa である値を実行している。値は `{モジュール名, 関数名, 引数}` になっているので、[`erlang:apply/3`](http://www.erlang.org/documentation/doc-6.1/erts-6.1/doc/html/erlang.html#apply-3) を使って実行している。

これによって各モジュールがスタート。

### 各モジュール

モジュール名 (ここでいうモジュール名は、`-rabbit_boot_step({MODULE_NAME, ...})` の MODULE_NAME のこととします)

#### rabbit

#### pre_boot

ダミーモジュール。依存関係のためだと思われる。

#### codec_correctness_check

`rabbit_binary_generator:check_empty_frame_size()` が呼ばれる。
`EMPTY_FRAME_SIZE` が正しく設定されているかどうかチェックする。
`rabbit_binary_generator:create_frame/3` で作られるものがちゃんと8バイトかどうか。

#### file_handle_cache



#### database

`rabbit_mnesia:init()` が呼ばれる。mnesia のセットアップ。

rabbit_table:definitions/0 にいろいろテーブルがある。


#### worker_pool

`rabbit_sup:start_supervisor_child(worker_pool_sup)` が呼ばれ、`worker_pool_sup:start_link()` が実行される。
[`supervisor:start_link/2`](http://www.erlang.org/doc/man/supervisor.html#start_link-2) は、Module:init/1 を与えられた引数でもって呼び出すという関数なので、`worker_pool_sup:init/1` が実行される。


### その他

rabbit_sup は supervisor 関連の便利関数がおいてあるモジュール。

例えば `{mfa, {rabbit_sup, start_supervisor_child, [worker_pool_sup]}}` の場合は、`rabbit_sup:start_supervisor_child(worker_pool_sup)` が呼ばれ、[supervisor:start_child/2](http://www.erlang.org/doc/man/supervisor.html#start_child-2) によって `worker_bool_sup:start_link()` が実行される。
