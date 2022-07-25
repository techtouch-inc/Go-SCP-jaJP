擬似乱数の生成
========================

OWASP Secure Coding Practices には、実に複雑に思える以下のガイドラインがあります。
「すべての乱数、ランダムなファイル名、ランダムな GUID、およびランダムな文字列は、推測されたくなければ暗号化モジュール内の認められた乱数生成器を利用する必要があります」

では、「乱数」について説明します。

暗号技術は、ある種のランダム性に依存していますが、ほとんどのプログラミング言語はその厳密さゆえに乱数を正しく扱うために、擬似的な乱数生成器が用意されています。
たとえば、[Go's math/rand][1]も例外ではありません。

「トップレベル関数」と書かれているもののドキュメントを注意深く読むと良いでしょう。
Float64 や Int のような「トップレベル関数」は、**決定論的な一連の値**を生成するデフォルトの共有ソースを使用します。

それがどういうことか見てみましょう：

```go
package main

import "fmt"
import "math/rand"

func main() {
    fmt.Println("Random Number: ", rand.Intn(1984))
}
```

このプログラムを何度実行しても、まったく同じ数字になります。
なぜでしょうか？

```bash
$ for i in {1..5}; do go run rand.go; done
Random Number:  1825
Random Number:  1825
Random Number:  1825
Random Number:  1825
Random Number:  1825
```

理由は、[Goのmath/rand][1] が決定論的な擬似乱数生成器だからです。
ほかの多くのものと同様に、シードと呼ばれるソースを使用します。このシード**だけ**が
決定論的擬似乱数生成器のランダム性に責任を負います。
シードが既知または予測可能な場合、同じシードを利用して数列が再生成されてしまうことがあります。

今回の例は、[math/rand Seed 関数][3]を使って、プログラムの実行ごとに 5 つの異なる値を取得するように修正もできます。
> 役者注： https://github.com/golang/go/blob/654b03167f7d887d09a2d25bb3d1b4db9c8ae4f8/src/math/rand/rng.go seed 生成ロジックのソースコード。Seed 関数を呼ぶと乱数列が再生成される。結局は実行時に異なる値、時刻情報とか使わないと。

しかし、ここは暗号のプラクティスのセクションですから、[Go's crypto/rand package][4] に従うべきでしょう。

```go
package main

import "fmt"
import "math/big"
import "crypto/rand"

func main() {
    rand, err := rand.Int(rand.Reader, big.NewInt(1984))
    if err != nil {
        panic(err)
    }

    fmt.Printf("Random Number: %d\n", rand)
}
```

[crypto/rand][4] の実行が [math/rand][1] より遅いことに気がつくかもしれません。
最速のアルゴリズムが常に最も安全とは限らないので、想像の範囲内です。Crypto の
rand は安全に利用できます。どのように安全かと言えば、たとえば crypto/rand は OS のランダム性を使用しているため、シードを与えることができません。
これによって開発者の誤用を防ぐことができます。

```bash
$ for i in {1..5}; do go run rand-safe.go; done
Random Number: 277
Random Number: 1572
Random Number: 1793
Random Number: 1328
Random Number: 1378
```

擬似乱数の誤用がどのように悪用されるか興味があるのなら、アプリケーションが、ユーザーのサインアップ時にデフォルトのパスワードを[Go's math/rand][1]を利用した擬似乱数を使って生成する場合を想像してみましょう。
そうです、ユーザーのパスワードが予測できてしまうのです。

[1]: https://golang.org/pkg/math/rand/
[2]: https://golang.org/pkg/math/rand/#pkg-overview
[3]: https://golang.org/pkg/math/rand/#Seed
[4]: https://golang.org/pkg/crypto/rand/
