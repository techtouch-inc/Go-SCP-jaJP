そのほかのガイドライン
================

認証はあらゆるシステムにおいて重要な部分であるため、常に正しく安全な方法を採用する必要があります。以下は、認証システムをより強固なものにするためのガイドラインです。

* "_Re-authenticate users prior to performing critical operations_"（重要な操作を行う前には、ユーザーを再認証する）
* "_Use Multi-Factor Authentication for highly sensitive or high value
  transactional accounts_"（機密性の高いアカウントや高額な取引を行うアカウントには、多要素認証を使用させる）
* "_Implement monitoring to identify attacks against multiple user accounts, utilizing the same password. This attack pattern is used to bypass standard lockouts, when user IDs can be harvested or guessed_"（同じパスワードを使用した複数のユーザーアカウントに対する攻撃を特定するための監視を実施すること。この攻撃パターンは、ユーザーID を取得または推測できる場合に、標準的なロックアウトを回避するために使用されます）
* "_Change all vendor-supplied default passwords and user IDs or disable the associated accounts_"（ベンダーが提供するデフォルトのパスワードとユーザー ID をすべて変更するか、関連するアカウントを無効化する）
* "_Enforce account disabling after an established number of invalid login attempts (e.g., five attempts is common).  The account must be disabled for a period of time sufficient to discourage brute force guessing of credentials, but not so long as to allow for a denial-of-service attack to be performed_"（無効なログイン試行回数が設定された回数に達したらアカウントの無効化を強制すること（5 回が一般的）。アカウントは、ブルートフォースによる認証情報の推測を阻止するのに十分な期間、無効にしなければならないが、DoS 攻撃が有効となるほど長くするべきではない）
