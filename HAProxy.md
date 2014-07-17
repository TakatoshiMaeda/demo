# HAProxy

## HAProxyとは
[HAProxy](http://haproxy.1wt.eu/) はソフトウェアロードバランサの実装です。
オープンソースで開発されており、ライセンスは GPLv2 です。
TCP と HTTP のプロトコルのサポートに加えて、（2013-07-15 現在の）開発版であるバーション1.5-dev12では SSL Termination の機能が追加されました。

## 本文書の概要
AWS上で運用しているアプリケーションサーバの前段にロードバランサを設置することを想定して実施した調査結果です。

## 文書一覧
* [[HAProxy の各種設定方法]]
* [[ロードバランサ性能テスト---HAProxy]]
* [[ロードバランサ性能テスト---HAProxy-with-SSL]]
* [[HAProxyの可用性に関する調査]]

## リンク
* [利用実績](http://haproxy.1wt.eu/they-use-it.html)