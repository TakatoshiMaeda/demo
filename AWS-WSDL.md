AWS WSDL
========

AWS WSDL は、AWS の各種 API のリクエスト及びレスポンスのデータ構造を XML として表したもの。

例: EC2(2014-05-21): https://aws.amazon.com/releasenotes/Amazon-EC2/7474008500482873

以下の文章では、

- WSDL の構造とそれが表す意味
- WSDL と API リファレンス・aws-java-sdk の関係

について書く。


XML の構造
-----------

WSDL は XML として書かれている。
WSDL の XML 要素の入れ子関係は下のようになっている。

- element 要素 (API 名が書かれている)
- complexType 要素 (データ型の定義)
  - sequence 要素
    - element 要素
    - choice 要素
      - element 要素
    - group 要素
  - choice 要素
- group 要素 (Enum 型の定義)
  - choice 要素
    - element 要素

これらの意味は基本的には下の3つに分けられる。

- complexType - sequence - element (complexType の下に sequence 要素があり、その下に element 要素がある場合 という意味)
- complexType - sequence - choice - element 及び complexType - choice - element
- complextype - sequence - group 及び group - choice - element

これらが表す意味について下に書く。

### complexType - sequence - element の場合

complexType - sequence - element は、レコード型。

例えば、下のような WSDL があったとき、
```xml
<xs:complexType name="HogeType">
  <xs:sequence>
    <xs:element name="foo" type="xs:string">
    <xs:element name="bar" type="xs:string" minOccurs="0">
  </xs:sequence>
</xs:complexType>
```
これは、Scala では以下のようになる。
```scala
case class Hoge(
  foo: String,
  bar: Option[String]
)
```

### complexType - sequence - choice - element の場合

- complexType - sequence - choice - element
- complexType - choice - element

は、代数的データ型。(2つの違いは、sequence (レコード型) になっているかなっていないかの違い)

例えば、下のような WSDL があったとき、
```xml
<xs:complexType name="HogeType">
  <xs:sequence>
    <xs:choice>
      <xs:element name="foo" type="xs:boolean">
      <xs:element name="bar" type="xs:string">
    </xs:choice>
  </xs:sequence>
</xs:complexType>
```

これは、Scala では以下ようになる。

```scala
case class Hoge(
  choice: HogeChoice
)

sealed trait HogeChoice
case class HogeFoo(
  foo: Boolean
) extends HogeChoice
case class HogeBar(
  bar: String
) extends HogeChoice
```

API リファレンス及び aws-java-sdk は代数的データ型とか考えてないので、意味を考えずに API を叩くとエラーレスポンスがかえってくる (こういうリクエストは出来れば型エラーにしたい)。

### group - choice - element の場合

group - choice - element は、Enum 型。
必ず complexType - sequence - group の属性 ref で指定される。

例えば、下のような WSDL があったとき、
```xml
<xs:group name="HogeGroup">
  <xs:choice>
    <xs:element name="foo" type="tns:EmptyElementType">
    <xs:element name="bar" type="tns:EmptyElementType">
  </xs:choice>
</xs:group>
```
これは、Scala だと以下のようになる。
```scala
sealed trait HogeGroup
case object Foo extends HogeGroup
case object Bar extends HogeGroup
```

### 一般

一般に、以下のような WSDL があったとき、
```xml
<xs:complexType name="HogeType">
  <xs:sequence>
    <xs:element name="foo" type="xs:boolean" minOccurs="0">
    <xs:element name="bar" type="xs:string">
    <xs:choice>
      <xs:element name="baz" type="xs:boolean">
      <xs:element name="qux" type="xs:string">
    </xs:choice>
    <xs:choice>
      <xs:element name="piyo" type="xs:string">
      <xs:element name="fuga" type="xs:string">
    </xs:choice>
    <xs:group ref="tns:HogeGroup">
  </xs:sequence>
</xs:complexType>
<xs:group name="HogeGroup">
  <xs:choice>
    <xs:element name="hoge1" type="tns:EmptyElementType">
    <xs:element name="hoge2" type="tns:EmptyElementType">
    <xs:element name="hoge3" type="tns:EmptyElementType">
  </xs:choice>
</xs:group>
```
これは、Scala だと以下のようになる。
```scala
case class Hoge(
  foo: Option[Boolean],
  bar: String,
  bazQux: BazQux,
  piyoFuga: PiyoFuga,
  hogeGroup: HogeGroup
)

sealed trait BazQux
case class Baz(
  baz: Boolean
) extends BazQux
case class Qux(
  qux: String
) extends BazQux

sealed trait PiyoFuga
case class Piyo(
  piyo: String
) extends PiyoFuga
case class Fuga(
  fuga: String
) extends PiyoFuga

sealed trait HogeGroup
case object Hoge1 extends HogeGroup
case object Hoge2 extends HogeGroup
case object Hoge3 extends HogeGroup
```

WSDL と API リファレンス・aws-java-sdk との関係
--------------------

WSDL から予想される API リファレンスでの定義・aws-java-sdk での定義を考える。

例えば以下のような WSDL があったとする。
```xml
<xs:element name="DescribeHoges" type="tns:DescribeHogesType">
<xs:complexType name="DescribeHogeType">
  <xs:sequence>
    <xs:element name="hogeId" type="xs:string"/>
    <xs:element name="maxResults" type="xs:int" minOccurs="0"/>
    <xs:element name="fugasSet" type="tns:DescribeHogesFugasType"/>
    <xs:group name="DescribeHogesAttributesGroup"/>
  </xs:sequence>
</xs:complexType>

<xs:complexType name="DescribeHogesFugasType">
  <xs:sequence>
    <xs:element name="item" type="tns:DescribeHogesFugaType" minOccurs="0" maxOccurs="unbounded">
  </xs:sequence>
</xs:complexType>

<xs:complexType name="DescribeHogesFugaType">
  <xs:sequence>
    <xs:element name="fuga" type="xs:string"/>
  </xs:sequence>
</xs:complexType>

<xs:group name="DescribeHogesAttributesGroup">
  <xs:choice>
    <xs:element name="foo" type="tns:EmptyElementType"/>
    <xs:element name="bar" type="tns:EmptyElementType"/>
  </xs:choice>
</xs:group>
```

このような定義だと、API リファレンス、aws-java-sdk の定義は、以下のようになる(と予想される)。

(API リファレンス)

パラメータ名 | 型       | 説明
------------ | ------- | ------------
HogeId       | string (Required) | 特になし
maxResults   | int (No Required) | minOccurs="0" の場合は Optional
Fuga.n       | string | ~~Set という名前の型が item というフィールドをもっている場合、(item の型が持っているフィールド名).n がパラメータの名前になる
Attribute    | string (foo または bar) | group の場合は必ず Attribute というパラメータ名になる

(aws-java-sdk)

```java
public class DescribeHogesRequest extends AmazonWebServiceRequest implements ... {

  public DescribeHogesRequest() {}
  public DescribeHogesRequest(String hogeId) { // Required になっているものがコンストラクタの引数になる
    setHogeId(hogeId);
  }

  void setHogeId(String hogeId) { ... }
  void setMaxResults(int maxResults) { ... }
  void setFugas(Collection<String> fugas) { ... }
  void setAttribute(HogeAttributeName attribute) { ... } // WSDL で group(choice) になっている部分は Enum 型がつくられる
  void setAttribute(String attribute) { ... } // String 型も許す

  ... // get~ with~
}
```


WSDL と API リファレンス (もしくは AWS Java SDK) の差異
------------------------------

1. WSDL にフィルターの情報がない。
2. WSDL の方で EmptyElementType (値のない型) になっているのに、aws-java-sdk の方では String を渡したりする。
  - EmptyElementType は choice (Enum 型) のとき使われているのに、aws-java-sdk の方ではレコード型のように扱われている (API リファレンスと合致しない)
  - 下の具体例の DescribeNetworkInterfaceAttribute
3. WSDL での名前と、API リファレンスや aws-java-sdk での名前が一致しないことがある
  - 下の具体例 (DescribeSnapshots の setRestorableByUserIds)など
4. WSDL で定義されているのに、API リファレンスや aws-java-sdk にはないものがある。
  - 下の具体例 (RunInstances の setLicense) など
5. WSDL にないのに、API リファレンスや aws-java-sdk にはあるものがある。
  - 下の具体例 (RunInstances の setSecurityGroupIds) など


全体的に、java-aws-sdk の実装がおかしいことが多いという印象
- Enum 型 (もしくは代数的データ型の直和型) なのに、レコード型として扱われていることがあったり
  - Enum 型の値を複数指定した場合にどうなるかは実際にやってみないとわからない (API の実装によっては通るかもしれないし、通らないかもしれない)
- EmptyElementType なのに String 型だったり
- 名前が WSDL や API リファレンスと一致しなかったり

具体的な API について
------------------------

### RunInstances

RunInstances のクエリパラメータは EC2 の API のなかでもかなり(一番？)複雑なので、これを完全に変換できればどんな API でも変換できそう。しかし、RunInstances は基本的な変換方針では変換できないものがある。

API リファレンス (WSDL からの予想)  | aws-java-sdk の実装 (WSDL からの予想)        | API リファレンス (実際の定義)   | aws-java-sdk (実際の定義)
----------------------------- | ------------------------------------ | ------------------------------------- | ----------------------------
`GroupId.n, GroupName.n` | `setGroupIds(Collection<String>), setGroupNames(Collection<String>)` |  そんなパラメータはない | そんなパラメータはない
`UserData.Data=<string>` | `setUserData(UserData data)` | `UserData=<string>` | `setUserData(String)`
`License.Pool=<string>` | `setLicense(License pool)`  | そんなパラメータはない | そんなパラメータはない
(NetworkInterfaceSet について、~Set という名前なので) `NetworkInterfaceId.n=<string>, DeviceIndex.n=<int> ...` | `setNetworkInterfaceIds(Collection<String>), setDeviceIndices(Collection<int>) ...` | `NetworkInterface.n.NetworkInterfaceId, NetworkInterface.n.DeviceIndex` | `setNetworkInterface(Collection<NetworkInterface>)`
`Monitoring.Enabled=<boolean>` | `setMonitoring(Enabled enabled)` | `Monitoring.Enabled=<boolean>` (これは UserData と矛盾) | `setMonitoring(Boolean)`
なし  |   なし  | `SecurityGroupId.n`, `SecurityGroup.n` | `setSecurityGroupIds(Collection<String>)`, `setSecurityGroup(Collection<String>)`

\~\~Set という名前でも item の中身が複数あると、\~\~.n.\~\~ というようになる？(~~Set という名前で item が中身にあるようなものは、item の中は一つしかないものが多い(そしてそういう場合に基本方針のようになっている))

(RequestSpotInstances の LaunchSpecification でも同様だったので) WSDL における GroupSetType (GroupId.n, GroupName.n) が、API リファレンスにおける SecurityGroupId.n と SecurityGroup.n のことなんだと思うが、こういうのがあると自動生成できない。

また、WSDL で instanceType は String 型になっているが、aws-java-sdk では、String 型を受け取る setter だけではなく、InstanceType という Enum 型を受け取る setter 型も提供している。こういうのは aws-java-sdk のように Enum 型にしたいが、WSDL にはそのための情報(m1.small とか m1.medium とか)がない。これだと m1.small や m1.medium などを要素とした Enum 型を自動生成できない。


### DescribeSnapshots

WSDL からすると、User.n なのに、API リファレンスは RestorableBy.n となっており、aws-java-sdk は `setRestorableByUserIds(Collection<String>)` となっている。

WSDL 及び API リファレンスは Owner.n なのに、aws-java-sdk は `setOwnerIds(Collection<String>)`


### DescribeNetworkInterfaceAttribute

[aws-java-sdk の DescribeNetworkInterfaceAttributeRequestMarshaller](https://github.com/aws/aws-sdk-java/blob/master/src%2Fmain%2Fjava%2Fcom%2Famazonaws%2Fservices%2Fec2%2Fmodel%2Ftransform%2FDescribeNetworkInterfaceAttributeRequestMarshaller.java) と、
[API reference の DescribeNetworkInterfaceAttribute](http://docs.aws.amazon.com/AWSEC2/latest/APIReference/ApiReference-query-DescribeNetworkInterfaceAttribute.html)
を見ると、API リファレンス的には `Attribute=description` のようになるはずだが、aws-java-sdk の方では `Description=(something)` のようになっており、aws-java-sdk が間違っているように見える。でも一応動く。
setDescription の引数が空文字列のときしかうまく動かない (API を叩いたときエラーが返る。すなわちコンパイルが通ってもランタイムエラー) ので、どうしてこういう実装にしているかわからない。


意見
----

自分(安武 @amutake)の考えとしては、以下の理由によって、自動生成させないほうがいいと思います。


- コスパが悪い
  - 時間をかけて頑張って自動生成しても、そんなにいいものはできなさそう。
  - 自分で書いたほうが、自動生成させるのと同じ時間で、いいものができそう。

詳しくは、

- API リファレンス、aws-java-sdk との差異がけっこうあり、差異に対応させる時間と手で書く時間が同じくらいかもしれない
  - 名前の違いがけっこうあるので、そういう違いを見つけることと、違いにいちいち対応させるのが面倒そう
  - 名前以外にも、aws-java-sdk とのすり合わせが面倒そう
  - コンパイルは通っても API を叩くとエラーが返ってくるということが多そう(そういうのを見つけるのに時間がかかる)
  - 自動生成させる理由の一つに、API のバージョンが上がった場合も自動で対応できるというのがあるけど、
    バージョンアップに伴ってまた新しい例外ができているかもしれないので、どっちにしろチェックしないといけないということになりそう
- 自動生成しても、ちゃんと、API を間違って使おうとしている場合にコンパイル時に検出できるようにならなさそう
  - フィルターの情報がないので、間違ったフィルターを指定できる
  - 例えば、RunInstances の InstanceType は case object にしたいけど、WSDL から自動生成したものは String になる。これはあんまり安全じゃない。
  - Option かどうかくらいなら大丈夫だけど…

メモ
----

- element (API 名と、API 名 + Response)
- complexType
  - sequence
    - element (通常のパラメータ)
    - choice
      - element (通常のパラメータだと思っていい(代数的データ型))
    - group (ref しかない)
  - choice (e.g., DisassociateAddressType)
- group (ref で指定されたもの、element の直和型)
  - choice
    - element


complexType - sequence - element が一つしかない場合、element の型になる？
