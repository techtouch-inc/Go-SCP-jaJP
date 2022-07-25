認証データの伝達
=================================

本章では、「コミュニケーション」を広い意味で使用し、以下を包含しています。
- ユーザーエクスペリエンス（UX）
- クライアント／サーバ間のコミュニケーション。

「入力されたパスワードは画面上で隠されるべき」というだけでなく、「remember me 機能を無効にすべき」ということも真実です。

入力フィールドを `type="password"` とし、さらに`autocomplete` 属性を`off`[^1]に設定することで、その両方を実現できます。

```html
<input type="password" name="passwd" autocomplete="off" />
```

> 訳者注：　autocomplete 機能を off にしてもブラウザ固有のIDパスワード管理機能がオフになることはありません。ただし例えば google chrome のIDパスワード保存機能であれば、アカウントに紐づけられるものなので、共用PCで入力履歴が残るというリスクが避けられたりと違いがあります。

認証情報は、送信される際には HTTPS のような暗号化された接続を介する必要があります。
認証情報のリセットなどに使われる電子メールによる一時的なパスワードの送信は、その例外の1つでしょう。
> 訳者注：メールは送信側だけでは経路の安全性をコントロールできません。受信者までの経路の途中で平文で通信されている可能性があります。

リクエストされた URL は通常、HTTP サーバによってクエリ文字列を含んだアクセスログに記録されることを覚えておいてください。
認証情報を HTTP サーバのアクセスログに漏らさないために、サーバへのデータ送信には
HTTP の `POST` メソッドを使用しましょう。
>  訳者注：　https://www.ietf.org/rfc/rfc2616.txt の 15.1.3 にも書かれている。

```text
xxx.xxx.xxx.xxx - - [27/Feb/2017:01:55:09 +0000] "GET /?username=user&password=70pS3cure/oassw0rd HTTP/1.1" 200 235 "-" "Mozilla/5.0 (X11; Fedora; Linux x86_64; rv:51.0) Gecko/20100101 Firefox/51.0"
```

認証のためによく設計された HTML フォームは次のようなものです。


```html
<form method="post" action="https://somedomain.com/user/signin" autocomplete="off">
    <input type="hidden" name="csrf" value="CSRF-TOKEN" />

    <label>Username <input type="text" name="username" /></label>
    <label>Password <input type="password" name="password" /></label>

    <input type="submit" value="Submit" />
</form>
```

認証エラーを処理する際、アプリケーションは認証情報のどの部分が不正だったか公開してはいけません。たとえば、"Invalid username" や "Invalid password" ではなく
”Invalid username and/or password" とすべきです。

```html
<form method="post" action="https://somedomain.com/user/signin" autocomplete="off">
    <input type="hidden" name="csrf" value="CSRF-TOKEN" />

    <div class="error">
        <p>Invalid username and/or password</p>
    </div>

    <label>Username <input type="text" name="username" /></label>
    <label>Password <input type="password" name="password" /></label>

    <input type="submit" value="Submit" />
</form>
```

あいまいなメッセージを使用することで、以下の情報が開示されません。

* 誰が登録しているか： "Invalid password" というメッセージから、ユーザー名がすでに存在することを意味します。
* システムがどのように動作するか：”Invalid password" というメッセージは、アプリケーションがどのように動作しているかを明らかにしています。まず、`uername`をデータベースに問い合わせ、次に、`password`をインメモリで比較しています。

認証情報のバリデーション（および保存）の実行例は、[バリデーションとストレージのセクション][5]でご覧いただけます。

ユーザーがログインに成功したら、最後にログインに成功/失敗した日時を教えてあげましょう。ユーザーが不審なアクティビティを発見して報告できるようにします。
ログに関する詳細についてはこのドキュメントの [`Error Handling and Logging`][4] のセクションをご覧ください。

また、タイミングアタックを防ぐためにパスワードのチェックの際には実行時間が不変の比較関数を使用することをお勧めします。 タイミングアタックとは、異なる入力による複数のリクエストに対する処理の時間差を解析する手法です。

`record == password` という文字列を標準的な方法で比較した場合、一致しない最初の文字で、false を返すでしょう。この場合、 送信されたパスワードが正解に近いほど応答時間が長くなります。

これを利用すれば、攻撃者はパスワードを推測できます。
もしもレコードが存在しない場合でも、空文字列とユーザーの入力を`subtle.ConstantTimeCompare`を使って比較することに気を付けましょう。

---

[^1]: [How to Turn Off Form Autocompletion][1], Mozilla Developer Network
[^2]: [Log Files][2], Apache Documentation
[^3]: [log_format][3], nginx log_module "log_format" directive

[1]: https://developer.mozilla.org/en-US/docs/Web/Security/Securing_your_site/Turning_off_form_autocompletion
[2]: https://httpd.apache.org/docs/1.3/logs.html#accesslog
[3]: http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format
[4]: ../error-handling-logging/logging.md
[5]: ./validation-and-storage.md#storing-password-securely-the-practice
