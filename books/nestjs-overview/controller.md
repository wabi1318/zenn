---
title: "Controller"
---
Controllerとは、クライアントからのリクエストを受け取り、レスポンスを返す役割を持つものです。
つまり、NestJSにおけるルーティングの機能を担っています。
![クライアントとコントローラーの関係図](/images/nestjs-overview/controller.png)
引用: https://docs.nestjs.com/controllers

なお、Controllerは必ずModuleクラスに属します。

# 定義方法
1. クラスに@Controller()デコレータをつける。
2. メソッドにHTTPメソッドデコレーターをつける。

```ts:cats.controller.ts
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post() // POST /cats
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get() // GET /cats?limit=XX
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id') // GET /cats/:id
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id') // PUT /cats/:id
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id') // DELETE /cats/:id
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
```

Controllerを使用するためには、Moduleへの登録が必要です。
```ts:cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController], // Controller の登録
  providers: [CatsService],
})
export class CatsModule {}
```


# 作成コマンド
```
nest g controller <name>
```
これを実行することで、Controllerの作成だけでなく、関連するFeatureモジュールにControllerが自動で登録されます。