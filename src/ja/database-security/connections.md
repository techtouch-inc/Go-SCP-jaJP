データベース接続
====================

## コンセプト

`sql.Open` はデータベース接続を返すのではなく、 `*DB`: データベース接続プールを返します。
データベース操作 (例: クエリ) が実行されようとすると、コネクションプールから利用可能なコネクションが取得されますが、操作が完了したらすぐにコネクションプールに返しましょう。

データベース接続は、クエリなどのデータベース操作の実行のために初めて要求とされたときにのみ開かれることに留意してください。

`sql.Open` は、データベース接続のテストさえ行いません。
クレデンシャルが間違っていると、最初のデータベース操作の実行時にエラーが発生します。

経験則から言うと、`database/sql` インターフェースのコンテキストバリアント(例: `QueryContext()`) は、常に適切な[Context][3]で利用されます。

Go の公式ドキュメントより "Package context は Context 型を定義します。
API境界やプロセス間を横断して、デッドライン、キャンセルシグナル、その他のリクエストに対応した値を保持します。

データベースレベルでは、コンテキストがキャンセルされると、コミットされない限りトランザクションはロールバックされます。
(QueryContext の) Rows がクローズされ、すべてのリソースが返却されます。

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"

    _ "github.com/go-sql-driver/mysql"
)

type program struct {
    base context.Context
    cancel func()
    db *sql.DB
}

func main() {
    db, err := sql.Open("mysql", "user:@/cxdb")
    if err != nil {
        log.Fatal(err)
    }
    p := &program{db: db}
    p.base, p.cancel = context.WithCancel(context.Background())

    // Wait for program termination request, cancel base context on request.
    go func() {
        osSignal := // ...
        select {
        case <-p.base.Done():
        case <-osSignal:
            p.cancel()
        }
        // Optionally wait for N milliseconds before calling os.Exit.
    }()

    err =  p.doOperation()
    if err != nil {
        log.Fatal(err)
    }
}

func (p *program) doOperation() error {
    ctx, cancel := context.WithTimeout(p.base, 10 * time.Second)
    defer cancel()

    var version string
    err := p.db.QueryRowContext(ctx, "SELECT VERSION();").Scan(&version)
    if err != nil {
        return fmt.Errorf("unable to read version %v", err)
    }
    fmt.Println("Connected to:", version)
}
```

## 接続文字列の保護

接続文字列を安全に保つために、認証の詳細については、一般に公開されていない独立した設定ファイルに保存することをお勧めします。

設定ファイルを `/home/public_html/` に置くのではなく、
`/home/private/configDB.xml`: 保護された領域を検討してください。

Instead of placing your configuration file at `/home/public_html/`, consider `/home/private/configDB.xml`: a protected area.

```xml
<connectionDB>
  <serverDB>localhost</serverDB>
  <userDB>f00</userDB>
  <passDB>f00?bar#ItsP0ssible</passDB>
</connectionDB>
```

そして、Goファイル上で configDB.xml ファイルを呼び出すことができます。

```go
configFile, _ := os.Open("../private/configDB.xml")
```

ファイルを読み込んだら、データベース接続を行います。

```go
db, _ := sql.Open(serverDB, userDB, passDB)
```

もちろん、攻撃者がルートアクセス権を持っていれば、そのファイルを見ることができます。
そこで、最も注意しなければならないのは、ファイルを暗号化することです。

## データベースのクレデンシャル

信頼区分とレベルごとに異なるクレデンシャルを使用しましょう。
例えば

* ユーザー
* 読み取り専用ユーザー
* ゲスト
* 管理者

こうすると、読み取り専用のユーザーが接続しても、そのユーザーは実際には読み取りしかできないので
データベースを壊すことはできません。

[1]: https://golang.org/pkg/database/sql/#DB.Close
[2]: ../error-handling-logging/README.md
[3]: https://golang.org/pkg/context/
