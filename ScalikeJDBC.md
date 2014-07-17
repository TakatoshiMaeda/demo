基本的にSQLをScalaのコードに効率的に・Type Safeに（何に関するType Safetyかは要調査）記述することを目標としたライブラリであるらしい。そのため、githubにあるFirst exampleも、InterpolationでSQLコードを直接埋めこんだコードになっている。
Query DSLもバージョン1.6.0から導入されている。

公式: [ScalikeJDBC](http://scalikejdbc.org/)

## SQL Interpolation

参考： [SQLInterpolation - ScalikeJDBC](http://scalikejdbc.org/documentation/sql-interpolation.html)

`sql" ... "`のようにInterpolationでSQL文を埋めこめる。

## SQLSyntaxSupport(テーブル定義に対応)

参考(公式ドキュメントに近い記述)： [SQLSyntaxSupport による DRY な SQL ライフ - seratch's weblog in Japanese](http://seratch.hatenablog.jp/entry/2013/02/28/220754)

トレイト`SQLSyntaxSupport[A]`をmixinすることで、テーブルに対応するデータ型を定義できる。

### 例
**テーブル**
```
mysql> select * from GroupId;
+---------+------+
| name    | gid  |
+---------+------+
| Alice   |    0 |
| Carol   |    0 |
| Dave    |    1 |
| Frank   |    1 |
| Mallory | 9999 |
+---------+------+
5 rows in set (0.00 sec)
```
**テーブル定義**
```scala
import scalikejdbc._, SQLInterpolation._

case class GroupId(name: String, gid: Int)

object GroupId extends SQLSyntaxSupport[GroupId] {
  // 対応させるテーブル名を指定
  // （デフォルトの名前は「クラス名にCamelCase -> snake_caseの変換を噛ませた文字列」になっている。）
  override val tableName = "GroupId"
  
  def apply(results: ResultName[GroupId])(wrs: WrappedResultSet) =
    new GroupId(wrs.string(results.name), wrs.int(results.gid))
}
```
**使用する側(クエリ部分の詳細は後述)**
```scala
import scalikejdbc._, SQLInterpolation._

object Test {
  def main: Unit = {
    // initialize JDBC driver & connection pool
    Class.forName("com.mysql.jdbc.Driver")
    ConnectionPool.singleton("jdbc:mysql://localhost/scalikeJDBCTest", "tester", "")

    // ad-hoc session provider on the REPL
    implicit val session = AutoSession

    val g = GroupId.syntax("g")
    val allUsers = sql"""
      select ${g.result.*} from ${GroupId as g} where ${g.gid} < 10
     """.map(GroupId(g.resultName)).list.apply

    println(allUsers)
  }
}
```

**実行結果**
```
List(GroupId(Alice,0), GroupId(Carol,0), GroupId(Dave,1), GroupId(Frank,1))
```

### 上の例のクエリ部について

```scala
    val g = GroupId.syntax("grp")
    val allUsers = sql"""
      select ${g.result.*} from ${GroupId as g} where ${g.gid} < 10
     """.map(GroupId(g.resultName)).list.apply()
```

`sql""" 〜 """` にSQL文を埋めこんでいる。

`g.result.*` と書くと、

```
grp.name as n_on_grp, grp.gid as g_on_grp
```

のように変換される。
その他、展開ルールは
（`g = GroupId.syntax("grp")`のとき）

| 展開前 | 展開後 |
|:-----------:|:------------:|
| `g.result.*`       | `grp.name as n_on_grp, grp.gid as g_on_grp` |
| `GroupId as g`, `GroupId.as(g)`     | `GroupId grp`                               |
| `g.gid`            | `grp.gid`                                   |
| `g.resultName.*`   | `n_on_grp, g_on_grp`                        |
| `g.resultName.gid` | `g_on_grp`                                  |
| `GroupId.column.name` | `name` |

のようになっている。

(この辺混乱しやすい上、エラー箇所がエラーメッセージだけではわかりづらい。上のテストコードを通すのにもかなり苦労した。)

Interpolation部分の型は`scalikejdbc.SQL[Nothing,scalikejdbc.NoExtractor]`となっているので、」
ここからデータを取り出すには`map`を用いて定義したテーブル型にマップしたのち、**Query API**を用いる。

| API | 意味 |
|:---:|:---:|
| `single` | 0個または1個の結果に対し、0個ならNoneを返し, 1個ならその値をSomeに包んで返す。 |
| `first` | 0個以上の結果に対し、0個ならNoneを返し, 1個以上なら最初の値をSomeに包んで返す。 |
| `list` | 結果のリストを返す。 |
| `foreach` | 値は返さず、引数として渡した関数を結果の各要素に適用する。 |

なお、`single`を2個以上の結果に適用すると`TooManyRowsException`を吐く。
`first`も`Option`なので`single`を使う意味はなさそう？

## 値の取り出し

`withSQL`で囲んだSQLクエリの結果は、`WrappedResultSet`型で返ってくる。
この「結果」型の値から値を取り出すには、`string`、`int`などの取り出したい値の型に対応するrow parserを利用する。
また、**`NULL`であるような型をパースする為のメソッドが、`stringOpt`、`intOpt`など`...Opt`という名前で利用できる。**

値を取り出すためのメソッドは[WrappedResultSet.scala](https://github.com/scalikejdbc/scalikejdbc/blob/master/scalikejdbc-library/src/main/scala/scalikejdbc/WrappedResultSet.scala)に全部定義されている。（はず）


## Query DSL
書けるならこちらで書いたほうが安全なはず。

SQLを直書きする代わりに

### 例
上のコードのクエリと同じ挙動をするコードをQuery DSLで書くと
```scala
import scalikejdbc._, SQLInterpolation._

object Test {
  def main: Unit = {
    // initialize JDBC driver & connection pool
    Class.forName("com.mysql.jdbc.Driver")
    ConnectionPool.singleton("jdbc:mysql://localhost/scalikeJDBCTest", "tester", "")

    // ad-hoc session provider on the REPL
    implicit val session = AutoSession

    val g = GroupId.syntax("grp")
    val allUsers = withSQL {
      select.from(GroupId as g).where.lt(g.gid, 10)
    }.map(GroupId(g.resultName)).list.apply
    
    println(allUsers)
  }
}
```

となる。

Query DSLのリファレンスは[ここ](http://scalikejdbc.org/documentation/query-dsl.html)で参照可能。

### 注意
`withsql {...}`文の中には複数のクエリが書ける（たとえばまとめて複数のテーブルにINSERTできる）と思いきや、
最後のひとつしか実行されないことがあるようなので注意が必要。

どのクエリでもそうなっているのかは要検証だが、`delete`文を4つ並べたところ最後の`delete`しか効いていないことを確認済み。



以下に代表的なクエリについてまとめる。

### CREATE
調べた限り、CREATEに対応するQuery DSLはない。
ので、SQL Interpolationで書くしかない。

### INSERT
素直なインタフェース。他のAPIにも言えることだが、
Slickとは違い語順（？）がほぼSQL文そのままなので、SQLに慣れている人にとってはより直感的でわかりやすいかもしれない。

ちょっと注意すべきはQuery API。`map`、`single`などを使う代わりに`update`を使うだけ。
```scala
    withSQL {
      insert.into(UserId).values("Eve", 666)
    }.update.apply
```

### DELETE
これもそのまま。
```scala
    withSQL {
      delete.from(UserId).where.eq(UserId.column.name, "Eve")
    }.update.apply
```

### UPDATE
これもSQLの語順そのままのインタフェースで実現されている。
`set`は可変長引数でいくつでも更新する要素を指定可能。
```scala
    withSQL {
      update(UserId).set(
          UserId.column.name -> "Evan",
          UserId.column.uid -> "777"
          ).where.eq(UserId.column.name, "Eve")
    }.update.apply
```

UPDATEで更新するテーブルにはasで別名を付けられるので
```scala
    withSQL {
      update(UserId as u).set(
          u.name -> "Evan",
          u.uid -> "777"
          ).where.eq(u.name, "Eve")
    }.update.apply
```
でも同じ。

### SELECT
上のコード例にも書いた通り、

```scala
val allUsers = withSQL {
  select.from(GroupId as g).where.lt(g.gid, 10)
}.map(GroupId(g.resultName)).list.apply
```

Query APIとして`apply`でなく`foreach`にすれば、テーブルの各要素に対する操作が書ける。
(この辺はテーブル自体をシーケンスのように扱えたSlickより若干面倒かもしれません。)

### JOIN
コード例

・テーブル定義

```scala
import scalikejdbc._, SQLInterpolation._

case class GroupId(name: String, gid: Int)
case class UserId(name: String, uid: Int)

object GroupId extends SQLSyntaxSupport[GroupId] {
  override val tableName = "GroupId"
  
  def apply(results: ResultName[GroupId])(wrs: WrappedResultSet) =
    new GroupId(wrs.string(results.name), wrs.int(results.gid))
}


object UserId extends SQLSyntaxSupport[UserId] {
  override val tableName = "UserId"
  
  def apply(results: ResultName[UserId])(wrs: WrappedResultSet) =
    new UserId(wrs.string(results.name), wrs.int(results.uid))
}
```

・2つのテーブルを{left, right, inner} joinするコード

```scala
import scalikejdbc._, SQLInterpolation._

object JoinTest {
  def main: Unit = {
    // initialize JDBC driver & connection pool
    Class.forName("com.mysql.jdbc.Driver")
    ConnectionPool.singleton("jdbc:mysql://localhost/scalikeJDBCTest", "tester", "")

    // ad-hoc session provider on the REPL
    implicit val session = AutoSession

    val (g, u) = (GroupId.syntax("grp"), UserId.syntax("usr"))
    
    withSQL {
      select.from(UserId as u)
    }.foreach{
      rs =>
        print(rs.string(u.resultName.name) + "\t|")
        println(rs.int(u.resultName.uid))
    }
    println()
    
    withSQL {
      select.from(GroupId as g)
    }.foreach{
      rs =>
        print(rs.string(g.resultName.name) + "\t|")
        println(rs.int(g.resultName.gid))
    }
    println()
    
    withSQL {
      select.from(UserId as u).leftJoin(GroupId as g).on(u.name, g.name)
    }.foreach{
      rs =>
        print(rs.string(u.resultName.name) + "\t|")
        print(rs.intOpt(u.resultName.uid) + "     \t|")
        println(rs.intOpt(g.resultName.gid))
    }
    println()
    
    withSQL {
      select.from(UserId as u).rightJoin(GroupId as g).on(u.name, g.name)
    }.foreach{
      rs =>
        print(rs.string(g.resultName.name) + "\t|")
        print(rs.intOpt(u.resultName.uid) + "     \t|")
        println(rs.intOpt(g.resultName.gid))
    }
    println()
    
    withSQL {
      select.from(UserId as u).innerJoin(GroupId as g).on(u.name, g.name)
    }.foreach{
      rs =>
        print(rs.string(u.resultName.name) + "\t|")
        print(rs.intOpt(u.resultName.uid) + "     \t|")
        println(rs.intOpt(g.resultName.gid))
    }    
  }
}
```

・実行結果

```
Alice	|0
Bob	|1
Carol	|2
Dave	|3
Ellen	|4
Frank	|5

Alice	|0
Carol	|0
Dave	|1
Frank	|1
Mallory	|9999

Alice	|Some(0)     	|Some(0)
Bob	|Some(1)     	|None
Carol	|Some(2)     	|Some(0)
Dave	|Some(3)     	|Some(1)
Ellen	|Some(4)     	|None
Frank	|Some(5)     	|Some(1)

Alice	|Some(0)     	|Some(0)
Carol	|Some(2)     	|Some(0)
Dave	|Some(3)     	|Some(1)
Frank	|Some(5)     	|Some(1)
Mallory	|None     	|Some(9999)

Alice	|Some(0)     	|Some(0)
Carol	|Some(2)     	|Some(0)
Dave	|Some(3)     	|Some(1)
Frank	|Some(5)     	|Some(1)

```

基本的には`select.from(...).left(|right|inner)join(...).on(...)`という、これまたSQL文に近いインターフェースで使える。
結果は`select`と同じように取り出せばよい。

ただし、`NULL`があった場合、`NULL`であるような結果にアクセスしようとすると`NullPointerException`になるので、`NULL`になりうるデータは`stringOpt`や`intOpt`などの、`...Opt`系アクセサで値を取り出す必要がある。

## Reverse Engeneering(コード生成)

参考: [http://scalikejdbc.org/documentation/reverse-engineering.html](http://scalikejdbc.org/documentation/reverse-engineering.html)

既存のデータベースからコードを自動生成する機能がある。
これは、最初にコード生成機能をセットアップしておけば、**`sbt "scalikejdbc-gen (テーブル名)"`コマンドでテーブルに対応するobjectを生成できる**というもの。

### 準備
`build.sbt`に
```
scalikejdbcSettings
```

`project/plugins.sbt`に
```
# 使用するDBのドライバ
libraryDependencies += "mysql" % "mysql-connector-java" % "5.1.28"

addSbtPlugin("org.scalikejdbc" %% "scalikejdbc-mapper-generator" % "[1.7,)")
```
をそれぞれ追記した後、`project/scalikejdbc.properties`というファイルを作り、各種設定をする。
```
jdbc.driver=com.mysql.jdbc.Driver # 使用するDBのドライバ
jdbc.url=jdbc:mysql://localhost/scalikeJDBCTest # DBのURL
jdbc.username=**** # ユーザ名
jdbc.password=******** # パスワード
jdbc.schema=
generator.packageName=models
# generator.lineBreak: LF/CRLF
geneartor.lineBreak=LF
# generator.template: basic/namedParameters/executable/interpolation/queryDsl
generator.template=queryDsl
# generator.testTemplate: specs2unit/specs2acceptance/ScalaTestFlatSpec
generator.testTemplate=specs2unit
generator.encoding=UTF-8
```

以上が終わったら以下のコマンド
```
sbt "scalikejdbc-gen (テーブル名)"
```
でテーブルを生成する。
テーブル定義ファイルは`generator.packageName`で定義したパッケージに`(テーブル名).scala`という名前で吐かれる。

注)テーブル名`UserId`だと吐かれるファイル名が`Userid.scala`になっていた。キャメルケース・スネークケースに対する名前の変換があるようなので、それっぽい名前で吐かれるということで。

また、テストファイルも吐かれるようだがまだ未調査。

### コード例

吐かれるコードの例（JOINのコード例で使用したUserIdで自動生成した結果）

```scala
package models

import scalikejdbc._
import scalikejdbc.SQLInterpolation._

case class Userid(
  name: String, 
  uid: Int) {

  def save()(implicit session: DBSession = Userid.autoSession): Userid = Userid.save(this)(session)

  def destroy()(implicit session: DBSession = Userid.autoSession): Unit = Userid.destroy(this)(session)

}
      

object Userid extends SQLSyntaxSupport[Userid] {

  override val tableName = "UserId"

  override val columns = Seq("name", "uid")

  def apply(u: ResultName[Userid])(rs: WrappedResultSet): Userid = new Userid(
    name = rs.string(u.name),
    uid = rs.int(u.uid)
  )
      
  val u = Userid.syntax("u")

  override val autoSession = AutoSession

  def find(name: String, uid: Int)(implicit session: DBSession = autoSession): Option[Userid] = {
    withSQL { 
      select.from(Userid as u).where.eq(u.name, name).and.eq(u.uid, uid)
    }.map(Userid(u.resultName)).single.apply()
  }
          
  def findAll()(implicit session: DBSession = autoSession): List[Userid] = {
    withSQL(select.from(Userid as u)).map(Userid(u.resultName)).list.apply()
  }
          
  def countAll()(implicit session: DBSession = autoSession): Long = {
    withSQL(select(sqls"count(1)").from(Userid as u)).map(rs => rs.long(1)).single.apply().get
  }
          
  def findAllBy(where: SQLSyntax)(implicit session: DBSession = autoSession): List[Userid] = {
    withSQL { 
      select.from(Userid as u).where.append(sqls"${where}")
    }.map(Userid(u.resultName)).list.apply()
  }
      
  def countBy(where: SQLSyntax)(implicit session: DBSession = autoSession): Long = {
    withSQL { 
      select(sqls"count(1)").from(Userid as u).where.append(sqls"${where}")
    }.map(_.long(1)).single.apply().get
  }
      
  def create(
    name: String,
    uid: Int)(implicit session: DBSession = autoSession): Userid = {
    withSQL {
      insert.into(Userid).columns(
        column.name,
        column.uid
      ).values(
        name,
        uid
      )
    }.update.apply()

    Userid(
      name = name,
      uid = uid)
  }

  def save(entity: Userid)(implicit session: DBSession = autoSession): Userid = {
    withSQL { 
      update(Userid).set(
        column.name -> entity.name,
        column.uid -> entity.uid
      ).where.eq(column.name, entity.name).and.eq(column.uid, entity.uid)
    }.update.apply()
    entity 
  }
        
  def destroy(entity: Userid)(implicit session: DBSession = autoSession): Unit = {
    withSQL { delete.from(Userid).where.eq(column.name, entity.name).and.eq(column.uid, entity.uid) }.update.apply()
  }
        
}
```

## ベンチマーク

### 要素数5, 10のテーブルに対するINSERT
Slickに比べ異様に遅い。

```
INSERT 5×256 COLUMNS:	12130msec
INSERT 10×256 COLUMNS:	12245msec
INSERT 5×512 COLUMNS:	24748msec
INSERT 10×512 COLUMNS:	24315msec
```

正確な原因は不明だが、おそらくSlickが256(or 512)個のINSERT文を繋げたクエリ文をSQLに一度に渡しているのに対して、
scalikeJDBCではそういうことができない（はず）。なので、scalikeJDBCのほうは「SQLにクエリを投げてレスポンスを受けとる」処理のオーバーヘッドが256(or 512)回発生しており、その分遅くなっているのではと考えられる。

**[検証してみた。](#slickOneByOne)やはり上の考えであっている模様。**

つまり、一度に多量のINSERTクエリを投げられれば、速度はSlickと比べても遜色ないと考えられる。

### UPDATE
こちらはSlickに比べ遜色ない。

```
UPDATE 128 COLUMNS OF (5×512) TABLE:	6448msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	6189msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	12478msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	12311msec
```

### JOIN
わずかにSlickより速い。
最初が遅いのは仕様？JOIN以外の何らかの処理が行なわれていそう。

```
INNER JOIN OF 5×256 + 5×512:	68msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	8msec
INNER JOIN OF 5×256 + 5×512:	4msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	6msec
INNER JOIN OF 5×256 + 5×512:	3msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	6msec
INNER JOIN OF 5×256 + 5×512:	7msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	5msec
INNER JOIN OF 5×256 + 5×512:	4msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	3msec
```