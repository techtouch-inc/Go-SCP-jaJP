出力のエンコーディング
===============

出力のエンコーディングは、[OWASP SCP Quick Reference Guide][1] のセクションには 6 つの箇条書きしかありません。しかし
出力のエンコーディングに関する悪い慣行が Web アプリケーション開発に広く浸透しており、その結果[インジェクション][2]がトップの脆弱性となっています。

Web アプリケーションが複雑になればなるほど、データソースが多くなるのが普通です。たとえば、ユーザー、データベース、サードパーティーのサービスなどです。
そして収集されたデータは、さまざまなコンテキストで Web ブラウザなどのさまざまなメディアに出力されます。インジェクションを受けてしまうのはまさにこのタイミングです。

おそらく私たちがこのセクションで提示するセキュリティイシューについては、すでにお聞きになったことがあるものでしょう。
しかし、どのように起こるのか、どのように回避するのか、本当にご存じでしょうか？

[1]: https://www.owasp.org/images/0/08/OWASP_SCP_Quick_Reference_Guide_v2.pdf
[2]: https://www.owasp.org/index.php/Top_10_2013-A1-Injection
