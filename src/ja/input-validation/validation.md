バリデーション
==========

バリデーションとは、ユーザーの入力を一連の条件と照らし合わせてチェックすることです。それによってユーザーが本当に期待通りのデータを入力していることを保証します。

**重要:** バリデーションが失敗したら、その入力は拒否されなければならない。

これは、セキュリティの観点だけでなく、データの一貫性と完全性という観点からも重要です。データは通常、さまざまなシステムやアプリケーションで使用されるためです。


本章では、開発者が Web アプリケーションの開発の際に注意するべきことをリストアップします。

## ユーザーインタラクティビティ

ユーザーからの入力を許可するアプリケーションのコンポーネントはすべて、潜在的なセキュリティリスクとなります。
アプリケーションを侵害しようとする脅威だけではなく、ヒューマンエラーによる誤入力からも問題は起こります。（統計的に。不正なデータが発生する原因の大半は人為的なものであることがほとんどです）
Go では、このような問題からアプリケーションを保護する方法がいくつかあります。

Go にはネイティブのライブラリにはこのようなエラーを確実に防ぐためのメソッドが含まれています。
文字列を扱う場合、以下のようなパッケージを利用できます。

例

* `strconv` パッケージは、文字列からほかのデータ型への変換を扱います。
    * [`Atoi`](https://golang.org/pkg/strconv/#Atoi)
    * [`ParseBool`](https://golang.org/pkg/strconv/#ParseBool)
    * [`ParseFloat`](https://golang.org/pkg/strconv/#ParseFloat)
    * [`ParseInt`](https://golang.org/pkg/strconv/#ParseInt)
* `strings` パッケージには、文字列とそのプロパティを処理するためのすべての関数が含まれています。
    * [`Trim`](https://golang.org/pkg/strings/#Trim)
    * [`ToLower`](https://golang.org/pkg/strings/#ToLower)
    * [`ToTitle`](https://golang.org/pkg/strings/#ToTitle)
* [`regexp`][4] パッケージは正規表現をサポートし、独自のフォーマットを扱えます[^1]。
* [`utf8`][9] パッケージは、UTF-8 テキストをサポートするための関数と定数を実装しています。ルーン文字とバイト列を変換する関数が含まれています。

  UTF-8 でエンコードされたルーンのバリデーション
    * [`Valid`](https://golang.org/pkg/unicode/utf8/#Valid)
    * [`ValidRune`](https://golang.org/pkg/unicode/utf8/#ValidRune)
    * [`ValidString`](https://golang.org/pkg/unicode/utf8/#ValidString)

  UTF-8 のエンコード
    * [`EncodeRune`](https://golang.org/pkg/unicode/utf8/#EncodeRune)

  UTF-8 のデコード
    * [`DecodeLastRune`](https://golang.org/pkg/unicode/utf8/#DecodeLastRune)
    * [`DecodeLastRuneInString`](https://golang.org/pkg/unicode/utf8/#DecodeLastRuneInString)
    * [`DecodeRune`](https://golang.org/pkg/unicode/utf8/#DecodeLastRune)
    * [`DecodeRuneInString`](https://golang.org/pkg/unicode/utf8/#DecodeRuneInString)

**Note**: Go では `Forms` は、`String` 値の `Map` として扱われます。

そのほか、データの妥当性を保証するためのテクニックとして、以下のようなものがあります。

* ホワイトリスティング - 可能な限り、ホワイトリスティングによるバリデーションを採用します。[バリデーション - すべてのタグを取り除く][1]を参照してください。
* バウンダリーチェック - データや数値の長さをチェックします。
* 数値バリデーション - 入力が数値である場合にチェックします。
* ヌルバイトのチェック - `(%00)`
* 文字エスケープ - シングルクオテーションのような特殊文字をチェックします。
* 改行文字のチェック - `%0d, %0a, \r, n`
* パス変換文字列のチェック - `../` or `\\...`
* 拡張 UTF-8 のチェック - 特殊文字の代替表現をチェックします。


**Note**: HTTP リクエストヘッダとレスポンスヘッダが、ASCII 文字だけで構成されていることを確認してください。

Go のセキュリティを扱うサードパーティパッケージが存在します。

* [Gorilla][6] - Web アプリケーションのセキュリティのために最もよく使われるパッケージの 1 つです。`websockets`, `cookie sessions`, `RPC` などをサポートしています。
* [Form][7] - `url.Values` と値をデコード、エンコードします。`Array`と`Map`をサポートします。
* [Validator][8] - `Struct` と `Field` をバリデーションします。対象は `Cross Field`、`Cross Struct`、`Map`、`Slice`、`Array` を含みます。

## ファイル操作

ファイルを利用する際（ファイルの読み取りまたは書き込み）、それはほとんどの場合がユーザーデータの処理であるため常にバリデーションが行われるべきです。

そのほか、ファイルチェックとしては、ファイル名による存在確認があります。

さらなるファイルに関するテクニックは、[ファイル管理][2]に、エラー処理については、[エラー処理][3]に記載されています。

## データソース

信頼できるソースから信頼度の低いソースにデータが渡される場合は、常に完全性のチェックが必要です。改ざんされておらず意図したデータであることを保証するためです。
そのほか、データソースのチェックには以下のようなものがあります。

* システム間整合性チェック
* ハッシュ合計
* 参照整合性（Referential integrity)

**Note：** 最近のリレーショナル・データベースに備わる内部メカニズムによって、主キー・フィールドの値制約がなされていない場合、バリデーションの必要があります。

* 一意性チェック
* テーブル・ルックアップ・チェック


## ポストバリデーション

データバリデーションのベストプラクティスによると、入力値バリデーショはあくまでもデータ検証のガイドラインの初歩です。ポストバリデーションも実施する必要があります。ポストバリデーションはコンテキストによって異なり、以下の 3 つのカテゴリに分類されます。


* **エンフォースメント**　アプリケーションとデータの安全性を高めるために、いくつかのタイプの方式が存在します。

  * 提出されたデータが規格に適合しておらず、条件を満たすように修正しなければいけないことをユーザーに通知する方式。
  * ユーザーから送信されたデータを、ユーザーに通知することなくサーバー側で修正する方式。インタラクティブなシステムに最適です。

  **Note：** 後者は主に見た目の変更に使用されます（たとえば機密性の高いユーザーデータの切り捨ていなどはデータ消失につながる可能性があります）。

* **アドバイザリー**　変更を強制はしませんがシステムには入力に問題があったことが通知、記録されます。非インタラクティブなシステムに適しています。

* **ベリフィケーション** アドバイザリーアクションの中でも特殊なケースです。システムはユーザーに検証と修正を依頼します。ユーザーがその提案を受け入れるか判断します。

  簡単に例を説明すると、請求アドレスの入力フォームで、ユーザーの入力に対してシステムが関連する住所を複数提案し、ユーザーがそれを受け入れると住所が補完入力され、受け入れなければ入力がそのまま残るといったシチュエーションです。

---

[^1]: 独自の正規表現を書く前に、[OWASP Validation Regex Repository][5]を見てください。

[1]: sanitization.md
[2]: ../file-management/README.md
[3]: ../error-handling-logging/README.md
[4]: https://golang.org/pkg/regexp/
[5]: https://www.owasp.org/index.php/OWASP_Validation_Regex_Repository
[6]: https://github.com/gorilla/
[7]: https://github.com/go-playground/form
[8]: https://github.com/go-playground/validator
[9]: https://golang.org/pkg/unicode/utf8/
