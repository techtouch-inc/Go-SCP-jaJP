一般的なコーディングプラクティス
========================

ソフトウェア開発時に考慮すべき一般的なガイドラインがいくつかあります。

* 一般的なタスクには、新しいコードを書くのではなく、テスト、承認、保守されているコードを使用すること。

  まったく同じ間違い、バグ、脆弱性に遭遇することが非常に多いでしょう。
  その原因の 1 つは、私たちが「バニラコード」、つまりゼロから書かれ、テストも保守もされていないコードで問題に取り組むことに慣れてしまっていることです。
  可能な限り、多くの人に開発、テスト、利用されているフレームワークのようなマネージドコードを選択しましょう。問題が発生してもより早期に修正されるでしょう。

* OS のタスクを実行するために、タスク固有の組込み API を使用しましょう。特に、アプリケーションが起動するコマンドシェルを利用して OS に直接コマンドを発行しないようにしましょう。

  ほとんどすべてのプログラミング言語では、Go と同じようにコマンドシェルを発行できます。

  ```go
  // Cat (command) a file example
  // set FS permissions to a given (by the user) file
  func main() {
      reader := bufio.NewReader(os.Stdin)
      // Ask the user what file to be read
      file, _ := reader.ReadString('\n')
      if err := exec.Command("cat", "-A", file).Run(); err != nil {
          fmt.Fprintln(os.Stderr, err)
          os.Exit(1)
      }
      fmt.Print("Executed command -> ")
      fmt.Println(file)
      fmt.Println("Command successful.")
  }
  ```

  一見、低レベルのタスクを実行するのに良い方法のように見えますが、深く注意せずに、たとえば、`-c` 引数を伴って OS シェルを呼び出すと、セキュリティ上の問題になります。

  `exec.Command()` を使うのは、引数としてプログラムを受け取るバイナリを実行しない限りは安全です。以下の例では `bash` と `-c` コマンドを使用してプログラムを受け取ってしまっています。

  ```go
  // pass file name as 'file.png; rm -rf / #
  if err := exec.Command("bash", "-c", input).Run(); err != nil {
      fmt.Fprintln(os.Stderr, err)
      os.Exit(1)
  }
  ```

  常にビルトインされた API を使いましょう。


  ```go
  if err := os.Chmod(file, 0644); err != nil {
      log.Fatal(err)
  }
  ```

* ライブラリ、実行ファイル、および設定ファイルのコードの整合性を検証するために、チェックサムまたはハッシュを使用しましょう。

  アプリケーションがライブラリや設定ファイルなどのサードパーティリソースに依存している場合、実行時にそれらがデプロイ時とまったく同じ状態であることを、どうしたら確認できるでしょうか？

  さらに悪いケースを考えてみましょう。アプリケーションがリモートホストからサードパーティのスクリプトをロードする場合、そのファイルがアプリケーションを破壊してしまう変更を加えられていないと、どのように保証するのでしょうか。

  もしかしたら、CDN（Content Delivery Networks）について考えているかもしれませんね。CDN はどこにでもあり、私たちはそれを「必要と」しています。しかし、もし CDN が危険にさらされ、リソースが何らかの形で変更されたらどうでしょう？

  [サブリソースの整合性][1]のセクションをご覧ください。
  私たちは長い間、これなしでどうやって生きてきたのでしょうか？

* 競合状態を防ぐために、複数の同時リクエストを防ぐためのロックを使用するか、同期メカニズムを使用しましょう。

  競合状態とは、共有リソースが複数のクライアントによって同時にアクセスされたときに発生するものです。共有リソースにアクセスする権利は誰にあるのでしょうか？

  これは古くからある問題で、同時並行環境ではよくあることです。
  しかし、その解決策が十分考慮されていないことが多いものです。

  これに対する最良のアプローチは、Go の `sync` パッケージで利用可能な Mutex を使うことです。
  パッケージで利用できます。”Go Tour" から引用した簡単な例です。

  ```go
  package main

  import (
  	"fmt"
  	"sync"
  	"time"
  )

  // SafeCounter is safe to use concurrently.
  type SafeCounter struct {
  	v   map[string]int
  	mux sync.Mutex
  }

  // Inc increments the counter for the given key.
  func (c *SafeCounter) Inc(key string) {
  	c.mux.Lock()
  	// Lock so only one goroutine at a time can access the map c.v.
  	c.v[key]++
  	c.mux.Unlock()
  }

  // Value returns the current value of the counter for the given key.
  func (c *SafeCounter) Value(key string) int {
  	c.mux.Lock()
  	// Lock so only one goroutine at a time can access the map c.v.
  	defer c.mux.Unlock()
  	return c.v[key]
  }

  func main() {
  	c := SafeCounter{v: make(map[string]int)}
  	for i := 0; i < 1000; i++ {
  		go c.Inc("somekey")
  	}

  	time.Sleep(time.Second)
  	fmt.Println(c.Value("somekey"))
  }
  ```

  もう 1 つの問題は、リソースの枯渇で、DoS につながる可能性があります。
  Go にはセマフォのネイティブなサポートはありませんが、バッファドチャネルを使用してセマフォを再現できます。

  セマフォの使い方の例をいくつか挙げます。

      - データベース接続
      - TCP/IP 出力接続
      - スレッド
      - メモリ

  Go でのセマフォの使い方の簡単な例です。

  ```go
  // write to file
  const (
      AvailableMemory         = 10 << 20 // 10 MB
      AverageMemoryPerRequest = 10 << 10 // 10 KB
      MaxOutstanding          = AvailableMemory / AverageMemoryPerRequest
  )

  var sem = make(chan int, MaxOutstanding)

  func Serve(queue chan *Request) {
      for {
          sem <- 1 // Block until there's capacity to process a request.
          req := <-queue
          go handle(req) // Don't wait for handle to finish.
      }
  }

  func handle(r *Request) {
      process(r) // May take a long time & use a lot of memory or CPU
      <-sem      // Done; enable next request to run.
  }
  ```

* 共有変数とリソースを不適切な同時アクセスから保護する。

  この問題にどのようにアプローチするかは、もうお分かりでしょう。mutex またはセマフォを使用することで、付随する問題を解決できます。

* すべての変数とそのほかのデータストアを、宣言時または最初の使用直前に、明示的に初期化します。

* アプリケーションが昇格した特権で実行されなければならない場合、特権を上げるのはできるだけ遅く、落とすのはできるだけ早くしてください。

* 計算ミスを防ぐには、プログラミング言語の基本的な表現技法と、それが数値計算とどのように相互作用するかを理解する必要があります。また
  バイトサイズの不一致、精度、符号付き/符号なし、切り捨て、型間の変換とキャスト、not-a-number　(数字ではない）計算、そして、その言語が基礎となる表現に対して大きすぎたり小さすぎたりする数値をどのように扱うかについて、細心の注意を払う必要があります。

  どんなに優れたプログラミング言語であっても、ハードウェアの制限に対処しなければなりません。私たちが普段忘れがちな制限のひとつには、浮動小数点数の精度の欠落があります。

  ```go
  package main

  import "fmt"

  func main () {
      var n float64 = 0

      for i := 0; i < 10; i++ {
          n += .1
      }

      fmt.Println(n)
  }
  ```

  `0.1` を `10` 回足し合わせているので、`1` になるとお考えかもしれませんが、違います。

  ```bash
  0.9999999999999999
  ```

  巨大な値を扱ったときに何が起きるか見てみましょう。

  ```go
  package main

  import "fmt"
  import "math"

  func main () {
      var n int64 = math.MaxInt64

      fmt.Println(n)
      fmt.Println(n + 1)
  }
  ```

  ```bash
  9223372036854775807
  -9223372036854775808
  ```

  巨大な数を扱うライブラリが必要となるでしょう。[math/big package][4]

  ```go
  package main

  import "fmt"
  import "math"
  import "math/big"

  func main () {
      n1 := new(big.Int).SetInt64(math.MaxInt64)
      n2 := new(big.Int).SetInt64(1)
      sum := new(big.Int)

      fmt.Println(n1)
      fmt.Println(sum.Add(n1, n2))
  }
  ```

    期待通りの答えを得られます。

  ```bash
  9223372036854775807
  9223372036854775808
  ```


* ユーザーから提供されたデータを動的実行関数に渡さないこと。

  詳細は、[入力の検証][2]と[出力のエンコーディング][3]のセクションを読んでください。近道はありません。


* ユーザーが新しいコードを生成したり、既存のコードを変更したりすることを制限しましょう。

  ユーザーがソースコードをアップロードすることを想定しているユースケースがいくつかあります。このような必要がある場合、制限された環境で行う必要があります。そうでなければ、コントロールを失うことになります。

  最初から見ていきましょう。

  アプリケーションのソースコード・ファイルは書き込み可能であってはならず、読み取り専用か、せいぜい実行可能なものにしてください。
  こうすることで、攻撃者がアプリケーションにソースコードを追加して実行させ、
  また、シェルを開いてサーバーを制御することを防げます。

  同じように、アップロードされたファイルのパーミッションも適切に設定する必要があります。通常は画像/写真、スプレッドシート、テキスト文書などで、これらは実行権限を必要としません。

  画像ファイルを扱う場合、サーバー側で前処理をする必要があります。
  安全で標準的なフォーマットに変換し、画像ファイルのメタデータからのスクリプトインジェクションを回避する必要があります。EXIF タグのついた画像は悪意のあるコードが仕込まれている場合があるため、処理する場合は [出力エンコーディング][3] のガイドラインに従うべきです。

  ```go
  // Open out file to be converted
  imageFile, err := os.Open("logo.jpg")
  if err != nil {
      fmt.Println("Error opening file.")
  }

  // decode jpeg into image.Image
  imageDecoded, err := jpeg.Decode(imageFile)

  // Create the new image file
  out, err := os.Create("logo.png")

  // Encode the image to png
  err = png.Encode(out, imageDecoded)
  ```

  特別な理由でユーザーの入力をソースコードとして評価しなければならない
  場合は、サンドボックス環境[^1]でのみ行ってください。

* すべてのセカンダリーアプリケーション、サードパーティーコード、ライブラリをレビューしましょう。ビジネス上の必要性を判断し、機能の安全性を検証しましょう。新たな脆弱性をもたらす可能性があるためです。

  プロジェクトに追加されたサードパーティライブラリを 1 つ 1 つ監査する必要があります。これらは、あなたのアプリケーションの一部であり、同じアクセス権や特権で実行されます。


* 安全なアップデートを実装してください。もし、アプリケーションが自動更新を利用する場合は、コードに暗号署名を使用し、クライアントでその署名を検証するようにしましょう。ホストサーバーからクライアントにコードを転送するためには暗号化されたチャネルを使用しましょう。

---

[^1]: ["Inside the Go Playground"][5], The Go Blog, December 2013


[1]: https://www.w3.org/TR/SRI/
[2]: /input-validation
[3]: /output-encoding
[4]: https://golang.org/pkg/math/big/
[5]: https://blog.golang.org/playground
