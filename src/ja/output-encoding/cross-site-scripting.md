XSS - クロスサイトスクリプティング
==========================

開発者の多くは、XSSという単語を聞いたことがあっても、実際に悪用しようとしたことはないでしょう。

XSSは、2003年から [OWASP Top 10][0] のセキュリティリスクとして取り上げられており、今でもよくある脆弱性の一つです。[2013年版][1]では、攻撃ベクトル、セキュリティの弱点、技術的な影響、ビジネスへの影響についてなどかなり詳しく書かれています。

> ユーザーから提供された入力値を利用して値を出力する場合、エスケープ処理と、サーバーサイドでのバリデーションを実施しない場合XSS脆弱性が生じます。([ソース][1])

Go言語も、他の多目的プログラミング言語と同様に、XSS の脆弱性を突くために必要な条件は全て揃っています。（訳者注釈：比較的安全に対策ができるパッケージであるところの）[html/template][2] のドキュメントは明快であるにもかかわらず。[net/http][3] や [io][4] パッケージを使った単純な「hello world」の例がありふれています。実はこうしたサンプルは、XSS の脆弱性を抱えていることがあります。

以下のようなコードがあったとしましょう:

```go
package main

import "net/http"
import "io"

func handler (w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, r.URL.Query().Get("param1"))
}

func main () {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

このスニペットは、ポート `8080` で待ち受けるHTTPサーバを作成し、起動します
(`main()`)。そしてルート (`/`) へのリクエストを処理します。

リクエストを処理する `handler()` 関数は、クエリパラメータ `param1` を待ち受け、レスポンスストリーム (`w`) に書き込みます。

HTTPレスポンスヘッダ `Content-Type` が明示的に指定されていないため、[WhatWG spec][5] に従う
`http.DetectContentType` のデフォルト値が使用されます。

すると、例えば `param1` を "test" とした場合、`Content-Type` には `text/plain` が指定されたレスポンスが送信されます。

![Content-Type: text/plain][content-type-text-plain]

逆にもし `param1` の文字列の最初を "&lt;h1&gt;"、とすると`Content-Type` は `text/html` が指定されます.

![Content-Type: text/html][content-type-text-html]

`param1` を任意の HTML タグにすると同様な挙動を示すと思われるかもしれませんが、そうはなりません。
`param1` を "&lt;h2&gt;"、"&lt;span&gt;"
または"&lt;form&gt;" としても`Content-Type` は `text/html` ではなく `plain/text` になります。

ここで、`param1` を `<script>alert(1)</script>` としてみましょう。

[WhatWG spec][5]で定義された通り、`Content-Type` HTTPレスポンスヘッダは、`text/html`として送信され、`param1`の値がそのままレンダリングされてしまい、XSS(クロスサイトスクリプティング）が成功します。

![XSS - Cross-Site Scripting][cross-site-scripting]

この状況についてGoogleに相談したところ、次のように教えてくれました。

> content-typeが自動的に設定されてブラウザで表示的されることは実用上便利であり、意図通りの挙動です。プログラマーが
> `html/template`を使って適切にエスケープ処理することを期待します。

Googleは、開発者はサニタイズとコードの保護に責任を負うものだと述べています。
私たちはそのことには完全に同意します。**しかし**、セキュリティが優先される言語で、`Content-Type` のデフォルトとして`text/plain`ではないものまで勝手に指定されてしまうことが最善とは言えません。

`text/plain` や [text/template パッケージ][6] は、ユーザー入力をサニタイズしないので、XSS を回避できないということは肝に銘じましょう。


```go
package main

import "net/http"
import "text/template"

func handler(w http.ResponseWriter, r *http.Request) {
        param1 := r.URL.Query().Get("param1")

        tmpl := template.New("hello")
        tmpl, _ = tmpl.Parse(`{{define "T"}}{{.}}{{end}}`)
        tmpl.ExecuteTemplate(w, "T", param1)
}

func main() {
        http.HandleFunc("/", handler)
        http.ListenAndServe(":8080", nil)
}
```


`param1` を "&lt;h1&gt;" にすると、`Content-Type` が
`text/html` として送信されます。これがXSSの脆弱性の元になるのです。


![XSS while using text/template package][text-template-xss]


[text/template][6]を[html/template][2]のものに置き換えることで、安全に処理できるようになります。

```go
package main

import "net/http"
import "html/template"

func handler(w http.ResponseWriter, r *http.Request) {
        param1 := r.URL.Query().Get("param1")

        tmpl := template.New("hello")
        tmpl, _ = tmpl.Parse(`{{define "T"}}{{.}}{{end}}`)
        tmpl.ExecuteTemplate(w, "T", param1)
}

func main() {
        http.HandleFunc("/", handler)
        http.ListenAndServe(":8080", nil)
}
```

この場合、`param1` が "&lt;h1&gt;" の状態でも、HTTP レスポンスヘッダ `Content-Type` は `text/plain` として送信されます。

![Content-Type: text/plain while using html/template package][html-template-plain-text]

それだけでなく `param1` は適切にエンコードされた状態でブラウザなどのメディアに出力されます。

![No XSS while using html/template package][html-template-noxss]

[exploit-of-a-mom]: images/exploit-of-a-mom.png
[content-type-text-plain]: images/text-plain.png
[content-type-text-html]: images/text-html.png
[cross-site-scripting]: images/xss.png
[text-template-xss]: images/text-template-xss.png
[html-template-plain-text]: images/html-template-plain-text.png
[html-template-noxss]: images/html-template-text-plain-noxss.png

[0]: https://www.owasp.org/index.php/Category:OWASP_Top_Ten_Project
[1]: https://www.owasp.org/index.php/Top_10_2013-A3-Cross-Site_Scripting_(XSS)
[2]: https://golang.org/pkg/html/template/
[3]: https://golang.org/pkg/net/http/
[4]: https://golang.org/pkg/io/
[5]: https://mimesniff.spec.whatwg.org/#rules-for-identifying-an-unknown-mime-typ
[6]: https://golang.org/pkg/text/template/
