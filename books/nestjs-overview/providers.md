---
title: "Provider（Service）とは"
---
Providerとは、他のアプリケーションコンポーネントにサービスのインスタンスを提供するオブジェクトのことです。つまり、Providerは、オブジェクトインスタンスの作成、管理、提供の役割を担います。

「NestJSの基本的なアーキテクチャ」の章で示した図のうち、「Featureサービス」がProviderに当たります。
![NestJSのアーキテクチャ](/images/nestjs-overview/architecture.png)

代表的なProviderはServiceです。Serviceとは、ビジネスロジックを定義したものです。
形式的には、以下のように定義します。
1. クラスに@Injectable()デコレーターをつける。
2. メソッドを作成する。

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

この```cats.service.ts```を他のコンポーネントで使用するには、対応するModuleに登録する必要があります。
```ts:cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService], // Serviceの登録
})
export class CatsModule {}
```

Moduleに登録した```cats.service.ts```を、```cats.controller.ts```で使用したい場合、コンストラクタの引数にServiceを取ることで実現できます。
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

# 作成コマンド
```bash
nest g service <name>
```
これを実行することで、Serviceが作成されるだけでなく、関連するFeatureモジュールにServiceが登録されます。