セッション管理
===========

このセクションでは、OWASP's Secure Coding Practices に従ってセッション管理の最も重要な側面について説明します。具体例とそれに沿った、プラクティスの背後にある論理的根拠の概要を説明します。
このセッションでテキストと合わせて分析するための完全なソースコードが入っているフォルダがあります。
このセクションで分析するプログラムの セッションプロセスの流れは以下の画像の通りです。

![SessionManagementOverview](img/SessionManagementOverview.png)

セッション管理において、アプリケーションはあくまでもサーバーのセッション管理コントロールを使用するべきで、セッションの作成は信頼できるシステムで行われるべきです。

コード例において、アプリケーションは信頼できるシステム上でJWTを使用したセッションを作成します。
これは以下の関数で行われます。

```go
// create a JWT and put in the clients cookie
func setToken(res http.ResponseWriter, req *http.Request) {
  ...
}
```
We must ensure that the algorithms used to generate our session identifier are
sufficiently random, to prevent session brute forcing.

セッション識別子を生成するために使用されるアルゴリズムは、
ブルートフォース（総当り）を防ぐために、十分にランダムであるものを利用してください。

```go
...
token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
signedToken, _ := token.SignedString([]byte("secret")) //our secret
...
```

Now that we have a sufficiently strong token, we must also set
the `Domain`, `Path`, `Expires`, `HTTP only`, `Secure` for our cookies. In this
case the `Expires` value is in this example set to 30 minutes since we are
considering our application a low-risk application.

十分に強力なトークンができたので、クッキーの `Domain`, `Path`, `Expires`, `HTTP only`, `Secure` を指定しましょう。
今回の例では低リスクのアプリケーションを想定しているため、`Expires`の値を30分に設定しています。

```go
// Our cookie parameter
cookie := http.Cookie{
    Name: "Auth",
    Value: signedToken,
    Expires: expireCookie,
    HttpOnly: true,
    Path: "/",
    Domain: "127.0.0.1",
    Secure: true
}

http.SetCookie(res, &cookie) //Set the cookie
```

サインインすると、常に新しいセッションが生成されます。古いセッションはたとえ有効期限が切れていなくても再利用されてはいけません。
また、セッションハイジャックを防止するために`Expire`パラメータを使用して、定期的にセッションを終了強制します。
クッキーのもう一つの重要な側面は、同一ユーザー名による同時ログインを禁止することです。
ログインしているユーザーのリストを保持し、新しいログインユーザー名をこのリストと比較する。このアクティブなユーザーの一覧は通常、データベースに保存されます。

セッション識別子は、決してURLの中で公開してはいけません。セッション識別子は
HTTP クッキーヘッダになくてはいけません。好ましくない例として、セッション識別子を GET パラメータとして受け渡してしまうような場合があります。
またセッション情報は、他の認可されないユーザーによる不正なアクセスから保護する必要があります。


Regarding HTTP to HTTPS connection changes, special care should be taken to
prevent Man-in-the-Middle (MITM) attacks that sniff and potentially hijack the
user's session. The best practice regarding this issue, is to use HTTPS in all
requests. In the following example our server is using HTTPS.


HTTP から HTTPS への接続変更については、ユーザーのセッション情報を窃取ハイジャックするようなMITM（Man-in-the-Middle）攻撃を防ぐために特に注意が必要です。
この問題に関するベストプラクティスは、すべてのセッションでHTTPSを使用することです。
次の例では、サーバーはHTTPSを使用しています。

```go
err := http.ListenAndServeTLS(":443", "cert/cert.pem", "cert/key.pem", nil)
if err != nil {
  log.Fatal("ListenAndServe: ", err)
}
```

機密性の高い、または重要な操作の場合は、セッション単位ではなく、リクエスト単位で生成されるトークンを使用する必要があります。
トークンはブルートフォースから保護するため常に十分にランダムかつ十分に長いものが使われるようにしてください。

セッション管理で考慮すべき最後の側面は、**ログアウト** です。
アプリケーションは、認証の元すべてのページからのログアウト方法を提供すべきで、関連するコネクションやセッションを完全に終了させる必要があります。
この例では、ユーザがログアウトすると、クッキーはクライアント側で削除されます。
ユーザーセッション情報が格納される場所においても同様に削除処理が実行されるべきです。

```go
...
cookie, err := req.Cookie("Auth") //Our auth token
if err != nil {
  res.Header().Set("Content-Type", "text/html")
  fmt.Fprint(res, "Unauthorized - Please login <br>")
  fmt.Fprintf(res, "<a href=\"login\"> Login </a>")
  return
}
...
```
全てのコードサンプルは次にあります。 [session.go](session.go)
