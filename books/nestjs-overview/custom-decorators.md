---
title: "Custom Decoratorとは"
---
## Decoratorとは
:::message
以下のサンプルコードは、わかりやすさのためTypeScript5.0以降で用いることができるDecorator機能を使用しています。
:::

Decoratorとは、クラスのメンバに対して処理を付与できる仕組みです。以下のコードを見てみましょう: 
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ron");
p.greet();
```

このシンプルな```greet()```メソッドに、デバッグ用の```console.log```をいくつか用いたいとします:
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    greet() {
        console.log("LOG: Entering method.");

        console.log(`Hello, my name is ${this.name}.`);

        console.log("LOG: Exiting method.")
    }
}
```

これを、他のメソッドでも実現するのにDecoratorが便利です。
以下はこのデバッグ用のDecoratorを定義するコードです:
```ts
function loggedMethod(originalMethod: any, _context: any) {

    function replacementMethod(this: any, ...args: any[]) {
        console.log("LOG: Entering method.")
        const result = originalMethod.call(this, ...args);
        console.log("LOG: Exiting method.")
        return result;
    }

    return replacementMethod;
}
```

これを```greet```に適用させるには、```greet```の上に```@loggedMethod```と記述します:
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    @loggedMethod
    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ron");
p.greet();

// Output:
//
//   LOG: Entering method.
//   Hello, my name is Ron.
//   LOG: Exiting method.
```

##　カスタムデコレータ
これまで見てきた通り、NestJSではDecoratorが多く出てきました。このようにNestJSは、Decoratorをフレームワークの中心的なコンセプトとして採用しています。
そして、これまでに出てきたDecoratorに加え、NestJSでは独自のCustom Decoratorを作成することができます。




##　余談
```createParamDecorator```の内部実装を確認したところ、

```
```
ECMAScriptで提案されていたデコレータ