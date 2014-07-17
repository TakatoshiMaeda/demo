Cloud Haskell とは
------------------

Well-Typed 社が開発している、Haskell 上で Erlang 風のインターフェースを提供する並行・分散処理のためのライブラリ群のこと。

- 公式ページ: http://haskell-distributed.github.io/
- ブログポスト: http://www.well-typed.com/blog/68
- 論文: http://research.microsoft.com/en-us/um/people/simonpj/papers/parallel/remote.pdf

以下のような特徴がある。

- アクターモデルに基づいた並行処理
    - 共有変数がなく、メッセージをやりとりすることでプロセス間のコミュニケーションをとる
- スケーラブル
    - 複数コア、複数マシンで動かすこと前提
- 耐障害性
    - プロセスをモニタリングし、プロセスが死んだときにもう一度起動させるなど

ライブラリの構成
----------------

- [distributed-process](http://hackage.haskell.org/package/distributed-process): プロセスなどの定義と API を提供するライブラリ
- [distributed-process-simplelocalnet](http://hackage.haskell.org/package/distributed-process-simplelocalnet): ローカルネットワークにおける、マスタースレーブモデルの実装を提供するライブラリ
- [network-transport](http://hackage.haskell.org/package/network-transport): トランスポートの API を提供するライブラリ
- [network-transport-tcp](http://hackage.haskell.org/package/network-transport-tcp): TCP の network-transport 実装

他には、

- distributed-process-platform
- distributed-process-azure
- distributed-static
- rank1dynamic
- network-transport-inmemory
- network-transport-composed

などがある(未完成のものや、滅多につかわないものなので説明は省略)


distributed-process
-------------------

[distributed-process](http://hackage.haskell.org/package/distributed-process) は、Cloud Haskell のコアとなるパッケージ。プロセスやノードのデータ型や、コンビネータの定義など。

### Process

distributed-process では、並行に動作して互いにメッセージを送りあうというものを、プロセスと呼んでいる。

基本的な型

- `ProcessId`
    - プロセスのIDを表す型。
- `Process a`
    - Process はモナドになっていて、他のプロセスへのメッセージの送信や、メッセージの受信、プロセスの作成などは、全てこの Process モナドの中で行う。

基本的な関数

- `send :: Serializable a => ProcessId -> a -> Process ()`
    - 指定したプロセスにメッセージを送信する
    - Serializable 型クラス(ByteString にシリアライズ可能であるということを表す型クラス)がついているのは、send はネットワークを介してメッセージを送るものであるため
- `expect :: forall a. Serializable a => Process a`
    - 自分宛のメッセージを一つ受け取る
    - 期待する型ではないメッセージは、メッセージボックスのなかに残される

### Node

プロセスはノードというものの上に存在している。基本的にはホストとポートの組に一対一対応している。

基本的な型

- `NodeId`
    - ノードのID
- `LocalNode`
    - ローカルのノードのことを特別に LocalNode という型にしている

基本的な関数

- `newLocalNode :: Transport -> RemoteTable -> IO LocalNode`
    - 新しくローカルノードをつくる。Transport と RemoteTable については後述
- `forkProcess :: LocalNode -> Process () -> IO ProcessId`
    - プロセスを起動させる
- `runProcess :: LocalNode -> Process () -> IO ()`
    - forkProcess とほとんど同じだが、こちらはプロセスが終了するまで待つ

### Spawning Process

プロセスは新しいプロセスを起動させることができる。これを spawn と呼ぶ。

- `spawn :: NodeId -> Closure (Process ()) -> Process ProcessId`
    - 指定したノードにプロセスの定義を送信し、起動させる。返り値は起動したプロセスのプロセスID
    - Closure (詳細は後述)でプロセスの行動の定義がラッピングされているのは、プロセスモナドはシリアライズ可能ではないため。Closure で包むことによってシリアライズ可能にしている
- `spawnLocal :: Process () -> Process ProcessId`
    - ローカルノードでプロセスを起動させる関数
    - ローカルノードでプロセスを起動させるときは、シリアライズする必要はないので、Closure で包まなくてもよい

### Closure, Serializable

前述したように、`Process ()` 型の値はバイト列に変換することができないが、Closure で包むことによってバイト列に変換することができるようになる。

通常は Template Haskell を使って、`$(mkStaticClosure process_def)` をすると、`process_def :: Process ()` が `Closure (Process ())` になる。
また、プロセスの定義が引数をとる場合は、`$(mkClosure process_def)` とすると、`process_def :: SomeArgment -> Process ()` が、 `SomeArgment -> Closure (Process ())` になる。ただし、mkClosure は2引数以上には対応していないので、2引数以上の場合は uncarry しなければいけない。

Serializable 型クラスはバイト列に変換可能であることを示す型クラス。 binary パッケージの `Binary` クラスと、 `Typeable` クラスのインスタンスになっていれば自動で導出される。

### Registry

プロセスに名前をつけ、登録することができる。特定の名前のプロセスに向けてメッセージを送る、というようなことができるようになる。

- `register :: String -> ProcessId -> Process ()`
    - ローカルノードに、指定したプロセスIDを指す名前を登録する
- `nsend :: Serializable a => String -> a -> Process ()`
    - プロセスIDの代わりに、登録してある名前を使用してメッセージを送る
- `registerRemoteAsync :: NodeId -> String -> ProcessId -> Process ()`
    - 指定したノードに、ローカルノードにあるプロセスを指す名前を登録する
    - RegisterReply 型のメッセージが返信され、登録が成功したか失敗したかを知ることが出来る
- `nsendRemote :: Serializable a => NodeId -> String -> a -> Process ()`
    - 指定したノードに登録されている名前宛てにメッセージを送る


### Log

- `say :: String -> Process ()`
   - "logger" という名前で登録されたプロセスにログメッセージを送る
   - デフォルトでは stderr がロガーになっている


### Monitoring

あるプロセスを別のプロセスからモニタリングし、プロセスが例外などで終了してしまった場合に検知することができる。これによって、プロセスが死んだ場合は即座に再起動させたりすることができる。

- `monitor :: ProcessId -> Process ()`
    - 指定したプロセスのモニタリングを開始する

サンプルコード

```haskell
import Import

slave :: Process ()
slave = do
  liftIO $ threadDelay 500000
  liftIO $ print (1 `div` 0) -- ゼロ割で異常終了する
  say "no exeption happened" -- ここは表示されない
  slave

master :: Int -> Process ()
master 0 = return ()
master n = do
  pid <- spawnLocal slave
  monitor pid
  ProcessMonitorNotification _ _ reason <- expect -- プロセスが死ぬと、 ProcessMonitorNotification 型のメッセージがモニタリングしているプロセスに投げられる
  say $ show reason -- 終了した理由は ProcessMonitorNotification 値コンストラクタの第三引数に
  master (n - 1)

main :: IO ()
main = do
  Right t <- createTransport "localhost" "12345" defaultTCPParameters
  node <- newLocalNode t initRemoteTable
  runProcess node (master 5)
  liftIO $ threadDelay 1000
```

実行結果

```sh
$ runghc monitor.hs
Tue Nov 19 09:03:12 UTC 2013 pid://localhost:12345:0:3: DiedException "divide by zero"
Tue Nov 19 09:03:13 UTC 2013 pid://localhost:12345:0:3: DiedException "divide by zero"
Tue Nov 19 09:03:13 UTC 2013 pid://localhost:12345:0:3: DiedException "divide by zero"
Tue Nov 19 09:03:14 UTC 2013 pid://localhost:12345:0:3: DiedException "divide by zero"
Tue Nov 19 09:03:14 UTC 2013 pid://localhost:12345:0:3: DiedException "divide by zero"
```


### Link

Monitor は一方向にプロセスの死活状態を監視するものだったが、 Link は双方向に監視する。
Monitor のときと同様に、プロセスが死んだときはそのプロセスに link しているプロセスにメッセージ(ProcessLinkException 型)が送られる。


### Example

distributed-process を使ったプログラム例として、Ping サーバを考える。

今回使う、distributed-process の API は、

```haskell
data Process = ...
  deriving (Monad, MonadIO, ...)

data ProcessId = ...
  deriving (Binary, Typeable, Serializable, ...)

data NodeId = ...
  deriving (Binary, Typeable, Serializable, ...)

getSelfPid :: Process ProcessId

getSelfNode :: Process NodeId

spawn :: NodeId -> Closure (Process ()) -> Process ProcessId

send :: Serializable a => ProcessId -> a -> Process ()

expect :: Serializable a => Process a

say :: String -> Process ()
```

(途中)


distributed-process-simplelocalnet
----------------------------------

ローカルネットワーク内の複数マシンをマスタースレーブモデルで動作させる場合に便利なAPIを提供している。

IP マルチキャストを使っているため、ネットワークの設定で UDP を開放する必要がある。

###　Example

(途中)


ベンチマーク
------------

### ring-benchmarks

N 個のプロセスからなるリングを作り、1つのメッセージがリングを T 回回るようにし、実行時間を計るテスト。プロセスの起動にかかる時間と、メッセージの送信にかかる時間を計ることができる。

参考: http://d.hatena.ne.jp/everpeace/20120515/1337081070

Cloud Haskell および Erlang で実行時間を計った。

#### 計測するプログラム

Cloud Haskell 版

```haskell
module Main where

import Control.Concurrent(threadDelay)
import Control.Applicative
import Control.Distributed.Process
import Control.Distributed.Process.Node
import Control.Monad
import Network.Transport.TCP
import System.Environment

main :: IO ()
main = do
  (n : t : []) <- map read <$> getArgs
  Right transport <- createTransport "localhost" "12345" defaultTCPParameters
  localnode <- newLocalNode transport initRemoteTable
  runProcess localnode (master n t)

master :: Int -> Int -> Process ()
master n t = do
  self <- getSelfPid
  rootId <- makeRing n t self
  send rootId ()
  unit <- expect :: Process ()
  liftIO $ print unit

makeRing :: Int -> Int -> ProcessId -> Process ProcessId
makeRing n t mid = do
  pids <- replicateM n $ spawnLocal node
  rootId <- spawnLocal $ root mid t
  send rootId (pids ++ [rootId])
  return rootId

root :: ProcessId -> Int -> Process ()
root mid times = do
  (pid : pids) <- expect
  send pid pids
  [] <- expect :: Process [ProcessId]
  loop pid times
  where
    loop _ 0 = send mid ()
    loop pid t = do
      unit <- expect :: Process ()
      send pid unit
      loop pid (t - 1)

node :: Process ()
node = do
  (pid:pids) <- expect
  send pid pids
  loop pid
  where
    loop pid = do
      unit <- expect :: Process ()
      send pid unit
      loop pid
```

Erlang 版

```erlang
-module(ring_benchmarks).
-export([start/2]).

start(N, T) ->
    Self = self(),
    RootId = spawn(fun() -> ring_root(Self, T) end),
    NodeIds = replicate_process(N, fun ring_node/0),
    RootId ! lists:append(NodeIds, [RootId]),
    RootId ! ok,
    receive
        ok -> io:write(ok)
    end.

replicate_process(0, _) -> [];
replicate_process(N, Fun) ->
    [spawn(fun() -> Fun() end)|replicate_process(N - 1, Fun)].

ring_root(Mid, Times) ->
    receive
        [Pid|Pids] ->
            Pid ! Pids,
            receive
                [] -> root_loop(Mid, Pid, Times)
            end
    end.

root_loop(Mid, _, 0) -> Mid ! ok;
root_loop(Mid, Pid, Times) ->
    receive
        ok ->
            Pid ! ok,
            root_loop(Mid, Pid, Times - 1)
    end.

ring_node() ->
    receive
        [Pid|Pids] ->
            Pid ! Pids,
            node_loop(Pid)
    end.

node_loop(Pid) ->
    receive
        ok ->
            Pid ! ok,
            node_loop(Pid)
    end.
```

#### 計測した環境

環境は以下の通り。

- CPU: Intel Core i3-2120 3.30GHz * 4
- RAM: 4GB
- OS: Ubuntu 12.04 LTS(64bit)

また、各言語でのコンパイル及び実行は以下のように行った。

- Haskell: `ghc -threaded -O2 ring-benchmarks.hs; time ./ring-benchmarks <N> <T> +RTS -N`
- Erlang: `erlc ring_benchmarks.erl; time erl -noshell -eval "ring_benchmarks:start(<N>, <T>)." -s init stop`

実行時間は、zsh の組み込み time コマンドで計った。


#### 計測結果

 (N, T)          | Cloud Haskell | Erlang
 --------------- | ------------- | ------
 (10, 10)        | 0.022         | 1.191
 (100, 10)       | 0.065         | 1.197
 (1000, 10)      | 1.500         | 1.224
 (10000, 10)     | 14:44.67      | 2.092
 (100000, 10)    | 未計測        | 6:47.24
 (10, 100)       | 0.030         | 1.182(0.002)
 (10, 1000)      | 0.341         | 1.184(0.004)
 (10, 10000)     | 3.340         | 1.207(0.027)
 (10, 100000)    | 27.737        | 1.582(0.302)
 (10, 1000000)   | 5:46.48       | 4.878(3.698)
 (10, 10000000)  | 未計測        | 37.646(36.466)
 (100, 100)      | 0.293         | 1.173
 (1000, 1000)    | 3:03.28       | 1.593
 (10000, 10000)  | 未計測        | 1:02.46

Haskell(10000, 10) については、メモリはあまり使わず CPU のパワー不足によって遅くなっており、
Erlang(100000, 10) については、逆に CPU はほとんど使っておらず、メモリ不足により遅くなっていた。

メッセージが回る回数は、Haskell の場合は回数に比例して時間が長くなっていった。

(10, 10) などで、 Haskell より Erlang のほうが時間がかかっているのは、VM の起動と終了によるものだと思われる。

速度の面では Erlang のほうが優秀。


追記 (2014-06-19)
-----------------

Cloud Haskell, Erlang, Akka についてもう一度ベンチマークをとった。

### Ring Benchmarks

Ring Benchmarks は、N 個のアクターをリング状に配置し、メッセージを T 周させるというもの。



#### ソースコード

[使ったソースコード](https://gist.github.com/amutake/c06cd8e647de62748c9e)

#### 計測環境

##### マシン環境

- CPU: Intel Core i3-2120 3.30GHz * 4
- RAM: 4GB
- OS: Ubuntu 12.04 LTS (64bit)

##### コンパイラなどのバージョン

- Haskell
  - ghc-7.8.2
  - base-4.7.0.0, distributed-process-0.5.0, network-transport-tcp-0.4.0
- Erlang
  - Erlang/OTP 17.0
- Scala
  - scala-2.11.0
  - sbt-0.13.1
  - akka-2.3.3

#### 実行方法

```sh
# Haskell
$ ghc -threaded -O2 ring_benchmarks.hs
$ time ./ring_benchmarks N T +RTS -N

# Erlang
# +P は、プロセス数の上限を上げるフラグ。デフォルトは 262144 (man erl を参照)。N >= 26212* のときはこれを有効化する。
$ erlc ring_benchmarks.erl
$ time erl (+P N+1) -noshell -eval "ring_benchmarks:start(N, T)." -s init stop

# Scala
# build.sbt などを用意する
$ sbt compile
$ time sbt 'run N T'
```

#### 計測結果

(N 個のアクター, T 周)

時間表記は 分:秒 (例: 3:45.56 は、3分45秒56 のこと)

"未計測"は、計測に時間がかかるまたはメモリの使いすぎによりマシンがフリーズすることが予想されたもの。
"計測中止"は、計測が終わらなかったり、マシンが重くなりすぎたため途中で終了させたもの。カッコ内はその理由。
"強制終了"は、メモリが足りなくなるなどで OS により強制終了させられた場合。

また、時間にカッコがついているものは、(N, T) = (1, 1) のときにかかった時間を引いたものである。VM の起動と終了や、sbt の起動に時間がかかるため。

 (N, T)             | Cloud Haskell          | Erlang                              | Akka
 ------------------ | -----------------------| ----------------------------------- | ------------------
 (1, 1)             | 0.011                  | 1.188                               | 3.889
 (10, 10)           | 0.013                  | 1.180                               | 3.910
 (100, 10)          | 0.051                  | 1.197                               | 4.030
 (1000, 10)         | 0.304                  | 1.208                               | 4.398
 (10000, 10)        | 12.914                 | 1.322                               | 6.108
 (100000, 10)       | 計測中止 (20分以上)    | 2.308                               | 8.386
 (1000000, 10)      | 未計測                 | 15.834                              | 33.821
 (10000000, 10)     | 未計測                 | 強制終了 (メモリ8GB(スワップ含む)を使いきったため) | 計測中止 (15分以上)
 (10, 100)          | 0.029                  | 1.194                               | 4.129
 (10, 1000)         | 0.183                  | 1.175                               | 4.321
 (10, 10000)        | 1.286                  | 1.264                               | 4.480
 (10, 100000)       | 14.754                 | 1.576                               | 4.890
 (10, 1000000)      | 2:33.58                | 4.786                               | 10.817
 (10, 10000000)     | 未計測                 | 37.221                              | 1:08.24
 (100, 100)         |     | 1.174     | 4.160
 (1000, 1000)       |     | 1.536     | 4.921
 (10000, 10000)     |     | 50.491    | 47.761


#### 簡単な考察

以前の計測では ghc-7.6.3 を使っていたが、今回は ghc-7.8 で行い、数倍〜数十倍は速くなった。すごい。
思ったよりも Akka が Erlang に近い結果を出した。すごい。

Akka (10000000, 10) で、もっとメモリを使うんじゃないかと思ったが、1GB ほどしか使わなかった。デフォルトではメモリ使用量に制限がかかっていて、フラグで上限を変えるとか？
(sbt が java -Xms1024m -Xmx1024m -XX:MaxPermSize=256m -XX:ReservedCodeCacheSize=128m ... というオプションで実行していたので、これっぽい)
もしかしたら Haskell も同じかもしれない。

### Future Work

[C++ でアクターモデルを提供しているライブラリ](https://github.com/Neverlord/libcppa) や、アクターモデルではないが Golang などでも行なってみたい。
