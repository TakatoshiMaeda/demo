AMQP (Advanced Message Queuing Protocol)
========================================

http://www.amqp.org/

AMQP とは、メッセージキューのオープンなプロトコルです。有名な実装には RabbitMQ, ActiveMQ などがあります。

AMQP の目指すところ

- 全ての AMQP クライアントは全ての AMQP サーバ と相互運用性を持つ
- いろいろなプログラミング言語から使える(言語間の通信)
- プロプライエタリなプロトコルからの解放
- クラウドサービスとしてのメッセージングを可能に
- 発展的な出版-購読モデル
- トランザクション

メッセージングサーバのことをブローカーと呼んでいます。

仕様
----

PDF: http://docs.oasis-open.org/amqp/core/v1.0/amqp-core-complete-v1.0.pdf

(RabbitMQ の実装は AMQP-0.9.1 で、0.9.1 と 1.0 は全然違うらしい。[参照](https://www.rabbitmq.com/specification.html))


Publisher
---------

Exchange
--------

ルーターっぽい

Binding
-------

Message Queue
-------------

Consumer
--------


参考文献
--------

- http://www.amqp.org/
  - 公式サイト
- http://docs.oasis-open.org/amqp/core/v1.0/amqp-core-complete-v1.0.pdf
  - 仕様書
- http://www.slideshare.net/somic/introduction-to-amqp-messaging-with-rabbitmq
  - AMQP と RabbitMQ のわかりやすいスライド
- http://labs.gree.jp/blog/2010/06/262/
  - GREE のブログ(日本語)
  - Exchange の説明
- http://perezvon.hatenablog.com/entry/20090105/1231183771
  - ブログ記事(日本語)
  - リンク集
