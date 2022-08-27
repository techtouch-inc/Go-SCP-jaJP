データベースのセキュリティ
=================

OWASP SCP のこのセクションは、データベースのセキュリティに関するすべての問題と、データベースを使用する Web サイトにおいて開発者と Database Administrator (DBA) がなすべきことを説明します。

Go にはデータベースドライバがありません。その代わり、コアインタフェースドライバが
[database/sql][1] パッケージにあります。つまりデータベース接続の際には、SQL ドライバ (例： [MariaDB][2], [sqlite3][3]) を登録する必要があります。

## ベストプラクティス

Go でデータベースを実装する前に、次に説明する設定に気を付けましょう。

* データベースサーバーの安全なインストール[^1]
    * `root` アカウントのパスワードを変更・設定。
    * ローカルホスト以外からアクセス可能な `root` アカウントを削除。
    * 匿名ユーザーアカウントを削除。
    * 既存のテスト用データベースの削除。
* 不要なストアドプロシージャ、ユーティリティパッケージ、不要なサービス、ベンダーのコンテンツ（例： サンプルスキーマ）をすべて削除。
* データベースが Go で動作するために必要な最小限の機能とオプションを準備。
* Web アプリケーションがデータベースへ接続するために必要のないデフォルトのアカウントはすべて無効化。


また、入力を検証し、エンコードしてからデータベースに出力することが重要です。
[Input Validation][4] と [Output Encoding][5] を必ず参照してください。

これはデータベースを使う場合、基本的にどのプログラミング言語でも同様です。

---


[^1]: MySQL/MariaDB には安全なインストールのためのプログラムがあります。: `mysql_secure_installation`<sup>[1][6], [2][7]</sup>

[1]: https://golang.org/pkg/database/sql/
[2]: https://github.com/go-sql-driver/mysql
[3]: https://github.com/mattn/go-sqlite3
[4]: ../input-validation/README.md
[5]: ../output-encoding/README.md
[6]: https://dev.mysql.com/doc/refman/5.7/en/mysql-secure-installation.html
[7]: https://mariadb.com/kb/en/mariadb/mysql_secure_installation/
