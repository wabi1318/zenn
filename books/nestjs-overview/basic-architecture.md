---
title: "NestJSの基本的なアーキテクチャ"
---
前章で挙げた概念のうちControllers、Providers、ModulesはNestJSのアーキテクチャの基本をなす概念です。
このアーキテクチャを図にすると以下のようになります。

![NestJSのアーキテクチャ](/images/nestjs-overview/architecture.png)

### main.ts
アプリケーションインスタンスを作成するエントリーポイントです。
ここに、app.module.tsを登録します。

### app.module.ts
NestJSの**Rootモジュール**です。
Rootモジュールとは、開発者の作成した各機能（Feature）をNestJSが扱えるようにするためのモジュールです。
各Featureは、このRootモジュールに登録しないと使えません。

### feature.module.ts, feature.service.ts, feature.controller.ts
これら３つのファイルがひとまとまりになり、１つの機能を実現します。
アプリケーションの規模が大きくなり、機能が増えるほど、このまとまりの数が増えていくことになります。

では、Controllers、Providers、Modulesについて詳しく見ていきましょう。