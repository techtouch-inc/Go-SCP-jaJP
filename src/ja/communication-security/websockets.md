Webソケット
==========

WebSocket は、HTML5 のために開発された新しいブラウザ機能で、完全な対話型アプリケーションを可能にします。
WebSocket を使用すると、ブラウザとサーバの両方が、ロングポーリングやコメットなどを使用せずに 1 つの TCP ソケットを介して非同期にメッセージを送信できます。

基本的に、WebSocket は、クライアントとサーバの間の標準的な双方向 TCP ソケットです。
このソケットは、最初は通常の HTTP 接続で始まり、HTTP ハンドシェイクの後、TCP ソケットにアップグレードされます。
ハンドシェイク後は、どちらもデータを送信できます。

## Origin ヘッダ

HTTP WebSocket ハンドシェイクの `Origin` ヘッダは、WebSocket が受け入れる接続が信頼できるオリジンドメインからのものであることを保証するために使用されます。
このヘッダの仕様を強制しないと、Cross Site Request Forgery (CSRF)につながる可能性があります。

最初の HTTP WebSocket ハンドシェイクで `Origin` ヘッダを検証するのは、サーバの責任です。
もしサーバが最初の HTTP WebSocket ハンドシェイクでオリジン ヘッダを検証しない場合
WebSocket サーバは任意のオリジンからの接続を受け入れる可能性があります。

以下の例では、`Origin` ヘッダをチェックし、
攻撃者が CSWSH (Cross-Site WebSocket Hijacking)を実行するのを防ぎます。
> 訳者注: https://developer.mozilla.org/ja/docs/Web/API/WebSockets_API/Writing_WebSocket_servers

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

WebSocket の通信チャネルは、暗号化されていない TCP または暗号化された TLS の上で確立できます。

暗号化されていない WebSocket が使用される場合、URI スキームは `ws://` で、そのデフォルトのポートは `80` 番です。
TLS の WebSocket を使用する場合、URI スキームは `wss://` で、デフォルトのポート番号は
`443` 番です。

WebSocket を調べる場合、元の接続を調べて TLS を使用しているか、それとも暗号化されていない状態で送信されているかを考慮する必要があります。

このセクションでは、接続が HTTP から WebSocket にアップグレードされたときに送信される情報と、正しく処理されない時に生じるリスクについて説明します。
最初の例では、通常の HTTP 接続が WebSocket 接続にアップグレードされる様子を示します。

![HTTP Cookie Leak](img/w2_1.png)

ヘッダにセッション用の Cookie が含まれていることに注意してください。機密情報が漏れないように、接続をアップグレードする際には、次の画像に示すように TLS を使用しましょう。

![HTTP Cookie TLS](img/ws_tls_upgrade.png)

後者の例では、接続のアップグレード要求が SSL を使用し、さらに Websocket も SSL を使用します。

![Websocket SSL](img/wss_secure.png)

## 認証と認可

WebSocket は認証や認可を処理しません。
つまり、セキュリティを確保するために、Cookie、HTTP 認証、TLS 認証のようなメカニズムを使用しなければいけません。
これに関するより詳細な情報は[認証][1]と[アクセス制御][2]を参照してください。

## 入力のサニタイズ

信頼できないソースから発信されたデータだけでなく、すべてのデータは適切にサニタイズされ、エンコードされる必要があります。
これらのトピックの詳細は[サニタイズ][3] と [出力のエンコーディング][4] を参照してください。

[1]: ../authentication-password-management/README.md
[2]: ../access-control/README.md
[3]: ../input-validation/sanitization.md
[4]: ../output-encoding/README.md
