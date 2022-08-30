SQLインジェクション
=============

出力のエンコーディングが適切でないために発生するもう 1 つの一般的なインジェクションは、SQL インジェクションです。これは主に、古い悪習である文字列の連結が原因です。

DBMS にとって特別な意味を持つ文字を含むことができる変数を SQL クエリに単純に追加するような処理は、SQL インジェクションに対して脆弱だと言えます。

以下のようなクエリがあったとします。

```go
ctx := context.Background()
customerId := r.URL.Query().Get("id")
query := "SELECT number, expireDate, cvv FROM creditcards WHERE customerId = " + customerId

row, _ := db.QueryContext(ctx, query)
```

まさに脆弱性を利用され、侵入されてしまうようなコードです。

たとえば、有効な `customerId` 値が提供された場合、顧客のクレジットカード情報のみをリストアップすることになります。
しかし、`customerId` に `1 OR 1=1` を入れたらどうなるでしょう？

クエリは次のようになります。

```SQL
SELECT number, expireDate, cvv FROM creditcards WHERE customerId = 1 OR 1=1
```

すると、すべてのテーブルレコードがダンプされます（`1=1` はどのレコードに対しても真になります）。

データベースを安全に保つ方法はたった 1 つ、[プリペアドステートメント][1]です。

```go
ctx := context.Background()
customerId := r.URL.Query().Get("id")
query := "SELECT number, expireDate, cvv FROM creditcards WHERE customerId = ?"

stmt, _ := db.QueryContext(ctx, query, customerId)
```

プレースホルダー `?` に注目してください。これであなたのクエリは

 * 読みやすく
 * より短く
 * 安全

になりました。プリペアドステートメントにおけるプレースホルダーの構文は、データベースによって異なります。

たとえば、MySQL、PostgreSQL、Oracle は以下になります。

| MySQL | PostgreSQL | Oracle |
| :---: | :--------: | :----: |
| WHERE col = ? | WHERE col = $1 | WHERE col = :col |
| VALUES(?, ?, ?) | VALUES($1, $2, $3) | VALUES(:val1, :val2, :val3) |

「データベースのセキュリティ」セクションをチェックして、このトピックに関してより詳細な情報を入手してください。

[1]: https://golang.org/pkg/database/sql/#DB.Prepare
