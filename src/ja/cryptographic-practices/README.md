暗号に関するプラクティス
======================

まずは強く明言します。
**ハッシュ化と暗号化は異なるものです**。

一般的に誤解があるようで、ほとんどの場合、ハッシュ化と暗号化は
は間違った用法で、同じように使われています。両者は異なる概念であり目的も異なります。

ハッシュとは、ソースデータから（ハッシュ）関数によって生成された文字列または数値のことです。

```
hash := F(data)
```

ハッシュ値は固定長で、入力値の小さな変動に対して大きく変化します。
（それでも衝突は起こるかもしれませんが）。優れたハッシュアルゴリズムでは、ハッシュ値を元の値に戻すことはできません[^1]。ハッシュアルゴリズムとしては、MD5 が最も有名ですが、セキュリティ的には [BLAKE2][4] が最も強く、柔軟性があると考えられています。
> 訳者注：2021年には BLAKE3 が出ていますがまだ x/crypto には採用されていなそうです。単にオブジェクトの整合性取りたいだけなら SHA-1 とか MD5 も使います。

Go の補助的な暗号ライブラリには、[BLAKE2b][5]（単に BLAKE2 のこと）と[BLAKE2s][6] の実装があます。前者は64ビット環境用に最適化されており、後者は8ビットから32ビットまでの環境に対応しています。BLAKE2 が使えない場合が利用できない場合は、SHA-256 が良い選択肢となります。

暗号ハッシュアルゴリズムにおいて、遅いことは望ましいということに注意してください。
コンピュータは年々高速化しているため、攻撃者はより多くのパスワードを試すことができるようになっています。（[クレデンシャルスタッフィング][7]および[ブルートフォースアタック][8]）。それに対抗してハッシュ関数は、少なくとも10,000回の反復的な処理を内包する、本質的に遅い関数であるべきです。

中身が何であるか知らなくても良いが、中身の正しさを確認したい時（ダウンロード後のファイルの整合性チェックなど）はハッシュ化[^2]を使うべきです。


```go
package main

import "fmt"
import "io"
import "crypto/md5"
import "crypto/sha256"
import "golang.org/x/crypto/blake2s"

func main () {
        h_md5 := md5.New()
        h_sha := sha256.New()
        h_blake2s, _ := blake2s.New256(nil)
        io.WriteString(h_md5, "Welcome to Go Language Secure Coding Practices")
        io.WriteString(h_sha, "Welcome to Go Language Secure Coding Practices")
        io.WriteString(h_blake2s, "Welcome to Go Language Secure Coding Practices")
        fmt.Printf("MD5        : %x\n", h_md5.Sum(nil))
        fmt.Printf("SHA256     : %x\n", h_sha.Sum(nil))
        fmt.Printf("Blake2s-256: %x\n", h_blake2s.Sum(nil))
}
```

出力

```
MD5        : ea9321d8fb0ec6623319e49a634aad92
SHA256     : ba4939528707d791242d1af175e580c584dc0681af8be2a4604a526e864449f6
Blake2s-256: 1d65fa02df8a149c245e5854d980b38855fd2c78f2924ace9b64e8b21b3f2f82
```

**Note**: ソースコードサンプルを実行するためには事前に `$ go get golang.org/x/crypto/blake2s` を実行してください。

一方、暗号化は、データを鍵を使って可変長のデータに変換するものです

```
encrypted_data := F(data, key)
```

ハッシュとは異なり、正しい復号化関数と鍵を使うと暗号化されたデータ（`encrypted_data`）から元データ（`data`）を計算することができます。

```
data := F⁻¹(encrypted_data, key)
```

機密性の高いデータを、通信・保存後に自分を含む誰かがアクセス・処理するような場合には、暗号化する必要があります。
わかりやすい暗号化の使用例としては、HTTPS が挙げられます。
共通鍵暗号化においては AES がデファクトスタンダードです。この
アルゴリズムは、他の多くの対称型暗号と同様に、さまざまな方式で実装することができます。
下のコードサンプルでは、より一般的な CBC/ECB ではなく GCM（ガロアカウンターモード）が使用されていることにお気づきでしょう。
GCM は **認証付き**暗号化であり、暗号化の後に認証タグが暗号文に付与されます。そしてその認証タグを使って復号化の**前に**改ざんされていなことを確認することができます。それが GCM と CBC/ECB の一番の違いです。

これに対して、公開鍵と秘密鍵のペアを使用する公開鍵暗号方式または非対称暗号方式と呼ばれる暗号化方式があります。
非対称暗号方式はほとんどの場合において、対称鍵暗号方式よりも性能が劣ります。
そのため、ほとんどの一般的なユースケースでは、2人の間で共通鍵を非対称暗号を使用して共有しています。その対称鍵を使って対称暗号で暗号化されたメッセージのやり取りを行うのです。

1990年代の技術である AES は当然として、Go の作者は chacha20poly1305 のような認証付きの現代的な対象暗号アルゴリズムの実装とサポートを開始しています。

Go のもう一つの面白いパッケージは、x/crypto/nacl です。
これは Daniel J. Bernstein 博士の NaCl ライブラリを参照したもので、非常に人気のある最新の暗号ライブラリです。
Go の nacl/box と nacl/secretbox は、最も一般的な2つのユースケースに対応した暗号化メッセージ送信のための NaCl の実装を抽象化したものです。

* 公開鍵暗号方式を用いた２者間での認証済み暗号化メッセージの送信（nacl/box）。
* 対称(秘密鍵)暗号を用いた２者間での認証済み暗号化メッセージの送信。

用途に適う場合、AESを直接使用するのではなく、これらの抽象化のいずれかを使用することを強く推奨します。

以下の例は、[AES-256][9](*256 bits/32 bytes*) をベースにした鍵による暗号化・復号化を示しています。
暗号化、復号化、秘密鍵の生成に対する考慮が明確に分離されています。
秘密鍵の生成の際には `secret` メソッドが便利な選択となるでしょう。
ソースコードのサンプルは [こちら][10]から引用したものですが、若干の修正を加えています。

```go
package main

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "io"
    "log"
)

func encrypt(val []byte, secret []byte) ([]byte, error) {
    block, err := aes.NewCipher(secret)
    if err != nil {
        return nil, err
    }

    aead, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    nonce := make([]byte, aead.NonceSize())
    if _, err = io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }

    return aead.Seal(nonce, nonce, val, nil), nil
}

func decrypt(val []byte, secret []byte) ([]byte, error) {
    block, err := aes.NewCipher(secret)
    if err != nil {
        return nil, err
    }

    aead, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    size := aead.NonceSize()
    if len(val) < size {
        return nil, err
    }

    result, err := aead.Open(nil, val[:size], val[size:], nil)
    if err != nil {
        return nil, err
    }

    return result, nil
}

func secret() ([]byte, error) {
    key := make([]byte, 16)

    if _, err := rand.Read(key); err != nil {
        return nil, err
    }

    return key, nil
}

func main() {
    secret, err := secret()
    if err != nil {
        log.Fatalf("unable to create secret key: %v", err)
    }

    message := []byte("Welcome to Go Language Secure Coding Practices")
    log.Printf("Message  : %s\n", message)

    encrypted, err := encrypt(message, secret)
    if err != nil {
        log.Fatalf("unable to encrypt the data: %v", err)
    }
    log.Printf("Encrypted: %x\n", encrypted)

    decrypted, err := decrypt(encrypted, secret)
    if err != nil {
        log.Fatalf("unable to decrypt the data: %v", err)
    }
    log.Printf("Decrypted: %s\n", decrypted)
}
```

```bash
Message  : Welcome to Go Language Secure Coding Practices
Encrypted: b46fcd10657f3c269844da5f824511a0e3da987211bc23e82a9c050a2be287f51bb41dd3546742442498ae9fcad2ce40d88625d1840c11096a55cb4f217382befbeb636e479cfecfcd3a
Decrypted: Welcome to Go Language Secure Coding Practices
```

暗号鍵の管理方法に関するポリシーとプロセスを確立し、不正アクセスからマスターキーを守る必要があることに注意してください。
つまり、暗号鍵はソースコードにハードコードされるべきではないのです。（上の例のように。）

[Go's crypto package][1] には、一般的な暗号用の定数が集められていますが、実装は [crypto/md5][2] のような独自のパッケージ内に分かれています。

最近の暗号アルゴリズムのほとんどは
https://godoc.org/golang.org/x/crypto で実装されているので、開発者は [`crypto/*` package][1] ではなく、前者のパッケージに注目すべきです。

---

[^1]: レインボーテーブル攻撃は、ハッシュアルゴリズム自体の弱点ではありません。
[^2]: [認証とパスワード管理][3]のセクションを読んで、認証情報のための強いソルト付き一方向性ハッシュを考慮してください。

[1]: https://golang.org/pkg/crypto/
[2]: https://golang.org/pkg/crypto/md5/
[3]: ../authentication-password-management/README.md
[4]: https://blake2.net/
[5]: https://godoc.org/golang.org/x/crypto/blake2b
[6]: https://godoc.org/golang.org/x/crypto/blake2s
[7]: https://www.owasp.org/index.php/Credential_stuffing
[8]: https://www.owasp.org/index.php/Brute_force_attack
[9]: https://en.wikipedia.org/wiki/Advanced_Encryption_Standard
[10]: http://www.inanzzz.com/index.php/post/f3pe/data-encryption-and-decryption-with-a-secret-key-in-golang
