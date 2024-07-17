---
title: "コーディング規約（仮）"
emoji: "🤝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript"]
published: true
---
これがよいのではと思った規約をまとめました。
適宜更新予定。

# 命名規則
|種別|規約|例|備考|
|---|---|---|---|
|変数|先頭小文字 (camel)|var fooVar;||
|関数|先頭小文字 (camel)|function barFunc() { }||
|クラス|先頭大文字 (Pascal)|class Foo { }||
|インターフェース名|先頭大文字 (Pascal)|interface Foo {}|先頭にIを付けてはいけない|
|インターフェースメンバ|先頭小文字 (camel)|interface Foo { foo: string }||
|型エイリアス名|先頭大文字 (Pascal)|type Foo {}||
|型エイリアスメンバ|先頭小文字 (camel)|type Foo { foo: string }||
|名前空間|先頭大文字 (Pascal)|namespace Foo {}||
|Enum|先頭大文字 (Pascal)|enum Color {}||
|ファイル名|コンポーネントファイル：先頭大文字 (Pascal)<br>その他のファイル：先頭小文字 (camel)|コンポーネント：Component.tsx<br>その他：index.tsx||

具体的な命名の仕方に悩んだ場合は、[Naming cheatsheet](https://github.com/kettanaito/naming-cheatsheet)が参考になります。

# コーディング規約
1. 1ファイルにつき記述できるコンポーネントは1つ
2. ファイル名とコンポーネント名を一致させる
3. ディレクトリと同名で参照したいコンポーネントを作る際のファイル名はindex.tsxにする
4. default exportにする際にも、必ずコンポーネントには名前をつける
5. 関数を記述するときは、アロー構文を使う
6. ```var```は使わず、```let```または```const```を使う
7. 配列を使用するときは、```foos: Array<Foo>```の代わりに```foos: Foo[]```として配列にアノテーションをつける
8. コンポーネントの設計は、Container/Presentationalパターンで行う。

## 1. 1ファイルにつき記述できるコンポーネントは1つ
### よい例
```tsx:MyComponent.tsx
import { FC } from 'react';

export const MyComponent: FC = () => {
  return (
    <div>
      <h1>Hello, world!</h1>
    </div>
  );
};
```

### 悪い例
```tsx:MyComponent.tsx
import { FC } from 'react';

export const MyComponent: FC = () => {
  return (
    <div>
      <h1>Hello, world!</h1>
    </div>
  );
};

export const AnotherComponent: FC = () => {
  return (
    <div>
      <p>This is another component in the same file.</p>
    </div>
  );
};
```

## 2. ファイル名とコンポーネント名を一致させる
### よい例
```tsx:Header.tsx
import { FC } from 'react';

export const Header: FC = () => {
  return (
    <header>
      <h1>Welcome to my site</h1>
    </header>
  );
};
```

### 悪い例
```tsx:headerComponent.tsx
import { FC } from 'react';

export const Header: FC = () => {
  return (
    <header>
      <h1>Welcome to my site</h1>
    </header>
  );
};
```

## 3. ディレクトリと同名で参照したいコンポーネントを作る際のファイル名はindex.tsxにする
### よい例
```tsx:Footer/index.tsx
import { FC } from 'react';

export const Footer: FC = () => {
  return (
    <footer>
      <p>&copy; 2024 My Company</p>
    </footer>
  );
};
```
```tsx:App.tsx
import { FC } from 'react';
import Footer from './Footer';

const App: FC = () => {
  return (
    <div>
      {/* other components */}
      <Footer />
    </div>
  );
};
```

### 悪い例
```tsx:Footer/Footer.tsx
import { FC } from 'react';

export const Footer: FC = () => {
  return (
    <footer>
      <p>&copy; 2024 My Company</p>
    </footer>
  );
};
```
```tsx:App.tsx
import { FC } from 'react';
import Footer from './Footer/Footer';

const App: FC = () => {
  return (
    <div>
      {/* other components */}
      <Footer />
    </div>
  );
};
```

## 4. default exportにする際にも、必ずコンポーネントには名前をつける
### よい例
```tsx:Button.tsx
import { FC } from 'react';

const Button: FC = () => {
  return (
    <button>Click me</button>
  );
};

export default Button;
```

### 悪い例
```tsx:Button.tsx
export default () => {
  return (
    <button>Click me</button>
  );
};
```

## 5. 関数を記述するときは、アロー構文を使う
### よい例
```tsx
const add = (a: number, b: number): number => {
  return a + b;
};

const greet = (name: string): string => `Hello, ${name}!`;
```

### 悪い例
```tsx
function add(a: number, b: number): number {
  return a + b;
}

function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

## 6. varは使わず、letまたはconstを使う
### よい例
```tsx
const PI = 3.14;

let counter = 0;
counter += 1;
```

### 悪い例
```tsx
var PI = 3.14;

var counter = 0;
counter += 1;
```

## 7. 配列を使用するときは、foos: Array<Foo>の代わりにfoos: Foo[]として配列にアノテーションをつける
### よい例
```tsx
interface Foo {
  id: number;
  name: string;
}

const foos: Foo[] = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
];
```

### 悪い例
```tsx
interface Foo {
  id: number;
  name: string;
}

const foos: Array<Foo> = [
  { id: 1, name: "Alice" },
  { id: 2, name: "Bob" },
];
```

## 8. コンポーネントの設計は、Container/Presentationalパターンで行う。
Container/Presentationalパターンは、コンポーネントベースのフロントエンド開発で使用される設計パターンのこと。
ロジックとUIを分離して実装することで、***関心の分離***を図り、コードの再利用性と保守性を向上させることができる。

このパターンは以下の２種類のコンポーネントに分かれる。
#### 1. Container Component
- アプリケーションのロジックや状態管理を担当
- データの取得や更新、イベントハンドリングなどを行う
- 通常、状態を持ち、ライフサイクルメソッドを使用する
#### 2. Presentational Component
- UIの表示のみを担当
- propsを通じて受け取ったデータを表示する
- 通常、状態を持たず、純粋な関数コンポーネントとして実装される

### コード例
#### Container Component
```tsx:TaskBox.tsx
import { FC, useEffect, useState } from 'react';
import { TaskList } from "src/components/TaskList";

export const TaskBox: FC = () => {
  const [tasks, setTasks] = useState([]);

  useEffect(() => {
    fetch('api/GetTasks')
      .then((response) => response.json())
      .then((tasks) => setTasks(tasks));
  }, []);

  return <TaskList tasks={tasks} />;
};
```
#### Presentational Component
```tsx:TaskList.tsx
import { FC } from 'react';

type Props = {
  tasks: {
    id: number;
    title: string;
    completed: boolean;
  }[];
};

export const TaskList: FC<Props> = ({ tasks }) => {
  return (
    <ul>
      {tasks.map(({ id, title, completed }) => (
        <li key={id}>
          <p>{title}</p>
          <p>{completed ? '完了' : '未完了'}</p>
        </li>
      ))}
    </ul>
  );
};
```

# 参考
https://oukayuka.booth.pm/items/2367992
https://typescript-jp.gitbook.io/deep-dive/styleguide
https://github.com/kettanaito/naming-cheatsheet
https://airbnb.io/javascript/react/
