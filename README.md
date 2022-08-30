This is a Japanese translation of the repo [Go Secure Coding Practice][13] published by [OWASP][4]. 

The content can be downloaded in the following formats. [PDF][12]


# はじめに

Go Language - Web Application Secure Coding Practices は、[Go 言語][1]を使用して Web 開発しようとする人のために書かれたガイドです。

本書は、[Checkmarx Security Research Team][2] との共著であり、[OWASP Secure Coding Practices - Quick Reference Guide v2 (stable)][3] リリースに準拠しています。

本書の主な目的は、「実践的なアプローチ」を通じてよくある間違いを避けられるようになることと同時に、新しいプログラミング言語を学習できるようにすることです。本書は、開発中にどのようなセキュリティ上の問題が発生しうるかを示しながら、「どのように安全に行うか」について、十分なレベルの詳細を提供しています。

## 本書の意義

Stack Overflow が毎年行っている開発者調査によると、Go は 2 年連続で最も愛されているプログラミング言語のトップ 5 に入っています。その人気の急上昇からも、Go で開発されるアプリケーションはセキュリティを考慮して設計されることが非常に重要となっています。

Checkmarx Research Team は、開発者、セキュリティチーム、そして業界全体に対して、一般的なコーディングエラーについての教育を支援し、ソフトウェア開発プロセスで発生しがちな脆弱性についての認識を高めることに貢献しています。

## この本のターゲット読者

Go Secure Coding Practices Guide の主なターゲット読者は、開発者です。
特に、ほかのプログラミング言語での経験をすでにお持ちの方にお勧めです。

また、初めてプログラミングを学ぶ人で [Go tour][8] を修了している方にも、本書は非常に参考になります。

## 何を学べるのか

本書は、[OWASP Secure Coding Practices Guide][3] をトピックごとに解説しています。
Go 言語を使用した例と推奨事項を通して学習することで、一般的な間違いや落とし穴を避けられます。

本書を読めば、安全な Go アプリケーションの開発に自信が持ていることでしょう。

## OWASP セキュア・コーディング・プラクティスについて

 [Secure Coding Practices Quick Reference Guide][3] は、オープン Web アプリケーションプロジェクト（[OWASP][4]）の 1 つです。このプロジェクトは、『技術スタックにとらわれない一般的なソフトウェアセキュリティコーディングプラクティスを、開発ライフサイクルに組み込めるように包括的なチェックリストという形で提供しています。』と宣言されています（[出典][3]）。

[OWASP][4] 自体は、『あらゆる組織が信頼できるアプリケーションの構想、開発、導入、運用、保守できることを目的としたオープンなコミュニティです。
すべての OWASP のツール、ドキュメント、フォーラム、チャプターはすべて無償で提供され、アプリケーションセキュリティの向上に興味のある者が誰でも利用可能です。』（[出典][5]）と宣言されています。

## 貢献するには

本書は、いくつかのオープンソースツールを使用して作成されています。
どのようにゼロから作ったか興味がある方は、
[貢献するには][6]をご覧ください。

## ライセンス

本書はクリエイティブコモンズライセンス 表示 - 継承 4.0 国際ライセンス (CC BY-SA 4.0) に基づいています。
再利用や配布の際には、ライセンスを明示しなければいけません。
[https://creativecommons.org/licenses/by-sa/4.0/][11]


[1]: https://golang.org
[2]: http://chkmrx.co/2sffXFr
[3]: https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/migrated_content
[4]: https://www.owasp.org
[5]: https://www.owasp.org/index.php/About_OWASP
[6]: src/ja/howto-contribute.md
[7]: https://www.twitter.com/checkmarx
[8]: https://www.gitbook.com/
[9]: https://checkmarx.gitbooks.io/go-scp/
[10]: https://www.gitbook.com/book/checkmarx/go-scp/
[11]: https://creativecommons.org/licenses/by-sa/4.0/
[12]: dist/go-webapp-scp.pdf
[13]: https://github.com/OWASP/Go-SCP/
