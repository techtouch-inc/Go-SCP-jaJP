ウェブソケット
==========

WebSocketは、HTML 5のために開発された新しいブラウザ機能で、完全な対話型アプリケーションを可能にします。
WebSocketを使用すると、ブラウザとサーバーの両方が、ロングポーリングやコメットなどを使用せずに1つのTCPソケットを介して非同期にメッセージを送信することができます。

基本的に、WebSocket は、クライアントとサーバーの間の標準的な双方向 TCP ソケットです。
クライアントとサーバーの間の標準的な双方向 TCP ソケットです。このソケットは、最初は通常のHTTP接続で始まり、HTTPハンドシェイクの後、TCP ソケットにアップグレードされます。
ハンドシェイク後は、どちらもデータを送信できるようになります。

## Origin ヘッダー

HTTP WebSocket ハンドシェイクの `Origin` ヘッダーは、WebSocket が受け入れる接続が信頼できるオリジンドメインからのものであることを保証するために使用されます。
このヘッダーの仕様を強制しないと、Cross Site Request Forgery (CSRF)につながる可能性があります。

最初の HTTP WebSocket ハンドシェイクで `Origin` ヘッダーを検証するのは、サーバーの責任です。
もしサーバーが最初の HTTP WebSocket ハンドシェイクでオリジン ヘッダーを検証しない場合
WebSocket サーバーは任意のオリジンからの接続を受け入れることができてしまいます。

以下の例では、`Origin` ヘッダーのチェックし、
攻撃者が CSWSH (Cross-Site WebSocket Hijacking)を実行するのを防ぎます。

![HTTP ヘッダリーク](img/w1_1.png)

アプリケーションは `Host` と `Origin` を検証して、リクエストの `Origin` が信頼できる `Host` であることを確認し、そうでない場合は接続を拒否すべきです。

シンプルなチェック方法を次のスニペットで紹介します。

```go
//Compare our origin with Host and act accordingly
if r.Header.Get("Origin") != "http://"+r.Host {
  http.Error(w, "Origin not allowed", 403)
    return
} else {
    websocket.Handler(EchoHandler).ServeHTTP(w, r)
}
```

## 機密性と完全性

WebSocketの通信チャネルは、暗号化されていないTCPまたは暗号化されたTLSの上で確立することができます。

暗号化されていないWebSocketが使用される場合、URIスキームは `ws://` で、そのデフォルトのポートは `80` 番です。
TLSのWebSocketを使用する場合、URIスキームは `wss://` で、デフォルトのポート番号は
`443`番です.

WebSocket を調べる場合、元の接続を調べて TLS を使用しているか、それとも暗号化されていない状態で送信されているかを考慮する必要があります。

このセクションでは、接続が HTTP から WebSocket にアップグレードされたときに送信される情報と、正しく処理されない時に生じるリスクについて説明します。
まず最初の例では、通常の HTTP 接続が WebSocket 接続にアップグレードされる様子を示しています。

![HTTP Cookie Leak](img/w2_1.png)

ヘッダーにセッション用のクッキーが含まれていることに注意してください。機密情報が漏れないようにするために
機密情報が漏れないように、接続をアップグレードする際には、次の画像に示すように TLS を使用しましょう。

![HTTP Cookie TLS](img/ws_tls_upgrade.png)

In the latter example, our connection upgrade request is using SSL, as well as
our WebSocket:
後者の例では、接続のアップグレード要求がSSLを使用し、さらに Websocket も SSL
を使用しています。

![Websocket SSL](img/wss_secure.png)

## 認証と認可

WebSocket は認証や認可を処理しません。
つまり、セキュリティを確保するために、Cookie、HTTP 認証、TLS 認証のようなメカニズムを使用しなければいけません。
これに関するより詳細な情報は[認証][1]と[アクセス制御][2]を参照してください。

## 入力のサニタイズ

信頼できないソースから発信されたデータだけでなく、全てのデータは適切にサニタイズされ、エンコードされる必要があります。
これらのトピックの詳細は[サニタイズ][3] と [出力のエンコーディング][4] を参照してください。

[1]: ../authentication-password-management/README.md
[2]: ../access-control/README.md
[3]: ../input-validation/sanitization.md
[4]: ../output-encoding/README.md
