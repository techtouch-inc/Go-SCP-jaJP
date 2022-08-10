認証データのバリデーション・保存
==========================================

## バリデーションと保存
----------

このセクションの主題は、「認証データの保存」です。
ユーザーアカウントのデータベースがインターネット上に流出することは、望ましいことではありません。もちろん、このような事態が確実に起こるとは言えません。しかし、万が一そのような事態になった場合、パスワードなどの認証情報が適切に保管されていれば、被害の拡大を避けることができます。

まず、"_all authentication controls should fail　securely_"（あらゆる認証制御は安全に失敗すべき）ということをはっきりさせましょう。
また「認証とパスワード管理」のすべてのセクションを読むことをお勧めします。
認証情報の誤りについてのレポートバックやログの処理方法に関する推奨事項を扱っています。

もう 1 つの予備的な推奨事項は、一連の認証の実装（最近の Google のような）では、あらゆる入力値は信頼できるシステム（サーバーなど）においてバリデーションされることです。

## パスワードを安全に保管する：理論編

パスワードの保存について説明します。

パスワードは平文で保存する必要はありません。しかしユーザーが認証のたびに同じトークンを利用しているのかを、毎回バリデーションしなければいけません。

そこで、セキュリティ上の理由から、「一方通行」な関数 `H` が必要になります。
 `p1` と `p2` が異なるパスワードの場合、 `H(p1)` もまた、H(p2)`とは異なるという性質を持ちます[^1]。

これは数学チックに感じますか？
次の要件に注意してください： `H` は次のような関数でなければなりません。
`H-¹(H(p1))` が `p1` と等しくなるような関数 `H-¹` は存在しない。これはつまり
ありとあらゆる `p` 試さない限り、元の `p1` を見つけられないということです。


`H` が一方通行であるなら、アカウント漏洩の本当の問題は何でしょうか？

もし考えられるすべてのパスワードを知っているなら、そのすべてのハッシュ値を事前に計算し
レインボーテーブル攻撃を実行できます。
> 訳者注：　ハッシュ関数がわかっているときに、大量のハッシュ値の候補を効率的に照会するような攻撃です。例え盗まれた情報がハッシュ値だけだとしてもパスワードが見つかってしまいます。検索するとわかりやすい記事も沢山見つかるのでロジックの説明は省きますが、ロジックを知ると下記に書いてある`salt` を利用することでこの攻撃が効かなくなりそうだと思えるはずです。

パスワードはユーザーにとって管理するのが難しいことは、すでにお話したとおりです。また、ユーザーはパスワードを再利用してしまうだけでなく、覚えやすいもの、つまり推測しやすいものを使う傾向があります。

これを避けるにはどうしたらよいのでしょうか。

2 人の異なるユーザーが同じパスワード `p1` を提供した場合、私たちは
は異なるハッシュ値を格納すると言うのがポイントです。
不可能に聞こえるかもしれませんが、`salt` を使えば良いのです。**ユーザーパスワード毎に別** の疑似乱数 `salt` を `p1` の前に追加し、結果としてハッシュを次のように計算します。`H(salt + p1)`.

パスワードストアの各エントリでは、計算結果のハッシュと
`salt` を平文で保存します。`salt` は非公開である必要はありません。

最後の推奨事項です。
* 非推奨のハッシュアルゴリズム (例： SHA-1, MD5, etc) は使わないでください。
* [擬似乱数生成器のセクション][1]をお読みください。

次のコード・サンプルは、この動作の基本的な例を示しています。

```go
package main

import (
    "crypto/rand"
    "crypto/sha256"
    "database/sql"
    "context"
    "fmt"
)

const saltSize = 32

func main() {
    ctx := context.Background()
    email := []byte("john.doe@somedomain.com")
    password := []byte("47;u5:B(95m72;Xq")

    // create random word
    salt := make([]byte, saltSize)
    _, err := rand.Read(salt)
    if err != nil {
        panic(err)
    }

    // let's create SHA256(salt+password)
    hash := sha256.New()
    hash.Write(salt)
    hash.Write(password)
    h := hash.Sum(nil)

    // this is here just for demo purposes
    //
    // fmt.Printf("email   : %s\n", string(email))
    // fmt.Printf("password: %s\n", string(password))
    // fmt.Printf("salt    : %x\n", salt)
    // fmt.Printf("hash    : %x\n", h)

    // you're supposed to have a database connection
    stmt, err := db.PrepareContext(ctx, "INSERT INTO accounts SET hash=?, salt=?, email=?")
    if err != nil {
        panic(err)
    }
    result, err := stmt.ExecContext(ctx, h, salt, email)
    if err != nil {
        panic(err)
    }

}
```

ただし、上記にはいくつかの欠点があるのでそのまま使用すべきではありません。理論を説明するためだけのサンプルです。次の章では、実際の場面で正しくパスワードソルトを利用する方法について説明します。

## パスワードを安全に保管する：実践編

暗号技術で最も重要な格言の 1 つは**決して自分で暗号化しないこと**です。
アプリケーション全体が危険にさらされる可能性があります。これは
デリケートで複雑なテーマです。うれしいことに暗号技術には専門家によって審査され、承認されたツールや規格が提供されています。再発明するのではなく、これらを利用することが大事です。

パスワード保存の場合、[OWASP][2] によって推奨されるハッシュアルゴリズムは、[`bcrypt`][2]、[`PDKDF2`][3]、[`Argon2`][4]、[`scrypt`][5]です。
これらは、堅牢な方法でパスワードのハッシュ化とソルティングを行います。Go の作者は
標準ライブラリに含まれない暗号のための拡張パッケージを提供しています。
このライブラリは、前述のほとんどのアルゴリズムを含みます。`go get` でダウンロードできます。

```
go get golang.org/x/crypto
```

次の例は、bcrypt を使用する方法を示していますが、広い場面でこれは十分な効果があるはずです。bcrypt の利点はシンプルに使用できることで、結果的にエラーの可能性が低く抑えられることです。

```go
package main

import (
    "database/sql"
    "context"
    "fmt"

    "golang.org/x/crypto/bcrypt"
)

func main() {
    ctx := context.Background()
    email := []byte("john.doe@somedomain.com")
    password := []byte("47;u5:B(95m72;Xq")

    // Hash the password with bcrypt
    hashedPassword, err := bcrypt.GenerateFromPassword(password, bcrypt.DefaultCost)
    if err != nil {
        panic(err)
    }

    // this is here just for demo purposes
    //
    // fmt.Printf("email          : %s\n", string(email))
    // fmt.Printf("password       : %s\n", string(password))
    // fmt.Printf("hashed password: %x\n", hashedPassword)

    // you're supposed to have a database connection
    stmt, err := db.PrepareContext(ctx, "INSERT INTO accounts SET hash=?, email=?")
    if err != nil {
        panic(err)
    }
    result, err := stmt.ExecContext(ctx, hashedPassword, email)
    if err != nil {
        panic(err)
    }
}
```

Bcrypt は、平文のパスワードとハッシュ化したパスワードを比較するための簡単で安全な方法も提供します。

 ```go
 ctx := context.Background()

 // credentials to validate
 email := []byte("john.doe@somedomain.com")
 password := []byte("47;u5:B(95m72;Xq")

// fetch the hashed password corresponding to the provided email
record := db.QueryRowContext(ctx, "SELECT hash FROM accounts WHERE email = ? LIMIT 1", email)

var expectedPassword string
if err := record.Scan(&expectedPassword); err != nil {
    // user does not exist

    // this should be logged (see Error Handling and Logging) but execution
    // should continue
}

if bcrypt.CompareHashAndPassword(password, []byte(expectedPassword)) != nil {
    // passwords do not match

    // passwords mismatch should be logged (see Error Handling and Logging)
    // error should be returned so that a GENERIC message "Sign-in attempt has
    // failed, please check your credentials" can be shown to the user.
}
 ```

パスワードのハッシュ化と比較に関するオプションやパラメータに不安がある場合、デフォルト値が安全な専用のサードパーティに任せてしまうことをお勧めします。
常にアクティブにメンテナンスされているパッケージを選択し、既知の問題がないかを確認することを忘れないでください。

* [passwd][6] - パスワードのハッシュ化と比較のデフォルトセーフな抽象化を提供する Go パッケージです。オリジナルの go bcrypt 実装、argon2、scrypt、パラメータマスキング、key'ed(uncrackable)ハッシュをサポートします。

[^1]: ハッシュ関数には Collisions の課題がありますが、推奨されているハッシュ関数は Collisions の確率が非常に小さいです。

[1]: ../cryptographic-practices/pseudo-random-generators.md
[2]: https://www.owasp.org/index.php/Password_Storage_Cheat_Sheet
[3]: https://godoc.org/golang.org/x/crypto/bcrypt
[4]: https://github.com/p-h-c/phc-winner-argon2
[5]: https://godoc.org/golang.org/x/crypto/pbkdf2
[6]: https://github.com/ermites-io/passwd
