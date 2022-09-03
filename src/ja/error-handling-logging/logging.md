ロギング
=======

ログは常にアプリケーションによって処理されるべきで、サーバー構成に依存するべきではありません。

すべてのログは、信頼できるシステム上のマスタルーチンによって実現されるべきです。
開発者は、機密データがログに含まれないようにする必要があります。(パスワード、セッション情報、システムの詳細など）同様に、デバッグやスタックトレースの情報も含まれてはいけません。

また重要なイベントデータの記録に重点を置いて、成功したセキュリティイベントと失敗したセキュリティイベントの両方をカバーする必要があります。

重要なイベントデータとは、一般的に以下のものを指します。

* 入力値のバリデーションの失敗
* 認証の試行、特にその失敗
* アクセス制御の失敗
* 状態データへの予期せぬ変更といった、明らかな改ざんの試行
* 無効または期限切れのセッション・トークンでの接続試行
* システム例外
* セキュリティ構成の設定の変更を含む管理機能の実行
* バックエンドの TLS 接続の失敗と暗号化モジュールの失敗

これを説明する簡単なログの例を示します。

```go
func main() {
    var buf bytes.Buffer
    var RoleLevel int

    logger := log.New(&buf, "logger: ", log.Lshortfile)

    fmt.Println("Please enter your user level.")
    fmt.Scanf("%d", &RoleLevel) //<--- example

    switch RoleLevel {
    case 1:
        // Log successful login
        logger.Printf("Login successful.")
        fmt.Print(&buf)
    case 2:
        // Log unsuccessful Login
        logger.Printf("Login unsuccessful - Insufficient access level.")
        fmt.Print(&buf)
     default:
        // Unspecified error
        logger.Print("Login error.")
        fmt.Print(&buf)
    }
}
```

一般的なエラーメッセージや、カスタムエラーページを実装することで、エラー発生時に情報が漏れないようにするのも良い方法です。

---

[Go's log package][0] は、ドキュメントによると**シンプル**なロギングを実装しているとのことです。レベルごとのロギング (例： `debug`、`info`、`warn`、`error`、`fatal`、`panic`) や、フォーマッタのサポート (例： logstash) といった一般的で重要な機能が欠けています。
これら 2 つの機能はログを利用しやすくするための重要な要素です。
(例： Security Information や Event Management system との統合）。

すべてではないにせよ、ほとんどのサードパーティのロギングパッケージは、これらの機能およびそのほかの機能を提供しています。
以下に挙げるのは、最も人気のあるサードパーティのロギングパッケージです。

* [Logrus][1] - https://github.com/Sirupsen/logrus
* [glog][2] - https://github.com/golang/glog
* [loggo][3] - https://github.com/juju/loggo

ここで、[Go's log package][0] に関する重要な注意事項があります。Fatal と Panic 関数はロギング後に異なる動作をします。Panic 関数は `panic` を呼び出しますが、Fatal 関数は `os.Exit(1)` を呼び出します。後者は defer ステートメントを無視してプログラムを終了させるので、バッファの Flash、および一時データの削除ができないかもしれません。


---

ログアクセスの観点としては、許可された個人のみがログにアクセスできるべきです。
また、ログを解析できるようなしくみを用意し、信頼できないデータがログ閲覧ソフトやインタフェースプログラムに実行されないことを保証する必要があります。


割り当てられたメモリのクリーンアップについては、Go には専用のガベージコレクタが組み込まれています。

ログの有効性と完全性を保証する最終段階として、暗号ハッシュ関数を使用して、ログが改ざんされていないことを確認するとより良いでしょう。


```go
{...}
// Get our known Log checksum from checksum file.
logChecksum, err := ioutil.ReadFile("log/checksum")
str := string(logChecksum) // convert content to a 'string'

// Compute our current log's SHA256 hash
b, err := ComputeSHA256("log/log")
if err != nil {
  fmt.Printf("Err: %v", err)
} else {
  hash := hex.EncodeToString(b)
  // Compare our calculated hash with our stored hash
  if str == hash {
    // Ok the checksums match.
    fmt.Println("Log integrity OK.")
  } else {
    // The file integrity has been compromised...
    fmt.Println("File Tampering detected.")
  }
}
{...}
```

注意： `ComputeSHA256()` はファイルの SHA256 を計算する関数です。ログファイルのハッシュは安全な場所に保存され、ログを更新する前に完全性を検証するために現在のログハッシュと比較されなければならないことに注意してください。
[リポジトリにワーキングデモがあります][4]。

[0]: https://golang.org/pkg/log/
[1]: https://github.com/Sirupsen/logrus
[2]: https://github.com/golang/glog
[3]: https://github.com/juju/loggo
[4]: ./assets/log-integrity.go
