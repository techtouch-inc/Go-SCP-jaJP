エラー処理
==============

Go では、組み込みの `error` 型があります。エラー型の値の違いは状態の異常を表します。
通常、Go では `error` の値が `nil` でない場合、エラーが発生したことを表します。
アプリケーションをクラッシュさせることなく、その状態から回復させるために対処する必要があります。

Go ブログから引用した簡単な例を以下に示します。

```go
if err != nil {
    // handle the error
}
```

組み込みのエラーだけでなく、独自のエラー型を指定することもできます。
これは、`errors.New` 関数を使用して実現できます。
例：


```go
{...}
if f < 0 {
    return 0, errors.New("math: square root of negative number")
}
//If an error has occurred print it
if err != nil {
    fmt.Println(err)
}
{...}
```

無効な引数を含む文字列を整形してエラーの原因を調べる必要がある場合は、`fmt` パッケージの `Errorf` 関数を使用します。

```go
{...}
if f < 0 {
    return 0, fmt.Errorf("math: square root of negative number %g", f)
}
{...}
```

エラーログを扱う場合、開発者はエラーレスポンスに機密情報が含まれないようにする必要があります。
同様に、エラーハンドラから情報（デバッグやスタックトレース情報など）が漏れないようにする必要があります。


Go には、`panic`、`recover` および `defer` という補助的なエラー処理関数があります。

アプリケーションの状態が `panic` になった場合、通常の処理は中断され、`defer` 文が実行されて関数は呼び出し元に戻ります。
`recover`は通常、`defer` 文の内部で使用されます。
パニックに陥ったルーチンの制御を回復させ、通常の処理に戻します。
次のスニペットは、Go のドキュメントに基づいて、実行の流れを説明したものです。

```go
func main () {
    start()
    fmt.Println("Returned normally from start().")
}

func start () {
    defer func () {
        if r := recover(); r != nil {
            fmt.Println("Recovered in start()")
        }
    }()
    fmt.Println("Called start()")
    part2(0)
    fmt.Println("Returned normally from part2().")
}

func part2 (i int) {
    if i > 0 {
        fmt.Println("Panicking in part2()!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in part2()")
    fmt.Println("Executing part2()")
    part2(i + 1)
}
```

Output:

```
Called start()
Executing part2()
Panicking in part2()!
Defer in part2()
Recovered in start()
Returned normally from start().
```

出力を調べると、Go がどのように `panic` 状態を処理し、アプリケーションを正常な状態に戻せるかがわかります。
これらの関数により、障害から上手く回復できます。

また、`defer` の使用法には、_Mutex Unlocking_、や他の関数が実行された後にコンテンツを読み込む場合 (例: footer) も含まれることも念頭に置いておきましょう。

`log` パッケージには、`log.Fatal` というものもあります。
致命的なレベルとは、メッセージをログに記録し、その後すぐに `os.Exit(1)` を呼び出すべき状況です。

つまり

* Defer ステートメントを実行しない
* バッファをフラッシュしない
* 一時的なファイルやディレクトリを削除しない

ということです。

これまでに述べたすべてを考慮すると、`log.Fatal` が `Panic` とどう違うのか、`log.Fatal` をなぜ注意深く使わなければならないのかがわかるでしょう。
`log.Fatal` の使用例としては、以下のようなものがあります。

* ロギングをセットアップし、健全な環境とパラメータが設定されているかチェックする際に、失敗した場合。その際は main() を実行する必要はありません。
* 決して発生してはならないエラーで、回復不可能であることが分かっている場合。
* 非対話型プロセスがエラーに遭遇し、完了できない場合、そのことをユーザに通知する方法がない場合。このような場合、さらなる問題が発生する前に、実行を停止するのが最善です。

以下に、初期化に失敗した場合の例を示します。

```go
func initialize(i int) {
    ...
    //This is just to deliberately crash the function.
    if i < 2 {
        fmt.Printf("Var %d - initialized\n", i)
    } else {
        //This was never supposed to happen, so we'll terminate our program.
        log.Fatal("Init failure - Terminating.")
    }
}

func main() {
    i := 1
    for i < 3 {
        initialize(i)
        i++
    }
    fmt.Println("Initialized all variables successfully")
}
```

危険な制御周りにエラーが発生した場合、そのアクセスはデフォルトで拒否されることを保証することが重要です。
