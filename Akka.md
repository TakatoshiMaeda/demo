Akka
====

http://akka.io/

Akka は、アクターモデルをベースにした並行処理のための Scala ライブラリです。


アクターモデル
--------------

アクターモデルとは、並行・分散システムのモデルです。
近年、多数のマシンが協力して動くシステムが一般的になってきましたが、アクターモデルはそのようなシステムを考える際に使いやすいモデルになっています。

アクターモデルは、大雑把に言うと以下の特徴を持っています。

- アクターと呼ばれる計算主体が、**非同期に**メッセージをやり取りすることで計算をすすめます
  - アクター同士で共有する状態はありません
- メッセージを送るときは、チャネルをではなく**アクターのID(アドレス)**を指定してメッセージを送ります
  - 逆に言うと、アドレスを知らないアクターにはメッセージを送れません
- アクターはメッセージを受け取って、そのメッセージに応じて以下の3つの動作を起こします。この、「メッセージの内容に応じて行う動作」のことをアクターが持つ**振る舞い** (behavior) と言います
  1. 新しいアクターを作る
  2. 他のアクターにメッセージを送る
  3. 自分自身の振る舞いを変える

より詳しく正確な定義は、[アクターモデルの教科書](http://dspace.mit.edu/handle/1721.1/6952) を参照してください。


### アクターモデルの得意分野

アクターモデルは以下のことを得意としています。

- 共有された状態がないので、分散化が容易になります
- 共有状態がなく排他制御(ロック)などを考えなくていいので並行プログラミングも容易になります
- システム全体としての耐障害性 (Fault Tolerance) をより優れたものにできます
  - TODO: ここらへんあまりよくわかっていないので調査が必要
  - アクターの状態(例外で死んだり)を、別のアクターが監視することで実現している？

システムのパフォーマンスの向上よりは、落ちないシステムを構築することに向いています。


### アクターモデルを採用している言語・ライブラリ

最も有名なものは、この Akka と、Erlang です。

Erlang は Ericsson 社で開発された言語で、無停止(nine nines)で稼働するシステムを構築することを目的として設計されました。
個々のノードは落ちてもいいから、全体としては落ちないようにすることが重視されています。

その他の言語・ライブラリは以下のリンクを見てください。

http://en.wikipedia.org/wiki/Actor_model#Programming_with_Actors


簡単な使い方
------------

(TODO)

リモートノードとの通信
----------------------

akka-acter では同一ノード (JVM) 内の通信のみで、リモートノードとの通信は行えません。
リモートノードと通信するためには、akka-remote というパッケージが必要です。


参照
- http://doc.akka.io/docs/akka/2.3.3/scala/remoting.html
- http://alvinalexander.com/scala/simple-akka-actors-remote-example
- http://alvinalexander.com/scala/akka-remote-sending-objects-messages

例
- https://github.com/amutake/actor-examples/tree/master/akka/remote


### application.conf

`src/main/resources/application.conf` に、以下のような設定ファイルを置く必要があります。

```scala
akka {
  actor {
    provider = "akka.remote.RemoteActorRefProvider"
  }
  remote {
    enabled-transports = ["akka.remote.netty.tcp"]
    netty.tcp {
      hostname = "127.0.0.1"
      port = 2552
    }
  }
}
```

- リモートアクターと、
- TCP を用いて、
- IP が 127.0.0.1 でポート 2552 で通信

という意味になります。

#### netty

akka-remote は netty を使っています。

netty は、ネットワークアプリケーションにおいて、非同期IOを行えるようにするフレームワークです。



### アクターの生成

基本的には通常のアクターの生成方法と同じですが、アクターシステムとアクターを作る際に必ず名前を明示しなければいけません。
これは、別のノードからアクターを特定できるようにするためです。

```scala
...
val system = ActorSystem("server") // ActorSystem() だとダメ。
val pongActor = system.actorOf(Props[PongActor], name = "pong") // system.actorOf(Props[PongActor]) のみだとダメ。name が必要。
...
```


### リモートアクターの取得

アクターの取得方法には、自分でアクターを作るか、akka の URI を使ってアクターを取得する2種類の方法があります。

リモートアクターの取得は、URI を使って取得することになります。

以下のように `actorSelection` メソッドを使って、別のノードからそのアクターを取得できます。

```scala
...
val server = context.actorSelection("akka.tcp://server@127.0.0.1:2552/user/pong")
...
```

この `akka.tcp://server@127.0.0.1:2552/user/pong` は以下の意味です。

```
akka.<protocol>://<actor system>@<hostname>:<port>/<actor path>
```

protocol, hostname, port については application.conf で指定したものが用いられます。
actor system, actor path については アクターの生成時に使われた名前が用いられます。


#### アクターパス

アクターパス (上でいう `<actor path>`) についてです。このアクターパスについては http://doc.akka.io/docs/akka/2.3.4/general/addressing.html#addressing に書かれています。

例えば、以下のように、メインメソッドのなかで `parent` というアクターを作り、その `parent` というアクターが `child` というアクターを作ったとすると、アクターパスは `/user/parent/child` となります。

```scala
def main(args: Array[String]) = {
  val system = ActorSystem("system")
  val parent = system.ActorOf(Props[ParentActor], name = "parent")
  parent ! "create child"
}

class ParentActor extends Actor {
  def receive = {
    case "create child" => context.actorOf(Props[ChilcActor], name = "child")
  }
}

class ChildActor extends Actor {
  def receive = ...
}
```

`/user` は、`ActorSystem.actorOf` を使ってつくられたアクターであることを表す名前になっています。参照: [Top-Level Scopes for Actor Paths](http://doc.akka.io/docs/akka/2.3.4/general/addressing.html#top-level-scopes-for-actor-paths)

また、同じアクターパスを持つアクターを作ろうとすると、`akka.actor.InvalidActorNameException: actor name [<actor name>] is not unique!` という例外が発生します。上の例で言うと、`parent` に `"create child"` というメッセージを2回送ると、2回目に例外が発生します。

名前を指定しないでアクターを作った場合は、`$a`, `$b` という名前になります。アクターパスを確認したいときは、`actor.self.path.toString` で確認できます。


### リモートノード上にアクターを作る

リモートノード上にアクターを作るには2つ方法があり、リモートノードについての情報を `application.conf` に記述する方法と、ソースコード内に書く方法があります。ここではより柔軟な、ソースコード内に書く方法について説明します。

リモートノードにアクターを作るには、Address オブジェクトを作り、それを使ってアクターを作ることになります。

(アクターを作られる側のノード port: 2552)
```scala
class Server extends App {
  ActorSystem("server")
}
```

(アクターを作る側のノード port: 2553)
```scala
import akka.actor._
import akka.remote.RemoteScope

class Server extends App {
  val system = ActorSystem("client")
  val address = AddressFromURIString("akka.tcp://server@127.0.0.1:2552")
  val server = system.actorOf(
    Props[ServerActor].withDeploy(Deploy(scope = RemoteScope(address))),
    name = "server"
  )
  val client = system.actorOf(
    Props(classOf[ClientActor], server),
    name = "client"
  )
  client ! "start"
}

class ServerActor extends Actor {
  def receive = {
    case "ping" => {
      println("Server received ping") // これはアクターを作られた側のノードで出力される
      sender ! "pong"
    }
  }
}

class ClientActor(server: ActorRef) extends Actor {
  def receive = {
    case "start" => server ! "ping"
    case "pong" => {
      println("Client received pong")
      Thread.sleep(1000) // 1秒スリープ
      sender ! "ping"
    }
  }
}
```

これを、別々に起動すると、それぞれの標準出力に、Server received ping と Client received pong が出力されます。
また、この server アクターを取得するときは、`akka.tcp://client@127.0.0.1:2553/user/server` を使います (`akka.tcp://server@127.0.0.1:2552/user/server` ではないことに注意)

アクターパスには論理パスと物理パスがあり、この場合は論理パスは `akka.tcp://client@127.0.0.1:2553/user/server` ですが、物理パスは `akka.tcp://server@127.0.0.1:2552/remote/akka.tcp/client@127.0.0.1:2553/user/server` になります。[参照](http://doc.akka.io/docs/akka/2.3.4/general/addressing.html#logical-actor-paths)

物理的には作られた側のノードにありますが、論理的には作った側のノードにあり、自ノードにアクターを作る場合と全く同じように扱います。
よって、作った側のアクターシステムが落ちた場合は、作られたアクターも落ちることになります。


### Serialization

String や Int 型は特に Serialize/Deserialize を定義しなくてもよかったが、ユーザー定義の型を用いる場合は Serialize/Deserialize も定義する必要がある。

(同じ定義かつグローバルに定義してあるものならOK?)
(TODO)



クラスタの構築
--------------

- http://www.slideshare.net/jboner/the-road-to-akka-cluster-and-beyond
Typesafe の CTO である Jonas Boner さんのプレゼン資料

Akka のクラスタは2400ノードの実例がある

以下の特徴があります。

- Dynamo スタイルの中央サーバのないP2P
- ゴシッププロトコル
- ベクタークロック


### ゴシッププロトコル

(TODO)

### ベクタークロック

http://funini.com/kei/logos/clock.shtml がわかりやすいです。


### Failure Detector


下の論文に基づいたアルゴリズムを使っています。(第一著者は日本人！)

- http://ddg.jaist.ac.jp/pub/HDY+04.pdf

あるノードがクラスタ内の他のノードから到達不可能になった場合、それを発見すること。


### Seed

設定ファイル内で、akka.cluster.seed-nodes が指定されていると、そこにはじめに通信しに行き、クラスタに入れてもらいます。
複数の seed が指定されている場合は、上から順番に通信しにいき、クラスタにいれてもらえればそれで終了、それ以降の seed には通信しに行きません。

例えば、

- A, B, C というノード(クラスタ)の akka-uri がこの順で akka.cluster.seed-nodes に指定されている
- A ノードはなんらかの故障が発生していて通信できない状態
- B, C ノードは正常な状態であったとする
- B, C ノードは同一クラスタではなく、それぞれのクラスタの一員

であるとします。

この状態でノードを起動すると、まず A ノードに通信しようとするが失敗し、次に B ノードに通信し、B ノードのクラスタに入れてもらいます。クラスタに入れてもらったので、C ノードとは通信はしません。
よって B ノードのクラスタのみに入ることになります。


### CurrentClusterState

unreachable のものも cluster.state.members に、status = Up の状態で入っているのが謎
Unreachable Event が発生した時にそいつをはずす？


### 実際に

remote の例で使った application.conf に、provider を `akka.cluster.ClusterActorRefProvider` に変更、cluster ディレクティブを追加します。

seed-nodes も通常は application.conf に追加する。

Listener を作っておけば、勝手にクラスタに関するあれこれをやってくれる。

Listener
```scala
class HogeListener extends Actor {

  def receive = {
    case MemberUp(member) => ... // クラスタにメンバーが追加されたときこのメッセージが

```


### Docker を使ってクラスタを構築する

参照: http://blog.michaelhamrah.com/2014/06/akka-clustering-with-sbt-docker-and-sbt-native-packager/

docker は、link として指定した

```sh
% docker run -i -t --name myseed hamrah.com/clustering
```

とし、

```sh
% docker run -i -t --name member1 --link myseed:seed hamrah.com/clustering
```

とすると、member1 という名前を付けられたコンテナの環境変数に、
myseed という名前が付けられたコンテナのIPアドレスやポート番号などが入る。

```
SEED_NAME=/c3/hogeseed
SEED_PORT_1600_TCP_ADDR=172.17.0.18
SEED_PORT_1600_TCP=tcp://172.17.0.18:1600
SEED_PORT_1600_TCP_PORT=1600
SEED_PORT=tcp://172.17.0.18:1600
SEED_PORT_1600_TCP_PROTO=tcp
```

この例は、これを用いて、seed を特定している。

seed は、


### ClusterClient

[サンプルコード](https://github.com/amutake/actor-examples/tree/master/akka/cluster/cluster-client)

ClusterClient を使うと、Cluster のノードに対して外部からメッセージを送ることができるようになります。

```
libraryDependencies ++= (
  "akka.typesafe.com" %% "akka-contrib" % "2.3.4"
)
```
が必要

(cluster)
```
akka {
  extensions = ["akka.contrib.pattern.ClusterReceptionistExtension"]
}
```

(client)
```
akka {
  provider = "akka.remote.RemoteActorRefProvider"
  # または provider = "akka.cluster.ClusterActorRefProvider"
  remote {
    netty.tcp {
      hostname = "127.0.0.1"
      port = 0
    }
  }
}
```

```scala
val initialContacts = Set(
  system.actorSelection("akka.tcp://cluster@127.0.0.1:2552/user/receptionist"),
  system.actorSelection("akka.tcp://cluster@127.0.0.1:2553/user/receptionist")
)
val c = system.actorOf(ClusterClient.props(initialContacts), "client")

c ! ClusterClient.Send("/user/worker", "hello worker")
c ! ClusterClient.SendToAll("/user/worker", "hello workers")
c ! ClusterClient.Publish("?", "hello")
```

- ClusterClient.Send は、クラスタ内のどれかのノードの、"/user/worker" というアクターにメッセージを送ります。
  - はじめは initialContacts で指定したノードのどれか(system.actorOf でアクターを作る時点で決まる)の、"/user/worker" にメッセージを送ります。
  - どのノードにメッセージを送るかというのは指定できません。
  - 一度どれかの receptionist に繋げば、その receptionist が落ちても、クラスタ内の他のノードに送られるようになります。なのでクラスタ内の全てのノードが落ちない限り、クラスタ内のどれかにはメッセージが届きます。
- ClusterClient.SendToAll は、クラスタの全てのノードの "/user/worker" にメッセージを送ります。
- ClusterClient.Publish もクラスタの全てのノードにメッセージを送るようですが、第一引数に何を指定するのかわかりませんでした。topic を指定する、となっていますが……。


これを使うと、worker のクラスタに対してタスクを投げるみたいなこともできるかもしれません。

フォールトトレラントシステムの構築
----------------------------------

http://doc.akka.io/docs/akka/snapshot/scala/fault-tolerance.html

Akka には、あるアクターが落ちてもシステム全体としては正常に動きつづけるための機能が備わっています。この節では、正常に動き続けるためにどのような機能があり、どのように使うかの説明をします。


### フォールトトレランスとは

耐障害性などと訳されます。
システムの一部が故障していても、システム全体として正常な動作を保ち続ける性質のことです。分散システムにおいては、あるノードやプロセスが落ちてもシステム全体は落ちずに正常に動き続けるという性質です。


### アクターの監視

Akka (やアクターモデルをサポートするその他の言語・ライブラリ) は、アクターの状態を別のアクターが監視することで、耐障害性を実現しています。


#### 監視戦略

アクターがアクターを生成するとき、生成した側のアクターを親アクター、生成された側のアクターを子アクターということにします。

子アクターが例外を投げたときそれをどのようにハンドリングするかを、親アクターのフィールド `supervisorStrategy` に定義します。

例:
```scala
class SupervisorExample extends Actor {
  ...
  override val supervisorStrategy = OneForOneStrategy() {
    case _: NullPointerException => Restart
    case _: Exception => Stop
  }
  ...
}
```

この場合だと、`SupervisorExample` の子アクターが `NullPointerException` を投げた場合は再起動、その他の例外が投げられた場合は停止させます。

この `Restart`, `Stop` のような、例外を吐いた子アクターに対してどのような処置をするか、ということを Directive といいます。Directive は4つあります。

 Directive           | 説明
 ------------------- | ----------------------
 Resume              | アクターを落ちる直前の状態にする
 Restart             | アクターを再起動させる
 Stop                | アクターを停止させる
 Escalate            | 例外を、さらに親のアクターまで伝搬させる


また、監視戦略の種類も2つあります。

 Strategy            | 説明
 ------------------- | -------------------
 OneForOneStrategy   | いずれかの子アクターが例外を投げた場合、その子アクターのみに対して処置を行う
 AllForOneStrategy   | いずれかの子アクターが例外を投げた場合、全ての子アクターに対して処置を行う



### リモートアクターの監視

通常の例外に対する監視は、リモートノードにアクターをデプロイすることで実現できます。

ただ、そのノードを SIGINT で落としてみたところ、エラーハンドリングはできませんでした。
supervisor が対応しているものは例外だけで、JVM がいきなり落ちて通信出来なくなるようなものには対応していないようです。

このような、リモートとの接続が切れて通信出来なくなるような障害は、クラスタの構築の節を参照してください。


### 自分の子ではないアクターの監視

おそらく不可能です。自分の子しか監視できません。

おそらくというのは、調べた限りでは、自分の子ではないアクターを監視するような記述がドキュメントにはなかった、という意味です。


### 限界

ここまでどのようにフォールトトレランスを実現するかということについて説明してきましたが、ここでは、どのようなことには対応できないか、について説明します。

(TODO)

### 分散のためのパターン

きちんと考えてコードを書かないと、分散させてもフォールトトレランスを実現できません。分散すればただそれだけで自動で耐障害性を持たせられるということはありません。

(TODO)

### フォールトの例とそのハンドリング

#### 通常の例外

NullPointerException や ArithmeticException など、ソフトウェアに起因する例外の場合です。
これは、上で説明したように Supervisor となるアクターを作っておき、そのアクターにエラーハンドリングをさせるということになります。

#### 応答なし

リモートノードとの接続が切れてしまった場合です。

(TODO)

CAP 定理
--------

- [Wikipedia - CAP定理](http://ja.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86)
- すごいErlangゆかいに学ぼう！- 26.4節
- http://yohei-y.blogspot.jp/2009/03/capcacidc.html
- http://www.slideshare.net/takahikosato/riak-24796781

CAP とは、一貫性 (**C**onsisteuncy)・可用性 (**A**vailability)・分断耐性 (**P**artition Tolerance) の3つの頭文字をとったものです。
CAP 定理とは、一貫性、可用性、分断耐性について、全てを同時に満たすことはできないという定理です(数学的に証明されています)。

### 一貫性

(TODO)

#### 結果整合性 (Eventual Consistency)

厳密な一貫性ではなく、結果的に一貫性が保たれればよいという考え方です。

### 可用性

(TODO)

### 分断耐性

ネットワークが分断してノード間で通信ができなくなっても、システムが動作し続けられるという性質です。
(TODO)


### 例

Basho 社が開発している分散KVSの Riak は、可用性と分断耐性を満たしています。(強い)一貫性は満たしていませんが、結果整合性は満たしています。

テスト
------

AkkaSpec という Akka のためのテストフレームワークがあります。




ベンチマーク
------------

[Cloud Haskell の調査内のベンチマーク](https://github.com/worksap-ate/demo/blob/master/Cloud-Haskell.md#%E8%BF%BD%E8%A8%98-2014-06-19)を参照してください

その他
------

### メッセージ欠落

参照: http://doc.akka.io/docs/akka/snapshot/general/message-delivery-reliability.html

Akka はメッセージが必ず送られるということは保証しません。メッセージ欠落が起こるかもしれません。
ピュアなアクターモデルでは公平性が保証されています。これは「いつかは必ずメッセージが受け取られる」という性質です。

Akka ではこれを at-most-once delivery と言っていて、あるメッセージは、0回(メッセージ欠落)または1回(正常)送られます。


### メッセージが受け取られる順番

参照: http://doc.akka.io/docs/akka/snapshot/general/message-delivery-reliability.html

Akka では、メッセージの順番は保存されます(ピュアなアクターモデルでは順番については公平性以外はなにも言っていません)。

つまり、アクターAがアクターBにメッセージM1,M2,M3をこの順番で送ったとき、アクターBは必ずM1,M2,M3の順番でメッセージを受け取るということです。
ただし、メッセージ欠落が起こるとM1,M3のような順番になることもあります。

1. アクターAがアクターBにメッセージM1を送る
2. アクターAがアクターCにメッセージM2を送る
3. アクターBがメッセージM1を受け取り、アクターCにそのまま送る

こういう状況のときは、アクターCはM1,M2の順番で受け取ることもあるし、M2,M1の順番で受け取ることもあります。



Pickling and Spores
-------------------

クロージャのシリアライズ/デシリアライズのためのライブラリ。
分散システムにおいて、クロージャ(関数)を渡せるようになる？

EPFL (Odersky 先生がいるところ) と Typesafe の人が開発しています。

これを使って、関数として定義されたタスクを別のノードに渡して実行させる、とかもできるようになるかもしれません。

Pickling のほうは 0.9 まで出ていて、普通に使えそうだけど、Pickling のみではクロージャのシリアライズはできない？
Spores の方はまだ開発は全然進んでいない

参照
- http://lampwww.epfl.ch/~hmiller/pickling/
  - (おそらく) 公式ページ
- http://infoscience.epfl.ch/record/187787/files/oopsla-pickling_1.pdf
  - 論文 (PDF)
  - OOPSLA(2013) というオブジェクト指向のトップカンファレンスに通った
- https://speakerdeck.com/heathermiller/spores-distributable-functions-in-scala
  - Strange Loop 2013 でのスライド
- http://docs.scala-lang.org/sips/pending/spores.html
  - Spores についての草案
- https://github.com/scala/pickling
  - pickling の リポジトリ


参考文献
--------

- [Actors: A Model of Concurrent Computation in Distributed Systems](http://dspace.mit.edu/handle/1721.1/6952)
- [Akka docs](http://doc.akka.io/docs/akka/2.3.4/scala.html)
- [すごいErlangゆかいに学ぼう！](http://estore.ohmsha.co.jp/titles/978427406912P)
- [The Road to Akka Cluster and Beyond](http://www.slideshare.net/jboner/the-road-to-akka-cluster-and-beyond)
- [Basho: Why Vector Clocks Are Easy](http://basho.com/why-vector-clocks-are-easy/)
- [Basho: Why Vector Clocks Are Hard](http://basho.com/why-vector-clocks-are-hard/)
- [ベクタークロック key@sodan](http://funini.com/kei/logos/clock.shtml)
- [Akka Clustering with SBT-Docker and SBT-Native-Packager](http://blog.michaelhamrah.com/2014/06/akka-clustering-with-sbt-docker-and-sbt-native-packager/)
