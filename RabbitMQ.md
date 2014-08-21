RabbitMQ
========

RabbitMQ は、AMQP (Advanced Message Queuing Protocol) の実装です。
見た目1つのメッセージキューシステムを複数のマシンに分散させて動かすことができるので、高い信頼性とスケーラビリティを備えています。

チュートリアル
--------------

### "Hello World!"

http://www.rabbitmq.com/tutorials/tutorial-one-python.html


- キューにメッセージを入れるプログラムを "Producer" と言います
- メッセージの列を "Queue" と言います
  - キューに入れられるメッセージ数に制限はありません
- キューからメッセージを受け取るプログラムを "Consumer" と言います

python クライアント `pika` を使います。





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
