---
title: "NestJSとは"
---
NestJSとは、Node.jsサーバーサイドアプリケーションを構築するためのフレームワークです。

公式ドキュメントにはその哲学として、以下の説明がなされています。

>  However, while plenty of superb libraries, helpers, and tools exist for Node (and server-side JavaScript), none of them effectively solve the main problem of - Architecture.
> Nest provides an out-of-the-box application architecture which allows developers and teams to create highly testable, scalable, loosely coupled, and easily maintainable applications. 

引用: https://docs.nestjs.com/

このように、Node.jsに**アーキテクチャ**をもたらすこと哲学として構成されています。

公式ドキュメントのOverViewには、以下の概念が項目として挙げられています。
- Controllers
- Providers
- Modules
- Middleware
- Exception filters
- Pipes
- Guards
- Interceptors
- Custom Decorator

このうち、**Controllers**、**Providers**、**Modules**はクライアントからのリクエストに対するルーティングやビジネスロジック、またそれらをまとめるための機能を提供します。
**Middleware**、**Exception filters**、**Pipes**、**Guards**、**Interceptors**はリクエストとレスポンスの経路上で様々な役割を果たします。
次の章から、これらの機能について詳しく見ていきましょう。