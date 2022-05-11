# はじめに

Go Language - Web Application Secure Coding Practices は、[Go 言語][1]を使用してWeb開発しようとする人のために書かれたガイドです。

本書は、[Checkmarx Security Research Team][2] との共著であり、[OWASP Secure Coding Practices - Quick Reference Guide v2 (stable)][3] リリースに準拠しています。

本書の主な目的は、「実践的なアプローチ」を通じてよくある間違いを避けられるようになることと同時に、新しいプログラミング言語を学習できるようにすることです。本書は、開発中にどのようなセキュリティ問題が発生しうるかを示しながら、「どのように安全に行うか」について、十分なレベルの詳細を提供しています。

## 本書の意義

Stack Overflowが毎年行っている開発者調査によると、Goは2年連続で最も愛されているプログラミング言語のトップ5に入っています。その人気の急上昇からも、Goで開発されるアプリケーションはセキュリティを考慮して設計されることが非常に重要となっています。

Checkmarx Research Team は、開発者、セキュリティチーム、そして業界全体に対して、一般的なコーディングエラーについての教育を支援し、ソフトウェア開発プロセスで発生しがちな脆弱性についての認識を高めることに貢献しています。

## この本のターゲット読者

Go Secure Coding Practices Guide の主なターゲット読者は、開発者です。
特に、他のプログラミング言語での経験をお持ちの方におすすめです。

また、初めてプログラミングを学ぶ人で[Go tour][8]を修了している方にも、本書は非常に参考になります。

## 何を学べるのか

本書は、[OWASP Secure Coding Practices Guide][3]をトピックごとに解説しています。
Go言語を使用した例と推奨事項を通して学習することで、一般的な間違いや落とし穴を避けられるようになります。

本書を読めば、安全なGoアプリケーションの開発に自信が持てることでしょう。

## OWASP セキュア・コーディング・プラクティスについて

 [Secure Coding Practices Quick Reference Guide][3]は、オープンWEBアプリケーションプロジェクト（[OWASP][4]）の１つです。このプロジェクトは、『技術スタックにとらわれない一般的なソフトウェアセキュリティコーディングプラクティスを、開発ライフサイクルに組み込めるように包括的なチェックリストという形で提供しています。』と宣言されています（[出典][3]）。

[OWASP][4]自体は、『あらゆる組織が信頼できるアプリケーションの構想、開発、導入、運用、保守できることを目的としたオープンなコミュニティである。
全てのOWASPのツール、ドキュメント、フォーラム、チャプターはすべて無償で提供され、アプリケーションセキュリティの向上に興味のある者が誰でも利用可能である。』（[出典][5]）と宣言されています。

## 貢献するには

本書は、いくつかのオープンソースツールを使用して作成されています。
どのようにゼロから作ったか興味がある方は、
[貢献するには][6]をご覧ください。

[1]: https://golang.org
[2]: http://chkmrx.co/2sffXFr
[3]: https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/migrated_content
[4]: https://www.owasp.org
[5]: https://www.owasp.org/index.php/About_OWASP
[6]: /howto-contribute.md
[7]: https://www.twitter.com/checkmarx
[8]: https://tour.golang.org/list
