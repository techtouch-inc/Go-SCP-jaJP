正規表現
===================

正規表現は、検索やバリデーションをするために広く使用されている強力なツールです。Web アプリケーションでは、入力のバリデーション（例： 電子メールアドレス）によく使われます。

> 正規表現は、文字列の集合を記述するための表記法です。
> ある文字列が正規表現で記述された集合に含まれる場合、その正規表現が文字列に一致すると言います。([出典][1])

正規表現を使いこなすのが難しいことはよく知られています。
ちょっとしたバリデーションでの利用が [DoS][2] につながることがあります。

Go の作者はそれを真摯に受け取め、ほかのプログラミング言語とは異なり、[regex standard package][4] として [RE2][3] を実装することにしました。

## RE2 の意義

> RE2 は、信頼できないユーザによって正規表現を利用されたとしても、リスクなく扱えるという明確な目標を持って設計・実装されました。([ソース][10])

セキュリティを考慮しつつ、RE2 はパーサ、コンパイラ、実行エンジンが利用可能なメモリを制限することで、線形時間性能とグレースフルフェイルを保証します。

## 正規表現 DoS (ReDoS)

> 正規表現 DoS（ReDoS）とは、アルゴリズム複雑性を利用したサービス拒否（DoS）を誘発する攻撃です。
> ReDoS攻撃は、入力サイズに対して指数関数的に評価に長い時間を要する正規表現によって引き起こされます。
> これは、使用されている正規表現の実装に起因するものです。
> 例えば、再帰的なバックトラックなどが原因です。([ソース][8])

詳しくは [Diving Deep into Regular Expression Denial of Service（ReDoS）in Go][8] という記事を読むとよいでしょう。
最も人気のあるほかのプログラミング言語との比較も含まれています。本章では、実際のユースケースに焦点を当てます。

何らかの理由で、サインアップフォームで提供された電子メールアドレスの妥当性を確認するための正規表現を探しているとします。ざっと探したところ、次のようなものが見つかりました。
[RegEx for email validation at RegExLib.com][9]：

```
^([a-zA-Z0-9])(([\-.]|[_]+)?([a-zA-Z0-9]+))*(@){1}[a-z0-9]+[.]{1}(([a-z]{2,3})|([a-z]{2,3}[.]{1}[a-z]{2,3}))$
```

この正規表現に対して `john.doe@somehost.com` をマッチさせようとするとまさに探しているものだと確信できるかもしれません。次のようなもものを思い付くでしょう。

```go
package main

import (
    "fmt"
    "regexp"
)

func main() {
    testString1 := "john.doe@somehost.com"
    testString2 := "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"
    regex := regexp.MustCompile("^([a-zA-Z0-9])(([\\-.]|[_]+)?([a-zA-Z0-9]+))*(@){1}[a-z0-9]+[.]{1}(([a-z]{2,3})|([a-z]{2,3}[.]{1}[a-z]{2,3}))$")

    fmt.Println(regex.MatchString(testString1))
    // expected output: true
    fmt.Println(regex.MatchString(testString2))
    // expected output: false
}
```

これは問題ありません。

```
$ go run src/redos.go
true
false
```

しかし、たとえば JavaScript で開発していた場合はどうでしょうか？


```JavaScript
const testString1 = 'john.doe@somehost.com';
const testString2 = 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!';
const regex = /^([a-zA-Z0-9])(([\-.]|[_]+)?([a-zA-Z0-9]+))*(@){1}[a-z0-9]+[.]{1}(([a-z]{2,3})|([a-z]{2,3}[.]{1}[a-z]{2,3}))$/;

console.log(regex.test(testString1));
// expected output: true
console.log(regex.test(testString2));
// expected output: hang/FATAL EXCEPTION

```

この場合、**実行は永遠にハングアップします**（少なくともこのプロセスは）。**再実行をかけなければこれ以降のサインアップが不可能であることを意味します。**
結果として**ビジネス損失**となります。

## 何が足りないのか？

Perl、PHP、Python、JavaScript などのほかのプログラミング言語の経験がある場合は、正規表現がサポートする機能に関する違いに気がついているかもしれません。

RE2 では、[Backreferences][5] や [Lookaround][6] のようなバックトラックによる解決法だけが知られている構文がサポートされていません。

次のような問題を考えてみましょう。

文字列が正しい HTML フォーマットである。

a) 開閉タグ名が一致する。

b) その間にテキストが存在する。

b) の条件を満たすのは簡単で、`.*?` です。しかし、要件 a) を満たすことは困難です。なぜなら、タグが正しく閉じているかは、開始タグとして何がマッチしたかに依存するからです。これはまさにバックトレースが可能にしてくれることです。以下の JavaScript の実装をご覧ください。

```JavaScript
const testString1 = '<h1>Go Secure Coding Practices Guide</h1>';
const testString2 = '<p>Go Secure Coding Practices Guide</p>';
const testString3 = '<h1>Go Secure Coding Practices Guid</p>';
const regex = /<([a-z][a-z0-9]*)\b[^>]*>.*?<\/\1>/;

console.log(regex.test(testString1));
// expected output: true
console.log(regex.test(testString2));
// expected output: true
console.log(regex.test(testString3));
// expected output: false

```

`\1` は、`([A-Z][A-Z0-9]*)` で捕捉した値を保持します。

これは、Go でも行えるとは思ってはいけません。

```go
package main

import (
    "fmt"
    "regexp"
)

func main() {
    testString1 := "<h1>Go Secure Coding Practices Guide</h1>"
    testString2 := "<p>Go Secure Coding Practices Guide</p>"
    testString3 := "<h1>Go Secure Coding Practices Guid</p>"
    regex := regexp.MustCompile("<([a-z][a-z0-9]*)\b[^>]*>.*?<\/\1>")

    fmt.Println(regex.MatchString(testString1))
    fmt.Println(regex.MatchString(testString2))
    fmt.Println(regex.MatchString(testString3))
}

```

上記の Go ソースコードサンプルを実行すると、以下のようなエラーが発生するはずです。

```
$ go run src/backreference.go
# command-line-arguments
src/backreference.go:12:64: unknown escape sequence
src/backreference.go:12:67: non-octal character in escape sequence: >
```

これらのエラーを修正するために、次のような正規表現を思い付くかもしれません。

```
<([a-z][a-z0-9]*)\b[^>]*>.*?<\\/\\1>
```

得られるのは以下です。

```
go run src/backreference.go
panic: regexp: Compile("<([a-z][a-z0-9]*)\b[^>]*>.*?<\\/\\1>"): error parsing regexp: invalid escape sequence: `\1`

goroutine 1 [running]:
regexp.MustCompile(0x4de780, 0x21, 0xc00000e1f0)
        /usr/local/go/src/regexp/regexp.go:245 +0x171
main.main()
        /go/src/backreference.go:12 +0x3a
exit status 2
```

フルスクラッチで開発すれば、いくつかの機能の不足を補うためのすばらしい回避策が見つかるかもしれません。一方、既存のソフトウェアを利用する場合、標準的な正規表現パッケージの代替となるフル機能の代替品を探すことになります。おそらくいくつか見つかるでしょう（たとえば [dlclark/regexp2][7]）。
ただし、おそらく RE2 の安全性と線形時間計算性能を捨てることになることを、念頭においてください。

[1]: https://swtch.com/~rsc/regexp/regexp1.html
[2]: #regular-expression-denial-of-service-redos
[3]: https://github.com/google/re2/wiki
[4]: https://golang.org/pkg/regexp/
[5]: https://www.regular-expressions.info/backref.html
[6]: https://www.regular-expressions.info/lookaround.html
[7]: https://github.com/dlclark
[8]: https://www.checkmarx.com/2018/05/07/redos-go/
[9]: http://regexlib.com/REDetails.aspx?regexp_id=1757
[10]: https://github.com/google/re2/wiki/WhyRE2
