インタフェースがわかりやすい。Scalaにおける通常のコレクション(Seqなど)と同じ感覚で使える。

## テーブル定義

### テンプレート

`Table[T]`クラスを継承する。`T`にはカラムの型をタプルで指定する。

```scala
class MyTable(tag: Tag) extends Table[(MyColumn1, ..., MyColumnN)](tag, "MyTableName") {
  def myColumn1 = column[MyColumn1]("MyColumn1Name")
  // ...
  def myColumnN = column[MyColumnN]("MyColumnNName")

  def * = (myColumn1, ..., myColumnN)
}
val myTable = TableQuery[MyTable]
```

## DB接続

```scala
Database.forURL("jdbc:mysql://localhost/slickTest", driver = "com.mysql.jdbc.Driver") withSession {
  implicit session =>
  // ここに各種クエリを記述
}
```

## Basic Queries

### Example code
テーブル定義:
```scala
import scala.slick.driver.MySQLDriver.simple._

class UserIdTable(tag: Tag) extends Table[(String, Int)](tag, "UserId") {
  def name = column[String]("name")
  def uid = column[Int]("uid")
  def * = (name, uid)
}

class GroupIdTable(tag: Tag) extends Table[(String, Int)](tag, "GroupId") {
  def name = column[String]("name")
  def gid = column[Int]("gid")
  def * = (name, gid)
}
```

使用する側:
```scala
import scala.slick.driver.MySQLDriver.simple._

object BasicQuery {
  def main(): Unit = {
    
    //テーブル
    val userId = TableQuery[UserIdTable]
    
    // DB接続
    Database.forURL("jdbc:mysql://localhost/slickTest", driver = "com.mysql.jdbc.Driver") withSession {
      implicit session =>
      try {
        (userId.ddl).drop
      } catch {
        case e: Throwable => println("No UserId Table")
      }
      
      try {
        // CREATE
        (userId.ddl).create
        
        // INSERT
        userId += ("Alice", 0)
        userId += ("Bob",1)
        
        userId ++= Seq(("Carol", 2), ("Dave", 3), ("Ellen", 4))
        
        println("[0]")
        userId.foreach {case (user, uid) => println(s"User: ${user},\tUserId: ${uid}")}
        
        // UPDATE
        userId.where(_.name === "Carol")
              .map(_.name)
              .update("Charlie")
        
        userId.where(_.name === "Ellen")
              .update(("Eve", 666))
        
        println("[1]")
        userId.foreach {case (user, uid) => println(s"User: ${user},\tUserId: ${uid}")}
        
        // DELETE
        userId.where(_.name === "Eve").delete
        
        println("[2]")
        userId.foreach {case (user, uid) => println(s"User: ${user},\tUserId: ${uid}")}
      }
    }
  }
}
```

実行結果：
```
[0]
User: Alice,	UserId: 0
User: Bob,	UserId: 1
User: Carol,	UserId: 2
User: Dave,	UserId: 3
User: Ellen,	UserId: 4
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

### CREATE

`TableQuery[(Table class)]`型の値から`ddl`メソッドによりDDL(Data definition language)オブジェクトを取り出せる。

`(DDLオブジェクト).create`で対応するテーブルを持ったDBを作成できる。

   ```scala
   // CREATE
   (userId.ddl).create
   ```
`++`演算子でDDLオブジェクトを結合できる: `(myTable1.ddl ++ myTable2.ddl).create // myTable1とmyTable2を持つDBを作成`

### INSERT

テーブルへのINSERTはSeq型に対する操作とほぼ同じインタフェースで行える。

```scala
// INSERT
userId += ("Alice", 0)
userId += ("Bob",1)

userId ++= Seq(("Carol", 2), ("Dave", 3), ("Ellen", 4))
```

### WHERE
WHEREによる条件絞り込みはそのまま`where`メソッドを使う。
```scala
userId.where(_.name === "Carol")
```

条件が大きくなったり複数カラムが関係するようならfor文＋if文を使ったほうが綺麗にかけるかもしれない。
```scala
def columns =
  for {
    us <- userId
    if us.uid % 3 === 0 && us.name === "Bob"
  } yield us
```

### UPDATE
基本は

1. `where`を使い、更新するカラムを条件で絞り込む
2. `map`で更新するフィールドのみに絞る
3. `update`メソッドで更新する

```scala
userId.where(_.name === "Carol") // 1. name="Carol"のカラムを指定
      .map(_.name)               // 2. "name"を更新するフィールドに指定
      .update("Charlie")         // 3. nameを"Charlie"に更新
 
userId.where(_.name === "Ellen") // 1. name="Carol"のカラムを指定
      .map(r => (r.name, r.uid)) // 2. "(name, uid)"を更新するフィールドに指定。ここでは要素を絞っていない（全ての要素を更新している）のでこのmapは省略してもよい。
      .update(("Eve", 666))      // 3. nameを"Eve", uidを666に更新
```

複数カラムを同時に更新することもできる。

```scala
// カラムの内容全てを更新（このときは当然、mapでカラムを制限する必要はない）
userId.where(_.name === "Ellen")
      .update(("Eve", 666))
```

### DELETE
`update`メソッド同様、`where`で削除するカラムを指定して`delete`で削除できる。

```scala
userId.where(_.name === "Eve").delete
```

### テーブルに対するイテレーション

 * `myTable.foreach`や`for(elem <- myTable) ...`でテーブルの各要素を取り出して操作できる。(SELECT相当)
 * テーブルの内容を表示する例:

    ```scala
    userId.foreach {case (user, uid) => println(s"User: ${user},\tUserId: ${uid}")}
    ```

## JOIN
(テーブル定義は同じ)
```scala
import scala.slick.driver.MySQLDriver.simple._

object Join {
  def main(): Unit = {
    //テーブル
    val userId = TableQuery[UserIdTable]
    val groupId = TableQuery[GroupIdTable]
    
    // DB接続
    Database.forURL("jdbc:mysql://localhost/slickTest", driver = "com.mysql.jdbc.Driver") withSession {
      implicit session =>
      try {(userId.ddl).drop} catch {case e: Throwable => println("No UserId Table")}
      try {(groupId.ddl).drop} catch {case e: Throwable => println("No GroupId Table")}
      
      try {
        // CREATE
        (userId.ddl ++ groupId.ddl).create // まとめてCREATEできる
        
        userId ++= Seq(("Alice", 0), ("Bob",1), ("Carol", 2), ("Dave", 3), ("Ellen", 4), ("Frank", 5))
        groupId ++= Seq(("Alice", 0), ("Carol", 0), ("Dave", 1), ("Frank", 1), ("Mallory", 9999))
        
        println("[0]")
        println("UserId table")
        userId.foreach {case (user, uid) => println(s"User: ${user},\tUserId: ${uid}")}
        println("GroupId table")
        groupId.foreach {case (user, gid) => println(s"User: ${user},\tGroupId: ${gid}")}
        
        // Inner Join (1)
        val innerjoined1 =
          for {
            usid <- userId
            gpid <- groupId
            if usid.name === gpid.name
          } yield (usid.name, usid.uid, gpid.gid)
        
        println("[1]")
        innerjoined1.foreach {case ((user, uid, gid)) => println(s"User: ${user},\tUserId: ${uid},\tGroupId: ${gid}")}
        
        // Inner Join (2)
        val innerjoined2 =
          for {
            (usid, gpid) <- userId innerJoin groupId on (_.name === _.name)
          } yield (usid.name, usid.uid, gpid.gid)
        
        println("[2]")
        innerjoined2.foreach {case ((user, uid, gid)) => println(s"User: ${user},\tUserId: ${uid},\tGroupId: ${gid}")}
        
        // (Left) Outer Join
        val leftjoined = for {
          (usid, gpid) <- userId leftJoin groupId on (_.name === _.name)
        } yield (usid.name, usid.uid.?, gpid.gid.?) // カラムに ? をつけるとOptionで値を包める（NULLはNoneになる）
        
        println("[3]")
        leftjoined.foreach {case ((user, uid, gid)) => println(s"User: ${user},\tUserId: ${uid},\tGroupId: ${gid}")}
        
      }
    }
  }
}
```

実行結果:
```
[0]
UserId table
User: Alice,	UserId: 0
User: Bob,	UserId: 1
User: Carol,	UserId: 2
User: Dave,	UserId: 3
User: Ellen,	UserId: 4
User: Frank,	UserId: 5
GroupId table
User: Alice,	GroupId: 0
User: Carol,	GroupId: 0
User: Dave,	GroupId: 1
User: Frank,	GroupId: 1
User: Mallory,	GroupId: 9999
[1]
User: Alice,	UserId: 0,	GroupId: 0
User: Carol,	UserId: 2,	GroupId: 0
User: Dave,	UserId: 3,	GroupId: 1
User: Frank,	UserId: 5,	GroupId: 1
[2]
User: Alice,	UserId: 0,	GroupId: 0
User: Carol,	UserId: 2,	GroupId: 0
User: Dave,	UserId: 3,	GroupId: 1
User: Frank,	UserId: 5,	GroupId: 1
[3]
User: Alice,	UserId: Some(0),	GroupId: Some(0)
User: Bob,	UserId: Some(1),	GroupId: None
User: Carol,	UserId: Some(2),	GroupId: Some(0)
User: Dave,	UserId: Some(3),	GroupId: Some(1)
User: Ellen,	UserId: Some(4),	GroupId: None
User: Frank,	UserId: Some(5),	GroupId: Some(1)
```

### INNER JOIN
forを使う。
 1. それぞれから値を取り出して`if`で条件を絞り込む
 2. `innerJoin ~ on ~`で同時にやる
の2通りがある。

```scala
        // Inner Join (if syntax)
        val innerjoined1 =
          for {
            usid <- userId
            gpid <- groupId
            if usid.name === gpid.name
          } yield (usid.name, usid.uid, gpid.gid)
        
        // Inner Join (innerJoin method)
        val innerjoined2 =
          for {
            (usid, gpid) <- userId innerJoin groupId on (_.name === _.name)
          } yield (usid.name, usid.uid, gpid.gid)
```

### Outer Join

`leftJoin`(`rightJoin`)関数を使うのが一番綺麗に書ける（？）

```scala
// (Left) Outer Join
val leftjoined =
  for {
    (usid, gpid) <- userId leftJoin groupId on (_.name === _.name)
  } yield (usid.name, usid.uid.?, gpid.gid.?) // カラムに ? をつけるとOptionで値を包める（NULLはNoneになる）
      
```

## コード生成
既存のDBからscalaのコードを生成できる。

Build.scalaに
```scala
import sbt._
import Keys._
import Tests._

/**
* This is a simple sbt setup generating Slick code from the given
* database before compiling the projects code.
*/

object myBuild extends Build {
  lazy val mainProject = Project(
    id="main",
    base=file("."),
    settings = Project.defaultSettings ++ Seq(
      slick <<= slickCodeGenTask, // register manual sbt command
      sourceGenerators in Compile <+= slickCodeGenTask // register automatic code generation on every compile, remove for only manual use
    )
  )

  // code generation task
  lazy val slick = TaskKey[Seq[File]]("gen-tables")
  lazy val slickCodeGenTask = (sourceManaged, dependencyClasspath in Compile, runner in Compile, streams) map { (dir, cp, r, s) =>
    val outputDir = (dir / "slick").getPath // place generated files in sbt's managed sources folder
    val url = "jdbc:mysql://localhost/slickTest" // connection info for a pre-populated throw-away, in-memory db for this demo, which is freshly initialized on every run
    val jdbcDriver = "com.mysql.jdbc.Driver"
    val slickDriver = "scala.slick.driver.MySQLDriver"
    val pkg = "example"
    toError(r.run("scala.slick.model.codegen.SourceCodeGenerator", cp.files, Array(slickDriver, jdbcDriver, url, outputDir, pkg), s.log))
    val fname = outputDir + "/" + pkg + "/Tables.scala"
    Seq(file(fname))
  }
}
```

などと指定すればsbtのビルド時に自動でテーブル定義がなされる。

上のように書いたときはテーブル定義のコードは`/target/scala-2.10/src_managed/slick/example/Tables.scala`に吐かれる。
例のコードで生成したDBを元に生成させると

```scala
package example
// AUTO-GENERATED Slick data model
/** Stand-alone Slick data model for immediate use */
object Tables extends {
  val profile = scala.slick.driver.MySQLDriver
} with Tables

/** Slick data model trait for extension, choice of backend or usage in the cake pattern. (Make sure to initialize this late.) */
trait Tables {
  val profile: scala.slick.driver.JdbcProfile
  import profile.simple._
  import scala.slick.model.ForeignKeyAction
  import scala.slick.jdbc.{GetResult => GR}
  // NOTE: GetResult mappers for plain SQL are only generated for tables where Slick knows how to map the types of all columns.
  
  /** Entity class storing rows of table Groupid
   *  @param name Database column name 
   *  @param gid Database column gid  */
  case class GroupidRow(name: String, gid: Int)
  /** GetResult implicit for fetching GroupidRow objects using plain SQL queries */
  implicit def GetResultGroupidRow(implicit e0: GR[String], e1: GR[Int]): GR[GroupidRow] = GR{
    prs => import prs._
    GroupidRow.tupled((<<[String], <<[Int]))
  }
  /** Table description of table GroupId. Objects of this class serve as prototypes for rows in queries. */
  class Groupid(tag: Tag) extends Table[GroupidRow](tag, "GroupId") {
    def * = (name, gid) <> (GroupidRow.tupled, GroupidRow.unapply)
    /** Maps whole row to an option. Useful for outer joins. */
    def ? = (name.?, gid.?).shaped.<>({r=>import r._; _1.map(_=> GroupidRow.tupled((_1.get, _2.get)))}, (_:Any) =>  throw new Exception("Inserting into ? projection not supported."))
    
    /** Database column name  */
    val name: Column[String] = column[String]("name")
    /** Database column gid  */
    val gid: Column[Int] = column[Int]("gid")
  }
  /** Collection-like TableQuery object for table Groupid */
  lazy val Groupid = TableQuery[Groupid]
  
  /** Entity class storing rows of table Userid
   *  @param name Database column name 
   *  @param uid Database column uid  */
  case class UseridRow(name: String, uid: Int)
  /** GetResult implicit for fetching UseridRow objects using plain SQL queries */
  implicit def GetResultUseridRow(implicit e0: GR[String], e1: GR[Int]): GR[UseridRow] = GR{
    prs => import prs._
    UseridRow.tupled((<<[String], <<[Int]))
  }
  /** Table description of table UserId. Objects of this class serve as prototypes for rows in queries. */
  class Userid(tag: Tag) extends Table[UseridRow](tag, "UserId") {
    def * = (name, uid) <> (UseridRow.tupled, UseridRow.unapply)
    /** Maps whole row to an option. Useful for outer joins. */
    def ? = (name.?, uid.?).shaped.<>({r=>import r._; _1.map(_=> UseridRow.tupled((_1.get, _2.get)))}, (_:Any) =>  throw new Exception("Inserting into ? projection not supported."))
    
    /** Database column name  */
    val name: Column[String] = column[String]("name")
    /** Database column uid  */
    val uid: Column[Int] = column[Int]("uid")
  }
  /** Collection-like TableQuery object for table Userid */
  lazy val Userid = TableQuery[Userid]
}
```

というテーブル定義が生成される。上の`Build.scala`の場合は

```scala
import example.Tables._
import example.Tables.profile.simple._
```

とimportすれば使える。

基本的には自前で定義したときと同じように扱えるが、テーブルのカラムの型がタプルではなくなっているので、

```scala
// ...
// INSERT
userId += UseridRow("Alice", 0)
userId += UseridRow("Bob",1)
        
userId ++= Seq(UseridRow("Carol", 2),
               UseridRow("Dave", 3),
               UseridRow("Ellen", 4))
       
println("[0]")
userId.foreach {case UseridRow(user, uid) =>
  println(s"User: ${user},\tUserId: ${uid}");
}
// ...
```

のように一々コンストラクタをつけなければならない。（省略する方法を調査中。）

## その他
## String Interpolation

String Interpolationで生のSQL文を埋めこめる。
SQLインジェクションの心配はないらしい。（単純に埋めこんだ文字列をSQL文として解釈するわけではない。）

```scala
def userOfGid(gid: Int) = sql"select * from user-id where GID=$gid".as[UserId]
```
## ベンチマーク

### 要素数5, 10のテーブルに対するINSERT
256, 512カラムを（**一度に**）挿入して速度を計測した。

実行結果:
```
INSERT 5×256 COLUMNS:	139msec
INSERT 10×256 COLUMNS:	108msec
INSERT 5×512 COLUMNS:	133msec
INSERT 10×512 COLUMNS:	141msec

INSERT 5×256 COLUMNS:	90msec
INSERT 10×256 COLUMNS:	100msec
INSERT 5×512 COLUMNS:	117msec
INSERT 10×512 COLUMNS:	125msec

INSERT 5×256 COLUMNS:	88msec
INSERT 10×256 COLUMNS:	91msec
INSERT 5×512 COLUMNS:	125msec
INSERT 10×512 COLUMNS:	125msec

INSERT 5×256 COLUMNS:	89msec
INSERT 10×256 COLUMNS:	108msec
INSERT 5×512 COLUMNS:	125msec
INSERT 10×512 COLUMNS:	125msec

INSERT 5×256 COLUMNS:	81msec
INSERT 10×256 COLUMNS:	100msec
INSERT 5×512 COLUMNS:	117msec
INSERT 10×512 COLUMNS:	117msec

```
<a name="slickOneByOne">
scalikeJDBCとの比較のため、あえて各要素をforeachで一つずつ挿入してみたときの結果が以下。

```
INSERT 5×256 COLUMNS:	12343msec
INSERT 10×256 COLUMNS:	12205msec
INSERT 5×512 COLUMNS:	24691msec
INSERT 10×512 COLUMNS:	24640msec
```

### UPDATE
128または256本のカラムの1要素を更新する時間を計測。

```
UPDATE 128 COLUMNS OF (5×512) TABLE:	6154msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	6440msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	12920msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	12355msec

UPDATE 128 COLUMNS OF (5×512) TABLE:	6598msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	6157msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	12870msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	12529msec

UPDATE 128 COLUMNS OF (5×512) TABLE:	6139msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	6424msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	12304msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	12647msec

UPDATE 128 COLUMNS OF (5×512) TABLE:	6424msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	6182msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	12821msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	12263msec

UPDATE 128 COLUMNS OF (5×512) TABLE:	6180msec
UPDATE 128 COLUMNS OF (10×512) TABLE:	6499msec
UPDATE 256 COLUMNS OF (5×512) TABLE:	12404msec
UPDATE 256 COLUMNS OF (10×512) TABLE:	12371msec
```

### JOIN
JOIN自体は(INNER JOINもLEFT JOINも)「JOINされたクエリ」を生成するだけなので1msecもかからない。
なので`foreach`で全て舐めるのにかかる時間を計測した。

```
INNER JOIN OF 5×256 + 5×512:	84msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	40msec
INNER JOIN OF 5×256 + 5×512:	18msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	25msec
INNER JOIN OF 5×256 + 5×512:	18msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	11msec
INNER JOIN OF 5×256 + 5×512:	7msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	8msec
INNER JOIN OF 5×256 + 5×512:	8msec
OUTER (LEFT) JOIN OF 5×256 + 5×512:	9msec
```

INNER JOIN・LEFT JOIN共に最初だけちょっと遅い。