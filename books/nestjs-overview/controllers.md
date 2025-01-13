---
title: "Controllerとは"
---
## Controllerとは
Controllerとは、クライアントからのリクエストを受け取り、レスポンスを返す役割を持つものです。
つまり、NestJSにおけるルーティングの機能を担っています。なお、Controllerは必ずModuleクラスに属します。
![クライアントとコントローラーの関係図](/images/nestjs-overview/controller.png)
引用: https://docs.nestjs.com/controllers

## Controllerの定義方法
Controllerは以下のように定義されます。
1. クラスに@Controller()デコレータをつける。
2. メソッドにHTTPメソッドデコレーターをつける。

以下は具体的なコードの例です。コメントに実際のパスを記述しています。
```ts:cats.controller.ts
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post() // POST /cats
  create(@Body() createCatDto: CreateCatDto) {
    return `This action adds a new cat: ${JSON.stringify(createCatDto)}`;
  }

  @Get() // GET /cats?limit=XX
  findAllWithLimit(@Query('limit') limit: string) {
    return `This action returns all cats (limit: ${limit} items)`;
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
```@Get()```, ```@Post()```, ```@Put()```, ```@Delete()```は、対応するHTTPリクエストメソッドのデコレータです（デコレータについては、「Custom Decoratorとは」の章を参照してください）。

```@Controller```に```cats```のようなパスプレフィックスを使用することで、関連するルーティングを簡単にグループ化することができます。
上記の例では全て```cats```がパスの最初に来ています。

## ルートパラメータ
動的なルートパラメータ（パスパラメータ）を使用したい場合は、対応するパスのHTTPメソッドデコレータにルートパラメータトークン（ex: ```:id```）を使用します。
このルートパラメータは```@Param```デコレータを使用してアクセスすることができます。

### コード例
```ts
  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }
```

### リクエスト例
```bash
GET /cats/123
```

### レスポンス
```text
This action returns a #123 cat
```

## クエリパラメータ
動的なクエリパラメータを使用したい場合は、対応するパスのHTTPメソッドデコレータに```@Query()```デコレータを使用します。

### コード例
```ts
  @Get()
  findAllWithLimit(@Query('limit') limit: string) {
    return `This action returns all cats (limit: ${limit} items)`;
  }
```

### リクエスト例
```bash
GET /cats/list?limit=5
```

### レスポンス
```text
This action returns all cats (limit: 5 items)
```

## リクエストボディ
HTTPリクエストのボディに含まれるデータを取得するためには、```@Body```デコレータを使用します。

### コード例
```ts
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return `This action adds a new cat: ${JSON.stringify(createCatDto)}`;
  }
```

### リクエスト例
リクエストURL:
```bash
POST /cats
```

リクエストボディ（JSON）:
```json
{
    "name": "Ran",
    "age": "18",
    "breed": "Mixed"
}
```

### レスポンス
```text
This action adds a new cat: {"name":"Ran","age":"18","breed":"Mixed"}
```

なお、Controllerを使用するためには、Moduleへの登録が必要です。
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

## 作成コマンド
```bash
nest g controller <name>
```
これを実行することで、Controllerの作成だけでなく、関連するFeatureモジュールにControllerが自動で登録されます。