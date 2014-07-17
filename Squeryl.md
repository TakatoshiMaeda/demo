JavaのORMであるJPA. JDO, Hibernateから出発し、各コミュニティにおける様々な問題点に関する議論を踏まえて作られた新しいScala用ORM、とのことらしい。

ライブラリ名の読み方に関しては[拘っていない模様](https://groups.google.com/forum/#!topic/squeryl/IpsjzjTwKGY)。
作者は"Squirrel"っぽく読んでいるらしい。

[公式サイト](http://squeryl.org/)

## DB接続
参考サイト: [公式](http://squeryl.org/sessions-and-tx.html) + (http://www.mwsoft.jp/programming/scala/squeryl.html)
`SessionFactory.concreteFactory`に設定すればあとはよろしくやってくれる。

MySQLで使うなら以下のようになる。

```scala
import org.squeryl.adapters.MySQLAdapter
import org.squeryl.{ SessionFactory, Session }

object TestCode {
  // Connection
  Class.forName("com.mysql.jdbc.Driver")
  SessionFactory.concreteFactory = Some(() =>
    Session.create(
      java.sql.DriverManager.getConnection("jdbc:mysql://localhost/データベース名", "ユーザ名", "パスワード"),
      new MySQLAdapter))

  // ...

}
```

## テーブル定義
色々細かい設定に関しては調査中

クラスを定義したら、`Schema`クラスを継承した`object`の中で`table[クラス名]`でテーブルを定義する。

```scala
import org.squeryl.Schema

case class GroupId(name: String, gid: Int)
case class UserId(name: String, uid: Int)

object Tables extends Schema {
  val groupid = table[GroupId]
  val userid = table[UserId]
  
  // テーブル名(TableName)がクラス名(AlternatingClass)と違う場合は
  // val alternation = table[AlternatingClass]("TableName")
}
```

なお、実際のカラム名とは異なる名前でフィールドを定義したいときは`@Column`アノテーションを使って

```scala
class GroupId(val name: String,
              @Column("GROUP_ID")
              val gid: Int)
```

とする。

## 各種基本クエリ

コード例

(テーブル定義は上のコード)

### コード例
```scala
import org.squeryl.adapters.MySQLAdapter
import org.squeryl.{ SessionFactory, Session }
import org.squeryl.PrimitiveTypeMode._

object Test {
  // Connection
  Class.forName("com.mysql.jdbc.Driver")
  SessionFactory.concreteFactory = Some(() =>
    Session.create(
      java.sql.DriverManager.getConnection("jdbc:mysql://localhost/squerylTest", "tester", ""),
      new MySQLAdapter))

  def main() = {
    val userId = Tables.userid
    
    transaction {
      // DROP, CREATE
      Tables.drop
      Tables.create
      
      // INSERT
      userId.insert(new UserId("Alice", 0))
      userId.insert(new UserId("Bob", 1))
      
      // Iterableの要素も取れる
      userId.insert(List(new UserId("Carol", 2), new UserId("Dave", 3), new UserId("Ellen", 4)))
      
      groupId.insert(List(new GroupId("Alice", 3), new GroupId("Bob", 1), new GroupId("Dave", 2)))

      // SELECT
      println("[0]")
      from(userId)(s => select(s)).foreach {
        case UserId(user, uid) =>
          println(s"User: ${user},\tUserId: ${uid}")
      }
      
      // SELECT + where + orderBy
      println("[0']")
      from(userId, groupId)((us, gs) =>
        where(us.name === gs.name)
        select(gs)
        orderBy(gs.gid.desc)).foreach {
        case GroupId(user, gid) =>
          println(s"User: ${user},\tGroupId: ${gid}")
      }
      
      // UPDATE
      // partially update
      update(userId)(s =>
        where(s.name === "Carol")
        set(s.name := "Charlie")
      )

      // fully update
      update(userId)(s =>
        where(s.name === "Ellen")
        set(s.name := "Eve",
            s.uid := s.uid.~ + 662) // もちろん s.uid := 666 でもOK
      )

      println("[1]")
      from(userId)(s => select(s)).foreach {
        case UserId(user, uid) =>
          println(s"User: ${user},\tUserId: ${uid}")
      }
      
      // DELETE
      userId.deleteWhere(_.name === "Eve")
      
      println("[2]")
      from(userId)(s => select(s)).foreach {
        case UserId(user, uid) =>
          println(s"User: ${user},\tUserId: ${uid}")
      }
    }
  }
}
```

実行結果
```
[0]
User: Alice,	UserId: 0
User: Bob,	UserId: 1
User: Carol,	UserId: 2
User: Dave,	UserId: 3
User: Ellen,	UserId: 4
[0']
User: Alice,	GroupId: 3
User: Dave,	GroupId: 2
User: Bob,	GroupId: 1
[1]
User: Alice,	UserId: 0
User: Bob,	UserId: 1
User: Charlie,	UserId: 2
User: Dave,	UserId: 3
User: Eve,	UserId: 666
[2]
User: Alice,	UserId: 0
User: Bob,	UserId: 1
User: Charlie,	UserId: 2
User: Dave,	UserId: 3
```

### CREATE/DROP
CREATE, DROPはクラスで定義したテーブルのまとまりごとに行われる。

注意すべきは**`create`メソッドで生成されるテーブルのカラム順が制御できない（クラスに定義したフィールド順に対応して決まるわけではない）**という点。
これを制御しようとすると、内部で使用しているメソッド`writeCreateTable`をoverrideしたり、`printDdl`メソッドで吐かれるSQL文を自分で書きかえて手動で発行するとかしないとならないようです。（参考: [https://groups.google.com/forum/#!topic/squeryl/ZwiDf5Q-IUI](https://groups.google.com/forum/#!topic/squeryl/ZwiDf5Q-IUI)）

### INSERT
`insert`メソッドを使う。
`insert`メソッドは要素1つを取ることもできるし、要素のIterableなコレクションを取ることもできる。
(つまり、まとめてINSERTできるのでscalikeJDBCで問題になった「大量の要素をINSERTするのが遅い」問題は起こらない。このことに関して[公式ページに言及(Batched updates and Inserts)](http://squeryl.org/inserts-updates-delete.html)がある。)

### SELECT
`from`を使う。第一引数で参照元テーブル（複数指定可能）を指定し、第二引数で`select`メソッドを使って取ってくる結果を指定する（詳しくは例を参照）。`from`の返り値の型は`Query[<返り値の型>]`であり、これはIterableトレイトをmixinしているので、`map`なり`foreach`なりで加工したり、`toList`でリストに変換してしまえたりする。

2つ目の例のように、`from`の第二引数では`where`で条件を指定しての絞り込みや、`orderBy`での昇順(`asc`)/降順(`desc`)ソートができる。**なお、ここでやっているのはSELECTというよりはJOINに近い。**

** LogicalBooleanについて **

`where`などに指定できる論理式は`Boolean`型ではなく`LogicalBoolean`型を使う。
使用可能な演算子などは[Scaladoc](http://squeryl.org/api/index.html#org.squeryl.dsl.BooleanExpression)を見ればわかるが、基本的に
 * 不等号はBooleanと同じ
 * 等号は`===`

だけ覚えれば事足りるはず。

### UPDATE
`update`メソッドを使う。第一引数に更新したいテーブルを指定し、第二引数では`where`での条件指定や、`set`で具体的な更新内容の決定などを行う。

`set`は可変長引数で、`A := B`という手続き型言語ライクなインタフェースになっている。

2つ目の例の`s.uid := s.uid.~ + 662`は、`s.uid`のフィールドの値に662を加えるという意味である。このように`.~`を付することで「現在の値」を参照できる。

### DELETE
`deleteWhere`メソッドを使う。`(対象のテーブル).deleteWhere(条件式)`で条件式を満たす値を全て削除する。
ここで`deleteWhere`の引数に使えるのも`LogicalBoolean`型の値なので注意。

## JOIN

コード例
```scala
import org.squeryl.adapters.MySQLAdapter
import org.squeryl.{ SessionFactory, Session }
import org.squeryl.PrimitiveTypeMode._

object Join {
  // Connection
  Class.forName("com.mysql.jdbc.Driver")
  SessionFactory.concreteFactory = Some(() =>
    Session.create(
      java.sql.DriverManager.getConnection("jdbc:mysql://localhost/squerylTest", "tester", ""),
      new MySQLAdapter))

  def main() = {
    val userId = Tables.userid
    val groupId = Tables.groupid
    
    transaction {
      // DROP, CREATE
      Tables.drop
      Tables.create
      
      // INSERT
      userId.insert(List(new UserId("Alice", 0), new UserId("Bob", 1), new UserId("Carol", 2), new UserId("Dave", 3), new UserId("Ellen", 4)))
      groupId.insert(List(new GroupId("Alice", 3), new GroupId("Bob", 1), new GroupId("Dave", 2), new GroupId("Frankie", 2)))

      println("[0:U]")
      from(userId)(s => select(s)).foreach {
        case UserId(user, uid) =>
          println(s"User: ${user},\tUserId: ${uid}")
      }
      println("[0:G]")
      from(groupId)(s => select(s)).foreach {
        case GroupId(user, gid) =>
          println(s"User: ${user},\tGroupId: ${gid}")
      }

      // INNER JOIN
      println("[INNER JOIN]")
      from(userId, groupId)((us, gs) =>
        where(us.name === gs.name)
          select ((us.name, us.uid, gs.gid))).foreach {
        case (user, uid, gid) =>
          println(s"User: ${user},\tUserId: ${uid},\tGroupId: ${gid}")
      }

      // LEFT JOIN
      println("[LEFT JOIN]")
      join(userId, groupId.leftOuter)((us, gs) =>
        select((us.name, us.uid, gs.map(_.gid)))
          on (us.name === gs.get.name)).foreach {
        case (user, uid, gid) =>
          println(s"User: ${user},\tUserId: ${uid},\tGroupId: ${gid}")
      }
    }
  }
}
```

実行結果
```
[0:U]
User: Alice,	UserId: 0
User: Bob,	UserId: 1
User: Carol,	UserId: 2
User: Dave,	UserId: 3
User: Ellen,	UserId: 4
[0:G]
User: Alice,	GroupId: 3
User: Bob,	GroupId: 1
User: Dave,	GroupId: 2
User: Frankie,	GroupId: 2
[INNER JOIN]
User: Alice,	UserId: 0,	GroupId: 3
User: Bob,	UserId: 1,	GroupId: 1
User: Dave,	UserId: 3,	GroupId: 2
[LEFT JOIN]
User: Alice,	UserId: 0,	GroupId: Some(3)
User: Bob,	UserId: 1,	GroupId: Some(1)
User: Carol,	UserId: 2,	GroupId: None
User: Dave,	UserId: 3,	GroupId: Some(2)
User: Ellen,	UserId: 4,	GroupId: None
```

上の「各種基本クエリ」の**SELECT**の二つ目の例がINNER JOINに対応する。
つまり、`from`には複数の引数が許されており、複数指定したケースがINNER JOINである。

一方,LEFT JOINは`join`メソッドで行える。
 * **`join(A.leftOuter, B)`でテーブル‘B‘を、
`join(A, B.leftOuter)`でテーブル‘A‘を優先したJOINができる。**
 * `on`で結合する条件を指定できる。ここもやはり`LogicalBoolean`型を使う。
 * `select`で取りだすところは同じ。

注意点として、優先していないほうのテーブルの扱いが挙げられる。

```scala
// LEFT JOIN
join(userId, groupId.leftOuter)((us, gs) =>
    // ...
    on (us.name === gs.get.name)).foreach {
    // ...
}
```

上のコードでは`userId`テーブルを優先したLEFT JOINを行っているので、それに対応する`groupId`テーブルのレコードが存在しないかもしれない。
よって空の値の処理をしなければならないが、
SlickやscalikeJDBCではこの場合**各フィールドが`null`になった**が、一方でSquerylでは**テーブルそのものがOption型**になっており、**レコードが空の場合は`None`が束縛される**。

具体的には、レコード「`name: Carol, uid: 2`」がUserIdテーブルにあって、`name: Carol`であるようなレコードがGroupIdテーブルにない場合、上のコードの`gs`には`None`が束縛される。それ以外、つまり対応するレコードがある場合は、`Some`に包まれたレコードが束縛される。

なので、優先していないほうのテーブルのレコードの型が`Option[<Record Type>]`であることに注意しなければならない。
（onの条件文の中ではこれらに考慮する必要はない。つまり、`gs.get.name`のように値が存在することを前提としたコードを書いてよい。）

`record.field`のような**フィールドへのアクセス**を、`record`が`Option`に包まれた状態でやりたいときは`record.map(_.field)`という書き方が便利。
`record`がSomeのときは`field`の値を`Some`で包んで返し、`None`のときは`None`のままである。

## コード生成
公式ページを見る限り、既存のDBを読み込んでテーブル型などを自動生成する機能はなさそう。

## ベンチマーク

### INSERT
```
INSERT 5×256 COLUMNS:	74msec
INSERT 10×256 COLUMNS:	58msec
INSERT 5×512 COLUMNS:	83msec
INSERT 10×512 COLUMNS:	184msec

INSERT 5×256 COLUMNS:	42msec
INSERT 10×256 COLUMNS:	39msec
INSERT 5×512 COLUMNS:	78msec
INSERT 10×512 COLUMNS:	73msec

INSERT 5×256 COLUMNS:	40msec
INSERT 10×256 COLUMNS:	38msec
INSERT 5×512 COLUMNS:	63msec
INSERT 10×512 COLUMNS:	77msec

INSERT 5×256 COLUMNS:	38msec
INSERT 10×256 COLUMNS:	45msec
INSERT 5×512 COLUMNS:	66msec
INSERT 10×512 COLUMNS:	72msec

INSERT 5×256 COLUMNS:	36msec
INSERT 10×256 COLUMNS:	36msec
INSERT 5×512 COLUMNS:	61msec
INSERT 10×512 COLUMNS:	74msec
```
かなり速い。速さだけ見れば比較しているO/R mapper中最高性能。

### UPDATE
```
UPDATE 128 COLUMNS OF (5×512) TABLE:	252msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	226msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	346msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	386msec

UPDATE 128 COLUMNS OF (5×512) TABLE:	161msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	176msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	324msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	283msec

UPDATE 128 COLUMNS OF (5×512) TABLE:	130msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	160msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	291msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	335msec

UPDATE 128 COLUMNS OF (5×512) TABLE:	139msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	197msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	391msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	532msec

UPDATE 128 COLUMNS OF (5×512) TABLE:	142msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	137msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	246msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	261msec
```
他の二つに比べて段違いに速い。

### JOIN
```
INNER JOIN OF 5×256 + 5×512:	160msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	90msec
INNER JOIN OF 5×256 + 5×512:	41msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	73msec
INNER JOIN OF 5×256 + 5×512:	28msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	62msec
INNER JOIN OF 5×256 + 5×512:	22msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	57msec
INNER JOIN OF 5×256 + 5×512:	27msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	57msec
```
JOINは他の二つに比べて遅い。