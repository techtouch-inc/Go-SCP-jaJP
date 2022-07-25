クロスサイトリクエストフォージェリ
==========================

[OWASP の定義][1]によると、Cross-Site Request Forgery (CSRF)とは、エンドユーザーに認証済みの Web アプリケーション上で望ましくない動作を実行させる攻撃であるとされます。([ソース][1])

CSRF 攻撃は、データの盗難に焦点を当てたものではありません。その代わり、状態を変更するリクエストに狙いを定めています。攻撃者はちょっとしたソーシャルエンジニアリング（メールやチャットでのリンクの共有など）を用いて、ユーザーを騙して、アカウントの復旧用メールアドレスを変更するなどの望まないアクションを Web サービスに対して実行させることができます。

## 攻撃のシナリオ

たとえば、`foo.com` が HTTP `GET` リクエストを使って、次のようにアカウント復旧用メールアドレスを設定できるとしましょう。


```
GET https://foo.com/account/recover?email=me@somehost.com
```

単純な攻撃のシナリオは以下のようになります。

1. 被害者が https://foo.com で認証する。
2. 攻撃者は被害者に以下のリンク付きのチャットメッセージを送信する。
   ```
   https://foo.com/account/recover?email=me@attacker.com
   ```
3. 被害者のアカウント復旧用メールアドレスは、`me@attacker.com`に変更され、攻撃者がアカウントを完全にコントロールできるようになる。

## 問題点

HTTP メソッドを `GET` から `POST` (または別のメソッド） に変更しても、問題は解決しません。
シークレット Cookie、URL リライト、HTTPS を使っても解決しません。

この攻撃が可能なのは、サーバが正当なユーザーセッション中のワークフロー(ナビゲーション）で行われたリクエストと、悪意あるリクエストを区別できないためです。

## 解決

### 理論編

前述したように、CSRF は状態を変更するリクエストを対象とします。Web
アプリケーションでは、ほとんどの場合、フォームの送信によって発行された `POST` リクエストを意味します。

このシナリオでは、ユーザーが最初にフォームを表示するページを要求したとき、サーバは [nonce][2] (一度だけ使われる任意の数字） を計算します。
このトークンはフォームのフィールドとして含まれます (ほとんどの場合、この
フィールドは hidden ですが、必須ではありません）。

次に、フォームが送信されると、hidden フィールドはほかのユーザー入力と一緒にサーバに送信されます。
サーバはトークンがリクエストデータの一部であるかどうかを検証し、トークンがリクエストデータの一部に含むか検証し、有効かどうかを判断します。

この nonce/token は、以下の要件に従うべきです。

* ユーザーセッションごとに異なること
* 大きなランダム値であること
* 暗号的に安全な乱数生成器によって生成されること

**Note:** HTTP `GET` リクエストは状態を変化させないことが期待されていますが (そうあるべきと言われています）望ましくないプログラミングによって、実際にはリソースを変更できます。そのため、CSRF 攻撃の標的になる可能性があります。

API については、`PUT` と `DELETE` もまた CSRF 攻撃の一般的な標的です。

### 実践編

これをすべて手作業で行うのは、エラーが発生しやすいので、良いアイデアとは言えません。

ほとんどの Web アプリケーションフレームワークは、すぐに使える解決策を提供しているため、それを有効にすることをお勧めします。もしあなたがフレームワークを使用していないなら、使用しましょう。

次の例は、Go Web アプリケーション向けの[Gorilla Web ツールキット][3]の一部です。
GitHub の [gorilla/csrf][4] にあります。

```go
package main

import (
    "net/http"

    "github.com/gorilla/csrf"
    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()
    r.HandleFunc("/signup", ShowSignupForm)
    // All POST requests without a valid token will return HTTP 403 Forbidden.
    r.HandleFunc("/signup/post", SubmitSignupForm)

    // Add the middleware to your router by wrapping it.
    http.ListenAndServe(":8000",
        csrf.Protect([]byte("32-byte-long-auth-key"))(r))
    // PS: Don't forget to pass csrf.Secure(false) if you're developing locally
    // over plain HTTP (just don't leave it on in production).
}

func ShowSignupForm(w http.ResponseWriter, r *http.Request) {
    // signup_form.tmpl just needs a {{ .csrfField }} template tag for
    // csrf.TemplateField to inject the CSRF token into. Easy!
    t.ExecuteTemplate(w, "signup_form.tmpl", map[string]interface{}{
        csrf.TemplateTag: csrf.TemplateField(r),
    })
    // We could also retrieve the token directly from csrf.Token(r) and
    // set it in the request header - w.Header.Set("X-CSRF-Token", token)
    // This is useful if you're sending JSON to clients or a front-end JavaScript
    // framework.
}

func SubmitSignupForm(w http.ResponseWriter, r *http.Request) {
    // We can trust that requests making it this far have satisfied
    // our CSRF protection requirements.
}
```

OWASP が詳細な [Cross-Site Request Forgery (CSRF) Prevention Cheat Sheet][5] を公開していますので、一読されることをお勧めします。

[1]: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
[2]: https://en.wikipedia.org/wiki/Cryptographic_nonce
[3]: http://www.gorillatoolkit.org/
[4]: https://github.com/gorilla/csrf
[5]: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html#synchronizer-token-pattern
