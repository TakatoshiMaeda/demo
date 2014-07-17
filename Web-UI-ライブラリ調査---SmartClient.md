# SmartClient

- [公式サイト](http://smartclient.com/)
- [Examples](http://smartclient.com/#Welcome)
- [Reference](http://smartclient.com/docs/8.3/a/b/c/go.html)
- [クイックスタート(PDF)](http://smartclient.com/docs/8.3/docs/SmartClient_Quick_Start_Guide.pdf)
- [サンプル](https://github.com/amkkun/SmartClientExample)

## 特徴

- ExtJSと似たような方向性
- ライセンスがLGPLなので商用利用可
- ドキュメントや例もひと通り揃っている
- XMLで書けるという特徴があるが、XMLなしでもOK
- MVCが分離していない？
- JSのみで画面を作ることができる
- Wc3の完全移植(コピー)は可能

## ダウンロード

1. [ダウンロード画面](http://smartclient.com/product/download.jsp)からSmartClient 8.3 LGPL Editionをクリックしてダウンロード(会員登録画面に移るが、登録の必要はなし)
2. unzip
3. `cp -r SmartClient-8.3/smartclientRuntime/isomorphic path/to/`

- smartclientRuntime/isomorphicディレクトリのなかに必要なモジュールが全てはいっている
- smartclientSDKにはドキュメントやサンプルが入っている
    - start_embedded_server.shを実行するとサーバが立ち上がる
    - localhost:8080にアクセスするとドキュメントなどを見ることができる


## 使い方

### ディレクトリ構成

ファイルの構成は下のように仮定
```
root --- index.html
      |
      |- main.js
      |
      |- isomorphic - ...
```


### index.html

index.htmlは下のようにする。
必要なモジュールは全てhtmlに書かなければいけない。

(skins/Enterprise/load_skin.js のEnterpriseの部分をEnterpriseBlueなどに変えるとスキンを変更できる)

```html
<html>
  <head>
    <script>var isomorphicDir = "./isomorphic/";</script> <!-- なくても動く？ -->
    <script src="./isomorphic/system/modules/ISC_Core.js"></script>
    <script src="./isomorphic/system/modules/ISC_Foundation.js"></script>
    <script src="./isomorphic/system/modules/ISC_Containers.js"></script>
    <script src="./isomorphic/system/modules/ISC_Grids.js"></script>
    <script src="./isomorphic/system/modules/ISC_Forms.js"></script>
    <script src="./isomorphic/system/modules/ISC_DataBinding.js"></script>
    <script src="./isomorphic/skins/Enterprise/load_skin.js"></script>
  </head>
  <body>
    <script src="main.js"></script> <!-- bodyに書かないとエラー？ -->
  </body>
</html>
```

### main.js

main.jsにいろいろ書いていく。
基本的には下のように、コンポーネントを作って組み合わせていく。

```js
isc.Hogehoge.create({
    ID: "hogehoge",
    width: 300, height: 200,
    ...
})

isc.Fugafuga.create({
    ID: "fugafuga",
    ...
})
```

IDを使う代わりに下のような書き方もできる。
```js
var button = isc.IButton.create({
    title: "hogehoge"
});

button.click = function() {
    alert("hoge!!!");
};
```

## API

### DataSource

APIを叩くときには通常はDataSourceというクラスを使う。

```js
isc.DataSource.create({
    ID: "ds",
    dataFormat: "json",
    dataURL: "/rest/hoge",
    fields: [
        { name: "id", title: "ID" },
        { name: "name", title: "Name" }
    ]
})
```

#### 例: GETしグリッドに格納する
```js
isc.DataSource.create({
    ID: "hogeDS",
    dataFormat: "json",
    dataURL: "/rest/hoge"
    fields: [
        { name: "id", title: "ID" },
        { name: "name", title: "Name" }
    ]
})

isc.ListGrid.create({
    ID: "hogeList",
    width: "100%", height: 500,
    dataSource: hogeDS,
    autoFetchData: true
})
```

### RPCManager

DataSourceよりプリミティブなものとして、RPCManagerというクラスが用意されている。

GETの例:
```js
RPCManger.sendRequest({
    httpMethod: "GET",
    actionURL: "/rest/getUsers",
    callback: function(rpcResponse, data, rpcRequest) {
        ...
    },
    paramsOnly: true, // これがないとクエリパラメータに余計なものが入る
    evalResult: true, // 結果をevalされた後の状態で受け取れる
})
```

POSTの例(FORM形式):
```js
RPCManager.sendRequest({
    httpMethod: "POST",
    actionURL: "/rest/addUser",
    params: {
        userId: 1,
        userName: "aaa",
        userUserName: "aaa"
    },
    paramsOnly: true,
})
```

POSTの例(JSON形式):
```js
var encoded = isc.JSON.encode(user);
RPCManager.sendRequest({
    httpMethod: "POST",
    contentType: "application/json",
    httpHeaders: {
        "Accept": "application/json"
    },
    actionURL: "/rest/addUserJSON",
    data: encoded,
    paramsOnly: true
})
```


## クラス定義

既存のクラスを継承して新しいクラスを作ることができる(isc.ClassFactory.defineClass)

```js
isc.ClassFactory.defineClass("NewClass", isc.HogeClass);
isc.NewClass.addProperties({
    ...
})
```


#### 例:リフレッシュボタンとデータグリッドというよくある組み合わせをクラスにしたもの
```js
// VLayout を継承したRefreshPanelというクラスを作る
isc.ClassFactory.defineClass("RefreshPanel", isc.VLayout);

isc.RefreshPanel.addProperties({
    membersMargin: 10,
    // initWidgetがコンポーネントが作られるときに実行される関数
    initWidget: function() {
        var me = this;
        me.Super("initWidget", arguments);

        me.dataSource = isc.DataSource.create({
            dataFormat: me.dataFormat,
            dataURL: me.dataURL,
            fields: me.fields
        })

        me.refreshButton = isc.IButton.create({
            title: "refresh",
            click: function() {
                me.refresh();
            }
        })

        me.createGrid();

        me.addMember(me.refreshButton);
        me.addMember(me.grid);
    },
    createGrid: function() {
        var me = this;
        me.grid = isc.ListGrid.create({
            width: me.width, height: me.height,
            dataSource: me.dataSource,
            autoFetchData: true,
        })
    },
    refresh: function() {
        var me = this;
        me.grid.destroy();
        me.createGrid();
        me.addMember(me.grid);
    }
})

// 新しく作ったRefreshPanelも他のクラスと同様に.createで作る
isc.RefreshPanel.create({
    ID: "userPanel",
    width: 400, height: 500,
    dataFormat:"json",
    dataURL:"/rest/user.js",
    fields:[
        { name: "id", title: "ID" },
        { name: "name", title: "Name" },
        { name: "username", title: "UserName" }
    ]
})

isc.RefreshPanel.create({
    ID: "groupPanel",
    width: 400, height: 500,
    dataFormat: "json",
    dataURL: "/rest/group.js",
    fields: [
        { name: "id", title: "ID" },
        { name: "name", title: "Name" }
    ]
})
```


## 問題点

- コンポーネントが多くなってきたときの依存関係

    - 初期化でコンポーネントAがコンポーネントBを呼び出しているとき、コンポーネントBはコンポーネントAより先に定義しなければならない。
    - 初期化以外のイベント(クリックイベントなど)により呼び出されるものはその限りではない。

- JSファイル内で他のモジュールを呼び出すことができない

    - モジュール間の依存関係がわかりにくい。
    - 必要なJSファイルはすべてHTMLに書かなければならない。
    - ファイルが多くなってきたとき、ファイルを読み込む順番によってはエラーが起こる可能性もある。

- 目的のメソッドやプロパティを見つけることが若干面倒

    - 同じような働きをするメソッド、プロパティが多すぎて目的のものを探しづらい。
    - プリミティブなものがあまりない。
