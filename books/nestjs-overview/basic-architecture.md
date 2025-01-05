---
title: "NestJSの基本的なアーキテクチャ"
---
## NestJSの基本的なアーキテクチャ
前章で挙げた概念のうちControllers、Providers、ModulesはNestJSのアーキテクチャの基本をなす概念です。
このアーキテクチャを図にすると以下のようになります。なお、ServicesはProvidersの一種です（詳しくは「Provider（Service）とは」の章を参照してください）。

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

Modules、Services、Controllersについて一言で説明すると、以下のようになります。
- Modules: Featureをグループ化する役割を持つもの。
- Services: ビジネスロジックを定義するもの（システム固有の処理の集まり）。
- Controllers: ルーティングの機能を持つもの。

この後さらに詳しく見ていくため、今は「そんなもんか〜」くらいの理解で構いません。

## 具体例
ここで、具体的なイメージを掴むために、猫の種類の登録や取得ができるアプリケーションを考えてみましょう。
これをNestJSのアーキテクチャで考えると、以下のような構成になります。
![猫の種類の登録や取得ができるアプリケーションのアーキテクチャ](/images/nestjs-overview/architecture-cat.png)

Serviceに猫の登録や取得のロジックを記述し、Controllerでルーティングを行い、Moduleでこれらを取りまとめる、といった構成になります。
実際のコードを見てみましょう。

```ts:cats.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class CatsService {
  constructor() {
    this.cats = [];
  }

  create(cat) {
    this.cats.push(cat);
  }

  findAll() {
    return this.cats;
  }
}
```

```ts:cats.controller.ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

```ts:cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

このアプリケーションに、犬の種類を登録・取得できる機能を追加すると、以下のようなアーキテクチャになります。
![犬の種類の登録や取得ができる機能を追加した場合のアーキテクチャ](/images/nestjs-overview/architecture-dog.png)
このように、機能が追加されればされるほど、Featureのまとまりの数が増えていきます。