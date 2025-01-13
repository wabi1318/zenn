---
title: "DTO"
---
今まで見てきたControllerの中に、以下のような記述がありましした。
```ts:cats.controller.ts
  @Post() // POST /cats
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }
```
ここでいきなり出てきた```CreateCatDto```とはなんでしょうか？
これは、DTO（Data Transfer Object）と呼ばれるものです。

DTOは、データをカプセル化し、クライアントとサーバー、またはサーバー内の異なるレイヤー間でのデータ転送を標準化するオブジェクトです。TypeScriptでのinterfaceと似ていますが、DTOは以下の点で異なります。

- interfaceは型チェックと構造定義に使用され、コンパイル後はJavaScriptから消える。
- DTOはJavaScriptの機能の一つであるクラスとして定義されるため、コンパイル後も残る。
- DTOは型チェックと構造定義に加え、データバリデーションを行うこともできる。

なおDTOはデータ構造を定義し、具体的な処理（ビジネスロジックなど）を含みません。

## 使用例
DTOは、さまざまなドメインオブジェクトからのデータを整理したり、ドメインオブジェクトからのデータの一部のみを取得したりできるように設計します。
さらに、データのバリデーションやシリアル化のロジックのカプセル化にも役立てることができます。
例えば、以下のような例を考えてみましょう。

### 1. 複数のデータをまとめる
ユーザー登録フォームなどでは、複数のドメインオブジェクトからデータを受け取ることがあります。
DTOを使用すると、このデータを1つのオブジェクトにカプセル化し、必要に応じてアプリケーション内で渡すことができます。

### 2. 必要なデータだけを抽出する
APIのレスポンスでは、ドメインオブジェクトからデータのサブセットのみを返したいことがあります。
DTOを使用すると、返すプロパティのみを含むクラスを定義し、APIエンドポイントからこのDTOのインスタンスを返すことができます。

### 3. バリデーション
クライアントからデータを受けとる場合、データを処理する前に、データが正しい形式であるかどうかを検証する必要がある場合があります。
DTOを使用すると、DTOクラス自体に検証ロジックを定義できるため、重複を減らしてコードの保守性を高めることができます。
NestJSにおいてバリデーションを行うには```ValidationPipe```を使用する必要があります。詳しくはPipesの章をご覧ください。

## 基本的な使い方
NestJSでDTOを使用する場合、以下のようになります。
1. DTOクラスを定義する
```ts
export class CreateUserDto {
  readonly name: string;
  readonly email: string;
  readonly password: string;
}
```

2. ControllerでDTOクラスを使用する
```ts
import { Controller, Post, Body } from '@nestjs/common';
import { CreateUserDto } from './create-user.dto';

@Controller('users')
export class UsersController {
  @Post()
  async create(@Body() createUserDto: CreateUserDto) {
    // ユーザー作成ロジック
    // …
  }
}
```
ここでは、```@Body()```デコレーターを使用してリクエストボディをDTOにバインドしています。

## 具体的な使用例
では、さらに具体的な使用方法について見ていきましょう。
以下は、Userエンティティと、それを元に必要な情報だけを切り出すDTOの定義です。
```ts
export class User {
  id: number;
  name: string;
  email: string;
  password: string;
}

export class UserProfileDto {
  name: string;
  email: string;
}
```

そして、以下のようなServiceでユーザー情報を取得します。
```ts
export class UserService {
 getUserById(userId: number): User {
    // データベースからユーザー情報を取得
    // …
    return user;
  }
}
```

```UserService```を使用してUserエンティティを取得し、DTO（```UserProfileDto```）を使用してデータのサブセットを返すControllerを定義します。
```ts
@Controller('users')
export class UsersController {
  constructor(private readonly userService: UserService) {}

  @Get(':id/profile')
  getUserProfile(@Param('id') userId: number): UserProfileDto {
    const user = this.userService.getUserById(userId);

    // 名前とメールのプロパティのみを含むDTOオブジェクトを作成
    const userProfileDto = new  UserProfileDto(); 
    userProfileDto.name = user.name; 
    userProfileDto.email = user.email; 

    // DTOオブジェクトをAPIレスポンスとして返却
    return userProfileDto; 
  }
}
```
これで、必要なデータだけを抽出しサブセットとして返すことができました。

さらに、ユーザー情報と、最新のブログ投稿を一緒に取得できるようなアプリケーションを考えてみましょう。
以下のように、ユーザーと投稿データを表すエンティティ、それらをまとめたDTOを定義します。
```ts
export class User {
  id: number;
  name: string;
  email: string;
}

export class Post {
  id: number;
  title: string;
  content: string;
  userId: number;
}

export class UserWithLatestPostDto {
  name: string;
  email: string;
  latestPostTitle: string;
  latestPostContent: string;
}
```

そして、以下のようなServiceでユーザー情報と最新のブログ投稿を取得します。
```ts
export class UserService {
  getUserWithLatestPost(userId: number): { user: User, latestPost: Post } {
    // データベースからユーザー情報を取得
    // …
    
    // データベースからユーザーの最新の投稿を取得する
    // …

    return { user, latestPost };
  }
}
```

```UserService```を使用してユーザー情報と最新の投稿を取得し、DTO（```UserWithLatestPostDto```）を使用してデータを返すControllerを定義します。
```ts
@Controller('users')
export class UsersController {
  constructor(private userService: UserService) {}

  @Get(':id/with-latest-post')
  getUserWithLatestPost(@Param('id') userId: number): UserWithLatestPostDto {
    const { user, latestPost } = this.userService.getUserWithLatestPost(userId);

    // ユーザーデータと最新の投稿を含むDTOオブジェクトを作成する
    const userWithLatestPostDto = new UserWithLatestPostDto();
    userWithLatestPostDto.name = user.name;
    userWithLatestPostDto.email = user.email;
    userWithLatestPostDto.latestPostTitle = latestPost.title;
    userWithLatestPostDto.latestPostContent = latestPost.content;

    // DTOオブジェクトをAPIレスポンスとして返却
    return userWithLatestPostDto;
  }
}
```
これで、複数のドメインオブジェクトをまとめ、クライアントに必要なデータのみを返すことができました。