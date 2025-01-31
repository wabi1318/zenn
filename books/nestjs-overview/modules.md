---
title: "Module"
---
ControllerやProvider（Service）をまとめ、１つの機能として利用できるセットとして登録する役割を持つものです。
アプリケーションには必ず１つのルートモジュールと０個以上のFeatureモジュールがありますが、コードの整理のためにFeatureモジュールを用いることが強く推奨されています。

## 定義方法
1. クラスに@Moduleデコレーターをつける。
2. @Moduleデコレーターの各プロパティを記述する。
   - providers
    DI(Dependency Injection)をするためのプロパティ。```@Injectable```デコレーターがついたクラス（＝Provider）を記述することで、そのクラスを使用することができる。
   - controllers
    Controllerを使用するためのプロパティ。```@Controller```デコレーターがついたクラスを記述する。
   - imports
    モジュール内部で必要な外部モジュールを記述するためのプロパティ。Featureモジュールをルートモジュールに追加する場合もここを利用する。
   - exports
    外部モジュールで利用したいものを記述する。

### Featureモジュール
いままで見てきた通り、同じ機能に属するControllerやServiceはFeatureモジュールに登録する必要があります。
このように特定の機能に関連するコードを整理することで、SOLID原則に従って開発できるようになります。
```ts:cats/cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
```

さらに、この```cats.module.ts```もまたRootモジュールに登録する必要があります:
```ts:app.module.ts
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```
これで、catsに関連する機能が使用できるようになります。

### Sharedモジュール
NestJSのModuleはデフォルトでシングルトン（一度だけインスタンス化でき、グローバルにアクセスできるようなクラス）になっているため、同じProviderのインスタンスを複数のModule間で共有することができます。
![Shared Module](/images/nestjs-overview/shared-module.png)
引用:https://docs.nestjs.com/modules#shared-modules

具体的なコードを見てみましょう。
例えば、CatsServiceのインスタンスを他のモジュールと共有したいとします。そのためにはまず、まず```CatsService```をエクスポートする必要があります:
```ts:cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

これを```PetModule```で使えるようにするために、```CatsModule```をインポートします:
```ts:pets/pets.module.ts
import { Module } from '@nestjs/common';
import { AnimalService } from '../animals/animals.service';
import { CatsModule } from '../cats/cats.module';

@Module({
  imports: [CatsModule],
  providers: [AnimalService],
  exports: [AnimalService],
})
export class CatsModule {}
```

もちろん、この```PetModule```も、使うためにはRootモジュールに登録する必要があります:
```ts:app.module.ts
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule, PetsModule],
})
export class AppModule {}
```

## Dependency Injection
DIとは、オブジェクト（クラス）同士の依存を外部から設定する設計方針のことです。
上で見てきた例では、

Dependency Injectionについての詳しい説明は、[Dependency InjectionをJSで理解する](https://zenn.dev/zhenyou620/articles/dependency-injection)をご覧ください。

## 作成コマンド
```bash
nest g module <name>
```
これを実行することで、Featureモジュールの作成に加え、ルートモジュールにFeatureモジュールが自動で追加されます。