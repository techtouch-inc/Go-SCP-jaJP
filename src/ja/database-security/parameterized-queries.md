パラメータライズドクエリ
=====================

プリペアドステートメント（パラメータライズドクエリ）は、SQL インジェクションから防御するための最も安全な方法です。

しかし、一部の報告では、プリペアド・ステートメントが Web アプリケーションのパフォーマンスを低下させる可能性があることが報告されています。したがって、何らかの理由でこの手のデータベースクエリを使用できない場合は、[入力値のバリデーション][1]と[出力のエンコーディング][2]を読むことを強くお勧めします。

Go はほかの言語でのプリペアドステートメントとは異なる動作をします。
接続時にステートメントを準備するのではありません。DB 上でステートメントを用意するのです。

## フロー

1. 開発者はプール内のあるコネクションでステートメント（`Stmt`)を準備する。
2. `Stmt` オブジェクトは、どのコネクションを使用したかを記憶する。
3. アプリケーションが `Stmt` を実行するとき、記憶したコネクションを使用しようとする。
   利用できない場合は、プール内の別のコネクションを探そうとします。

このようなフローは、データベースの高負荷な同時使用を引き起こし、多くのプリペアドステートメントを作成することになりえます。したがって、このことを心にとどめておくことが重要です。

以下は、パラメータ化されたクエリを使用したプリペアドステートメントの例です。

```go
customerName := r.URL.Query().Get("name")
db.Exec("UPDATE creditcards SET name=? WHERE customerId=?", customerName, 233, 90)
```

プリペアドステートメントが、望ましくない場合もあります。いくつかの理由が考えられます。

* データベースがプリペアドステートメントをサポートしていない場合。
MySQL ドライバを利用している場合、wire protocol をサポートしている　MemSQL や Sphinx は MySQL に接続できます。しかし、それらはプリペアドステートメントを含む "バイナリ "プロトコルをサポートしていないため、紛らわしい形で失敗する可能性があります。

* ステートメントが効果が出るほどには再利用されず、かつセキュリティの問題はアプリケーションスタックの別のレイヤで処理される場合 (参照： [Input Validation][1] and [Output Encoding][2])。上記のようなパフォーマンス低下は望ましくありません。

[1]: ../input-validation/README.md
[2]: ../output-encoding/README.md
