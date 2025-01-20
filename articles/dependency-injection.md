---
title: "Dependency Injectionとは"
emoji: "💉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [JavaScript]
published: true
---
# Dependency Injectionとは
Dependency Injection（DI: 依存性注入）とは、オブジェクト（クラス）同士の依存を外部から設定する設計方針のことです。
この記事では、JSで書かれたFizzBuzzの例を使い、DIの基本概念を学びます。

## DIがない場合
まずは、依存性注入を行わない例を見てみましょう。以下は、1から100までのFizzBuzzをコンソールに出力するクラスです:
```js:fizzbuzz-converter.js
export class FizzBuzzConverter {
  convert(number) {
    if (number % 15 === 0) return "FizzBuzz";
    if (number % 3 === 0) return "Fizz";
    if (number % 5 === 0) return "Buzz";
    return number.toString();
  }
}
```

```js:fizzbuzz-printer.js
import { FizzBuzzConverter } from "./fizzbuzz-converter.js";

export class FizzBuzzPrinter {
  printRange(start, end) {
    const converter = new FizzBuzzConverter(); // 依存関係を内部で生成
    for (let i = start; i <= end; i++) {
      console.log(`${i}: ${converter.convert(i)}`);
    }
  }
}
```

```js:app.js
import { FizzBuzzPrinter } from "./fizzbuzz-printer.js";

// 実行
const printer = new FizzBuzzPrinter();
printer.printRange(1, 100);
```

このコードは確かに動きますが、```FizzBuzzPrinter```は```printRange```メソッドの内部で```FizzBuzzConverter```のインスタンスを自力で生成しています。
```FizzBuzzPrinter```が```FizzBuzzConverter```に直接依存しているため、もし```FizzBuzzConverter```を変更すると、```FizzBuzzPrinter```にも直接影響が及びます。もし、この```FizzBuzzPrinter```の単体テストを行うとすると、 ```FizzBuzzConverter```の変更によってテストが失敗してしまう可能性もあります。

また、このプログラムは文字を```console.log```で出力します。プログラム外部にデータを出力する機能は、通常の関数の戻り値や内部状態とは異なり、プログラム外部に影響を与えるため、単体テストが難しい機能だと言えます。```FizzBuzzPrinter```の単体テストを行いたいのに、```FizzBuzzConverter```の実装も含めてテストしなければならないのは、少し大変かもしれません。

## DIを使った改善
これらの課題を解決するには、クラス内部から取り除くことが有効です。クラスには、固有のロジックとそれを実現するための設計だけを残すようにします。

以下のように設計を修正してみましょう:
- ```FizzBuzzPrinter```のコンストラクタで```FizzBuzzConverter```と```ConsoleOutput```を受け取る
- ```FizzBuzzPrinter```は、具体的な```FizzBuzzConverter```や```ConsoleOutput```の詳細な実装を知らない
```js:fizzbuzz-converter.js
export class FizzBuzzConverter {
  convert(number) {
    if (number % 15 === 0) return "FizzBuzz";
    if (number % 3 === 0) return "Fizz";
    if (number % 5 === 0) return "Buzz";
    return number.toString();
  }
}
```

```js:fizzbuzz-printer.js
export class FizzBuzzPrinter {
  constructor(converter, output) {
    this.converter = converter;
    this.output = output;
  }

  printRange(start, end) {
    for (let i = start; i <= end; i++) {
      const text = this.converter.convert(i);
      this.output.write(`${i}: ${text}`);
    }
  }
}
```

```js:console-output.js
export class ConsoleOutput {
  write(data) {
    console.log(data);
  }
}
```

```js:app.js
import { FizzBuzzConverter } from "./fizzbuzz-converter.js";
import { ConsoleOutput } from "./console-output.js";
import { FizzBuzzPrinter } from "./fizzbuzz-printer.js";

// 実行
const converter = new FizzBuzzConverter();
const output = new ConsoleOutput();
const printer = new FizzBuzzPrinter(converter, output);
printer.printRange(1, 100);
```

この設計を採用すれば、モックオブジェクトを使って、「依存を意図どおり呼び出していればよい」（＝「```FizzBuzzConverter```と```ConsoleOutput```を意図通り呼び出していればよい」）というテストシナリオを書くことができ、テストをシンプルにすることができます。
以下はJestを用いた```FizzBuzzPrinter```のテストコードです:

```js:test.js
import { FizzBuzzPrinter } from "./fizzbuzz-printer.js";
import { jest } from "@jest/globals";

describe("FizzBuzzPrinter", () => {
  let mockConverter;
  let mockOutput;

  beforeEach(() => {
    mockConverter = {
      convert: jest.fn(),
    };
    mockOutput = {
      write: jest.fn(),
    };
  });

  test("範囲が無効なとき、呼び出さない", () => {
    const printer = new FizzBuzzPrinter(mockConverter, mockOutput);
    printer.printRange(0, -1);

    expect(mockConverter.convert).not.toHaveBeenCalled();
    expect(mockOutput.write).not.toHaveBeenCalled();
  });

  test("1〜3の範囲でFizzBuzz", () => {
    mockConverter.convert.mockImplementation((number) => {
      const map = {
        1: "1",
        2: "2",
        3: "Fizz",
      };
      return map[number];
    });

    const printer = new FizzBuzzPrinter(mockConverter, mockOutput);
    printer.printRange(1, 3);

    expect(mockConverter.convert).toHaveBeenCalledTimes(3);
    expect(mockOutput.write).toHaveBeenCalledTimes(3);

    expect(mockOutput.write).toHaveBeenNthCalledWith(1, "1: 1");
    expect(mockOutput.write).toHaveBeenNthCalledWith(2, "2: 2");
    expect(mockOutput.write).toHaveBeenNthCalledWith(3, "3: Fizz");
  });
});
```
ここでは、モックオブジェクトを使用して、実際の出力や変換の動作をシミュレートしています。これにより、`FizzBuzzPrinter`が正しく動作するかを簡単に検証できます。
このように、外部からインスタンスを与えられる形にしたものは、自身の責務 (=ビジネスロジック) にのみ集中できる閉じた形になります。つまり、「オブジェクトのインスタンスを生成して構築すること」が別の責務に分離されたということです。

では、「オブジェクトのインスタンスを生成する部分」を考えていきましょう。
上の例では```app.js```でオブジェクトを生成していましたが、これをファクトリクラスを用いて以下のように分離してみます:

```js:fizzbuzz-factory.js
import { FizzBuzzConverter } from "./fizzbuzz-converter.js";
import { FizzBuzzPrinter } from "./fizzbuzz-printer.js";
import { ConsoleOutput } from "./console-output.js";

export class FizzBuzzFactory {
  createFizzBuzzPrinter() {
    return new FizzBuzzPrinter(this.createFizzBuzz(), this.createOutput());
  }

  createFizzBuzz() {
    return new FizzBuzzConverter();
  }

  createOutput() {
    return new ConsoleOutput();
  }
}
```

```js:app.js
import { FizzBuzzFactory } from "./fizzbuzz-factory.js";

// 実行
const factory = new FizzBuzzFactory();
const printer = factory.createFizzBuzzPrinter();
printer.printRange(1, 100);
```
「オブジェクトのインスタンスを生成する部分」をすべてファクトリクラスのメソッドに置き換えて考えることで、ほとんどのコードが、生成されたオブジェクトを使うことしか考えなくて済むようになります。これにより、各クラスは自分の依存オブジェクトを外部から受け取り、他のクラスの依存としても簡単に組み込むことができます。
この考え方を繰り返し適用することで、全体の構造が単純化されます。アプリケーション全体がこの方法で構築されると、依存関係が連鎖的に整理され、最終的には一つのルートオブジェクトに辿り着きます。
このように、依存オブジェクトは外部から与えるという形を徹底する考え方が、**Dependency Injection**です。
依存オブジェクトの中身の詳細がどうなっているかを意識せず、最小限のインターフェースで端的に「使うだけ」を意図するコードを書くことで間違いが減り、単体テストも簡単になるということです。

# リポジトリ
この記事内で用いたコードは、以下のリポジトリに格納しています。
https://github.com/zhenyou620/dependency-injection

# 参考
- [ちょうぜつソフトウェア設計入門 ――PHPで理解するオブジェクト指向の活用]([https://gihyo.jp/book/2022/978-4-297-13234-7](https://gihyo.jp/book/2022/978-4-297-13234-7))
- [Getting Started](https://jestjs.io/docs/getting-started)
- [ECMAScript Modules](https://jestjs.io/docs/ecmascript-modules)

