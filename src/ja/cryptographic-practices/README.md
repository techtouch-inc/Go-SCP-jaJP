実践的な暗号
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
（それでも衝突は起こるかもしれませんが）。優れたハッシュアルゴリズムでは、ハッシュ値を元の値に戻すことはできません[^1]。ハッシュアルゴリズムとしては、MD5が最も有名ですが、セキュリティ的には[BLAKE2][4]が最も強く、柔軟性があると考えられています。

Goの補助的な暗号ライブラリには、[BLAKE2b][5]（単にBLAKE2のこと）と[BLAKE2s][6]の実装があます。前者は64ビット環境用に最適化されており、後者は8ビットから32ビットまでの環境に対応しています。BLAKE2が使えない場合が利用できない場合は、SHA-256が良い選択肢となります。

暗号ハッシュアルゴリズムにおいて、遅いことは望ましいということに注意してください。
コンピュータは年々高速化しているため、同時に攻撃者はより多くのパスワードを試すことができるようになっています。（[クレデンシャルスタッフィング][7]および[ブルートフォースアタック][8]）。それに対抗してハッシュ関数は、少なくとも10,000回の反復的な処理を内包する、本質的に遅い関数であるべきです。

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
わかりやすい暗号化の使用例としては、HTTPSが挙げられます。
共通鍵暗号化においてはAESがデファクトスタンダードです。この
アルゴリズムは、他の多くの対称型暗号と同様に、さまざまな方式で実装することができます。
下のコードサンプルでは、より一般的なCBC/ECB（暗号のコードサンプルにおいては正しいはず。）ではなくGCM（ガロアカウンターモード）が使用されていることにお気づきでしょう。
GCM は **認証付き**暗号化であり、暗号化の後に認証タグが暗号文に付与されます。そしてその認証タグを使って復号化の**前に**改ざんされていなことを確認することができます。それが GCM と CBC/ECB の一番の違いです。

これに対して、公開鍵と秘密鍵のペアを使用する公開鍵暗号方式または非対称暗号方式と呼ばれる暗号化方式があります。
非対称暗号方式はほとんどの場合において、対称鍵暗号方式よりも性能が劣ります。
そのため、ほとんどの一般的なユースケースでは、2人の間で共通鍵を非対称暗号を使用して共有しています。その対称鍵を使って対称暗号で暗号化されたメッセージのやり取りを行うのです。

1990年代の技術であるAESは置いておいて、Goの作者はchacha20poly1305のような認証付きの現代的な対象暗号アルゴリズムの実装とサポートを開始しています。

Another interesting package in Go is x/crypto/nacl. This is a reference to
Dr. Daniel J. Bernstein's NaCl library, which is a very popular modern
cryptography library.
The nacl/box and nacl/secretbox in Go are implementations of NaCl's
abstractions for sending encrypted messages for the two most common use-cases:

* Sending authenticated, encrypted messages between two parties using public
  key cryptography (nacl/box)
* Sending authenticated, encrypted messages between two parties using symmetric
  (a.k.a secret-key) cryptography

It is very advisable to use one of these abstractions instead of direct use of
AES, if they fit your use-case.

The example below illustrates encryption and decryption using an [AES-256][9]
(*256 bits/32 bytes*) based key, with a clear separation of concerns such as
encryption, decryption and secret generation. The `secret` method is a
convenient option which helps generate a secret. The source code sample was
taken from [here][10] but has been slightly modified.

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

Please note, you should "_establish and utilize a policy and process for how
cryptographic keys will be managed_", protecting "_master secrets from
unauthorized access_". That being said, your cryptographic keys shouldn't be
hardcoded in the source code (as it is in this example).

[Go's crypto package][1] collects common cryptographic constants, but
implementations have their own packages, like the [crypto/md5][2] one.

Most modern cryptographic algorithms have been implemented under
https://godoc.org/golang.org/x/crypto, so developers should focus on those
instead of the implementations in the [`crypto/*` package][1].

---

[^1]: Rainbow table attacks are not a weakness on the hashing algorithms.
[^2]: Consider reading the [Authentication and Password Management][3] section about "_strong one-way salted hashes_" for credentials.

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
