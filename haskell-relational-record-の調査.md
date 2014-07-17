haskell-relational-record
=========================

haskell-relational-record とは
-----------------------------

@khibino さんが開発している、Haskell からデータベースを操作するためのライブラリ群。

SQL の値と Haskell のレコードをマッピングしたり、型安全な SQL を作れたりする。


リンク
------

ソースコード
- https://github.com/khibino/haskell-relational-record

サンプル
- https://github.com/khibino/haskell-relational-record/tree/master/doc/tutorial
- https://github.com/khibino/haskell-relational-record/tree/master/doc/examples
- https://github.com/khibino/haskell-relational-record/tree/master/relational-join-HDBC-pgTest
- https://github.com/amutake/hrr-examples


パッケージの構成
----------------

haskell-relational-record は、複数のパッケージで構成されている。

 パッケージ名          | 説明
 --------------------- | ------------------------------------------------------------------
 names-th              | Template Haskell のライブラリ。CamelCase の名前を生成するものなど。
 sql-words             | SQL のワードの定義。OverloadedStrings で書けるようになっている。
 DB-record             | SQL の値と Haskell のレコードをマッピングするためのライブラリ。
 relational-join       | SQL を構築するためのライブラリ。
 HDBC-session          | HDBC とのコネクションのエラーハンドリング。
 relational-query-HDBC | 以上のものを実際に HDBC 経由で使えるようにするためのライブラリ。


簡単な例
--------

haskell-relational-record を使った簡単な例。データベースは PostgreSQL を使う。

[全体のソースコード](https://github.com/amutake/hrr-examples/tree/master/DQ)

### 準備

まずデータベースにテーブルを作る。

family テーブルはモンスターの系統、 monster テーブルはモンスター。

```sql
/* create.sql */

create schema DQ;

/* 系統 */
create table DQ.family (
  family_id integer not null,
  family_name varchar(255) not null,
  primary key (family_id)
);

/* モンスター */
create table DQ.monster (
  monster_id integer not null,
  monster_name varchar(255) not null,
  level integer not null,
  family_id integer, /* 魔王の場合はNULL */
  primary key (monster_id),
  foreign key (family_id) references DQ.family (family_id)
);

/* スライム系とドラゴン系 */
insert into DQ.family (family_id, family_name) values (1, 'slime');
insert into DQ.family (family_id, family_name) values (2, 'dragon');

/* 適当にモンスターを入れる */
insert into DQ.monster (monster_id, monster_name, level, family_id) values (1, 'slime', 4, 1);
insert into DQ.monster (monster_id, monster_name, level, family_id) values (2, 'buchi-slime', 5, 1);
insert into DQ.monster (monster_id, monster_name, level, family_id) values (3, 'metal-king', 30, 1);
insert into DQ.monster (monster_id, monster_name, level, family_id) values (4, 'dragon', 10, 2);
insert into DQ.monster (monster_id, monster_name, level, family_id) values (5, 'great-dragon', 20, 2);
insert into DQ.monster (monster_id, monster_name, level, family_id) values (6, 'shinryu', 40, 2);
insert into DQ.monster (monster_id, monster_name, level) values (7, 'ryuoh', 50);
insert into DQ.monster (monster_id, monster_name, level) values (8, 'death-pisaro', 50);
```

以上の SQL を実行する。

```sh
$ createdb testdb
$ psql -f create.sql testdb
```


### テーブル定義から TH で型などを生成

Relational.HDBC.Query.TH の機能を使って、 DB のテーブル情報から Haskell のレコード型、インスタンスの定義およびユーティリティ関数を生成する。

まず、 DB とのコネクションを得る関数 `connect` を作る。

```haskell
-- DataSource.hs

module DataSource
  ( connect
  ) where

import Database.HDBC.PostgreSQL (connectPostgreSQL, Connection) -- HDBC-postgresql パッケージ

-- コネクションを得る関数
connect :: IO Connection
connect = connectPostgreSQL "dbname=testdb"
```

次に、`defineTableFromDB` を使って、 family テーブル、 monster テーブルそれぞれの定義から、 型や型クラスのインスタンスなどを生成する。

```haskell
defineTableFromDB 
    :: IConnection conn -- コネクションの型クラス (IConnection については後述)
    => IO conn -- コネクション
    -> Driver conn -- DBのドライバ
    -> String -- スキーマ名
    -> String -- テーブル名
    -> [ConName] -- derive する型クラス (ConName の値は Database.Record.TH に定義されている)
    -> Q [Dec]
```

```haskell
-- Family.hs

{-# LANGUAGE TemplateHaskell, MultiParamTypeClasses #-}

module Family where

import Database.HDBC.Query.TH (defineTableFromDB)
import Database.HDBC.Schema.PostgreSQL (driverPostgreSQL)
import Database.Record.TH (derivingShow)

import DataSource (connect)

defineTableFromDB connect driverPostgreSQL "DQ" "family" [derivingShow]
```

```haskell
-- Monster.hs

{-# LANGUAGE TemplateHaskell, MultiParamTypeClasses #-}

module Monster where

import Database.HDBC.Query.TH (defineTableFromDB)
import Database.HDBC.Schema.PostgreSQL (driverPostgreSQL)
import Database.Record.TH (derivingShow)

import DataSource (connect)

defineTableFromDB connect driverPostgreSQL "DQ" "monster" [derivingShow]
```

以上を行うと、

```haskell
data Family = Family
    { familyId :: !Int32
    , familyName :: !String
    } deriving (Show)

family :: Relation () Family
```

のようなコードが生成される。(生成される全コードは、 「defineTableFromDBが生成するもの」の節を参照)


### クエリの作成

クエリを作っていく。主に relational-join パッケージを使う。

まず簡単なものとして、

```sql
SELECT * FROM FAMILY WHERE FAMILY_NAME = 'slime';
```

という SQL に相当するものは、

```haskell
findSlimeFamily :: Relation () Family
findSlimeFamily = relation $ do
    f <- query family
    wheres $ f ! familyName' .=. value "slime"
    return f
```

となる。

これを `show` すると、以下の SQL となる。

```sh
shci> findSlimeFamily
SELECT T0.family_id AS f0, T0.family_name AS f1 FROM DQ.family AS T0 WHERE (T0.family_name = 'slime')
```

その他

```haskell
-- Query.hs

module Query where

import Data.Int (Int32)

import Database.Relational.Query -- クエリを作る用のものはほとんどこのモジュールに入っている

import Family (Family, family)
import qualified Family as F
import Monster (Monster, monster)
import qualified Monster as M

-- select * from DQ.family where family_name = 'slime'
findSlimeFamily :: Relation () Family
findSlimeFamily = relation $ do
    f <- query family
    wheres $ f ! F.familyName' .=. value "slime"
    return f

-- select f.family_id, f.family_name, m.monster_id, m.monster_name, m.level, m.family_id
-- from DQ.family f inner join DQ.monster m using (family_id)
-- order by monster_id 
familyMonsters :: Relation () (Family, Monster)
familyMonsters = relation $ do
    f <- query family
    m <- query monster
    on $ just (f ! F.familyId') .=. m ! M.familyId'
    asc $ m ! M.monsterId'
    return (f >< m)

-- select * from DQ.monster where level >= ? and level <= ?
findMonstersByLevelRange :: Relation (Int32, Int32) Monster
findMonstersByLevelRange = relation' $ do -- プレースホルダがあるときは relation'
    m <- query monster
    (ph, ()) <- placeholder $ \levelRange -> wheres $
        (m ! M.level' .>=. levelRange ! fst') `and'`
        (m ! M.level' .<=. levelRange ! snd')
    return (ph, m)

-- select family_id from DQ.family where family_name = ?
findFamilyIdByName :: Relation String Int32
findFamilyIdByName = relation' $ do
    f <- query family
    (ph, ()) <- placeholder $ \name ->
        wheres $ f ! F.familyName' .=. name
    return (ph, f ! F.familyId')

-- select m.monster_id, m.monster_name, m.level, m.family_id 
-- from DQ.monster m
-- inner join (select family_id from DQ.family where family_name = ?) f
-- on m.family_id = f.family_id
findMonstersByFamilyName :: Relation String Monster
findMonstersByFamilyName = relation' $ do
    m <- query monster
    (ph, i) <- query' findFamilyIdByName -- 作ったものを組み合わせることが出来る
    on $ m ! M.familyId' .=. just i
    return (ph, m)

-- select * from DQ.monster where family_id is null
findBoss :: Relation () Monster
findBoss = relation $ do
    m <- query monster
    wheres $ isNull $ m ! M.familyId'
    return m
```


### 実行

HDBC-session パッケージの `withConnectionIO` 関数と、 `handleSqlError'` 関数を使ってエラーハンドリングをすると良い。

作ったクエリ(`Relation p r` 型のもの)は、 `relationalQuery` で `Query p r` 型に変換し、 `runQuery` で実行する。

```haskell
-- main.hs

module Main where

import Database.Relational.Query (relationalQuery)
import Database.HDBC.Session (withConnectionIO, handleSqlError')
import Database.HDBC.Record.Query (runQuery)

import DataSource (connect)
import Query

run conn rel param = runQuery conn param (relationalQuery rel) >>= print

main :: IO ()
main = handleSqlError' $ withConnectionIO connect $ \conn -> do

    run conn familyMonsters ()

    run conn findMonstersByLevelRange (20, 40)

    run conn findMonstersByFamilyName "slime"
    run conn findMonstersByFamilyName "dragon"

    run conn findBoss ()
```



モナド内包表記
------------

モナド内包表記を使っても書けるようになっている。

例えば上の `findMonstersByLevelRange` の例だと、

```haskell
{-# LANGUAGE MonadComprehensions #-}

findMonstersByLevelRange' :: Relation (Int32, Int32) Monster
findMonstersByLevelRange' = relation'
    [ (ph, m)
    | m <- query monster
    , let lv = u ! level'
    , (ph, ()) <- placeholder $ \levelRange -> wheres $
        (lv .>=. levelRange ! fst') `and'`
        (lv .<=. levelRange ! snd')
    ]
```

と書ける。


主要な型および型クラス
--------

### IConnection 型クラス

haskell-relational-record ではなく、 HDBC パッケージの `Database.HDBC.Types` で定義されている、データベースとのコネクションについての型クラス。

http://hackage.haskell.org/package/HDBC/docs/Database-HDBC-Types.html

15個ほどメソッドがあるが、主なものを紹介

メソッド | 型 | 説明
--------- | --- | ----- 
disconnect | conn -> IO () | コネクションを閉じる
commit | conn -> IO () | トランザクションのコミット(HDBC は自動コミットされない)
rollback | conn -> IO () | ロールバックを行う

※`conn` は `IConnection` のインスタンス


### Relation p r 型

`relational-join` パッケージの `Database.Relational.Query.Relation` モジュールに定義されている。

`SELECT` 文になる型で、 `p` がプレースホルダの型、 `r` が結果の型となっている。

`show` すると SQL になる。

#### 作り方

基本的には、`relation` (または `relation'`) に `QuerySimple` モナドを引数に渡してやるとできる。

プレースホルダなし(`Relation () r` 型)のものをつくるときは `relation` 、
プレースホルダあり(`Relation p r` 型)のものをつくるときは `relation'` を使う。

```haskell
relation :: QuerySimple (Projection Flat r) -> Relation () r
relation' :: QuerySimple (PlaceHolders p, Projection Flat r) -> Relation p r
```

以下は例。

```haskell
-- TH で生成されるもの
monster :: Relation () Monster
monster = Database.Relational.Query.Relation.table $
    Database.Relational.Query.Table.table "DQ.monster"
        [ "monster_id"
        , "monster_name"
        , "level"
        , "family_id"
        ]

-- relation を使った例
familyMonster :: Relation () (Family, Monster)
familyMonster = relation $ do
    f <- query family
    m <- query monster
    on $ just (f ! Family.familyId') .=. m ! Monster.familyId'
    return (g >< u)

-- relation' を使った例
findFamilyByName :: Relation String Family
findFamilyByName = relation' $ do
    f <- query family
    -- ここでの placeholder の型は、
    -- placeholder :: (Projection Flat String -> QuerySimple ()) -> QuerySimple (PlaceHolders String, ())
    (ph, ()) <- placeholder $ \name -> wheres $ 
        f ! familyName' .=. name
    return (ph, f) -- ph :: PlaceHolders String も return する
```


#### 使い方

SQL を発行する。

```haskell
-- 下のような Relation 型をもつ値があるとすると、
sampleRelation :: Relation Int SampleData
sampleRelation = undefined

-- relationalQuery を使って Query 型に変換し、
sampleQuery :: Query Int SampleData
sampleQuery = relationalQuery sampleRelation

-- runQuery にコネクション、プレースホルダに入れる値、 Query 型の値を渡してやると、クエリが発行される
getSampleData :: IConnection conn => conn -> Int -> IO [SampleData]
getSampleData conn param = runQuery conn param sampleQuery
```

また、 Relation を組み合わせて別の Relation を作るときにも使える。

```haskell
-- select * from family where family_name = 'slime'
findSlimeFamily :: Relation () Family
findSlimeFamily = relation $ do
    f <- query family
    wheres $ f ! familyName' .=. value "slime"
    return f

-- select m.monster_name from monster m 
-- inner join (select * from family where family_name = 'slime') s
-- on s.family_id = m.family_id
findSlimeName :: Relation () String
findSlimeName = relation $ do
    m <- query monster
    s <- query findSlimeFamily
    on $ m ! familyId' .=. just (s ! familyId')
    return (m ! familyName')
```

### Update p 型

update 文になる型。 p 型はプレースホルダの型。

#### 作り方

`typedUpdate` 関数と、 `UpdateTarget p r` 型の値を使って作る。

#### 使い方

`runUpdate :: (IConnection conn, ToSql SqlValue p) => conn -> p -> Update p -> IO Integer` を使って、 update を行う。

```haskell
data Hoge = Hoge { hogeId :: Int32, hogeName :: String }

runUpdateName :: (IConnection conn) => conn -> String -> Update String -> IO ()
runUpdateName conn name = runUpdate conn name updateName
```

### Insert a 型

insert 文になる型。 a 型はレコードの型になる。

relational-join パッケージの、 Database.Relational.Query.Type に定義されている。

#### 作り方

`defineTableFromDB` などを行うと、 `insertHoge :: Insert Hoge` が自動的に生成されるので、それを使う。

#### 使い方

`runInsert :: IConnection conn => conn -> a -> Insert a -> IO Integer` を使って、 insert する。

```haskell
hoge :: Hoge

putHoge :: IConnection conn => conn -> Hoge -> IO ()
putHoge conn hoge = runInsert conn hoge insertHoge >> return ()
```

### Delete p 型

delete 文になる型

### Pi r0 r1 型

Projection Path

r0 から r1 への関数のようなもの

### Projection c t 型

Haskell のレコード t 型に射影される(？)

### Flat 型

aggregated query ではないことをあらわす幽霊型。
Projection 型コンストラクタの第一引数になることが多い。

### Aggregated 型

aggregated query であることをあらわす幽霊型。
Projection 型コンストラクタの第一引数になることが多い。


defineTableFromDB で生成されるもの
-----------------------------------

"簡単な例"の monster テーブルについて、 `defineTableFromDB connect driverPostgreSQL "DQ" "monster" [derivingShow]` を行うと、以下の Haskell コードが生成される。また、読みやすいように、関数名が被っているもの以外はモジュール名を省略している。

```haskell
data Monster = Monster
    { monsterId :: !Int32
    , monsterName :: !String
    , level :: !Int32
    , familyId :: !(Maybe Int32)
    } deriving (Show)

instance PersistableWidth Monster where
    persistableWidth = unsafePersistableRecordWidth 4

tableOfMonster :: Table Monster
tableOfMonster = Database.Relational.Query.Table.table "DQ.monster"
    [ "monster_id"
    , "monster_name"
    , "level"
    , "family_id"
    ]

monster :: Relation () Monster
monster = Database.Relational.Query.Relation.table tableOfMonster

monsterId' :: Pi Monster Int32
monsterId' = definePi 0

monsterName' :: Pi Monster String
monsterName' = definePi 1

level' :: Pi Monster Int32
level' = definePi 2

familyId' :: Pi Monster (Maybe Int32)
familyId' = definePi 3

insertMonster :: Insert Monster
insertMonster = typedInsert tableOfMonster

instance HasColumnConstraint Primary Monster where
    columnConstraint = unsafeSpecifyColumnConstraint 0

instance HasKeyConstraint Primary Monster where
    keyConstraint = derivedCompositePrimary

instance HasConstraintKey Primary Monster Int32 where
    constraintKey = unsafeDefineConstraintKey [0]

selectMonster :: Query Int32 Monster
selectMonster = relationalQuery (primary monster)

updateMonster :: KeyUpdate Int32 Monster
updateMonster = primaryUpdate tableOfMonster

instance HasColumnConstraint NotNull Monster where
    columnConstraint = unsafeSpecifyColumnConstraint 0

fromSqlOfMonster :: [SqlValue] -> Monster
fromSqlOfMonster [f1,f2,f3,f4] = Monster (fromSql f1) (fromSql f2) (fromSql f3) (fromSql f4)
fromSqlOfMonster _ = error "Generated code of 'defineRecordConstructFunction': Fail to pattern match in: fromSqlOfMonster, count of columns is 4"

toSqlOfMonster :: Monster -> [SqlValue]
toSqlOfMonster rec =
    [ toSql (monsterId rec)
    , toSql (monsterName rec)
    , toSql (level rec)
    , toSql (familyId rec)
    ]

instance Persistable SqlValue Monster where
    persistable = persistableRecord persistableWidth fromSqlOfMonster toSqlOfMonster

instance FromSql SqlValue Monster where
    recordFromSql = recordFromSql'

instance ToSql SqlValue Monster where
    recordToSql = recordToSql'
```