---
title: intro and background
cssclasses:
  - zenn
date: 2024-06-19
modified: 2024-06-19
AutoNoteMover: disable
tags:
  - type/zenn/book
  - TypeScript/type
---
[Reconstructing TypeScript 元記事](https://jaked.org/blog/2021-09-07-Reconstructing-TypeScript-part-0)

私は [Programmable Matter](https://github.com/jaked/programmable-matter)  という「ドキュメント開発環境（``document development environment``)」- ドキュメント内でのライブコーディングをシンプルで TypeScript のようなプログラミング言語で行える開発環境 - の開発をしています。TypeScript の型システムは珍しくてとても面白く、実装方法を考えるのは楽しいものでした。

 実際の TypeScript で書かれたこの言語(Programmable Matter内で使える言語)を部分的にサポートする型チェッカーを提示することで、TypeScript の何が面白くて珍しいのかを掘り下げていこうと思います。小さな部分的な機能から始まり、回を重ねるごとに少しずつ機能を追加していきます。この最初の投稿では、コードの理解に役立つ前提知識の紹介をします。

## 型チェッカーとは何か？

これ以降の記事では読者がこれまでに型チェッカー(type checker)を使ったことがあり、型(type)と型チェック(type checking)の意味を理解していることを前提にして話を進めていきます。ただし最初にこれらの概念の簡単な紹介をしましょう。

JavaScriptでは、変数はさまざまな型の値を持てます。 `x` が保持する値の型が分かっておらず、`string`か`boolean`のどちらかであることだけが判明している場合について考えてみましょう。

```js
if (typeof x === 'string') {
  ...
} else {
  ...
}
```

この例のように `x` の型をテストをすると、(実行時に) テストが成功した場合、`x` の値の型は必ず `string` であると（開発時に)推論できます。これにより、true ブランチでは `x` を `string` として、 false ブランチでは `x` を `boolean` として、安全に扱うことができます。

プログラマーの主な仕事はプログラムの動作についての推論です。主に頭の中で行っているこの作業を自動化（間違いを見つけたり、コード補完のような対話的な開発サポートなど）できればとても便利なことでしょう。型チェッカーはそのための、実行時のプログラムの動作を開発時に自動で推論する手法です。

人間の推論は任意に創造的で複雑なものですが、型チェッカーは単なるプログラムなので可能な「推論」には限界があります。型チェッカーは、例えばプログラムがリストを正しくソートしているかどうかの推論はできません。しかし、値に対してサポートされていない操作 (存在しないプロパティへのアクセスなど) を実行していないかのチェックはできます。

型チェッカーは開発時に実行されるため、実行時にプログラム内を流れる実際の値を確かめることはできません。その代わりに型チェッカーは、プログラムの各式についてその式で計算される可能性のある値をひとまとめにし、これを元にすべての値に共通する属性（どのような演算がサポートされているか、演算によってはどのような結果を返すか）を記述した型をその式に与えています。

例を出しましょう。

```typescript
const vec = { x: x, y: y };
```

この例において、もし `x` と `y` が `number` 型なら、 `vec` の型は

```typescript
{ x: number, y: number }
```

です。

プログラムが実行されると、 `x` と `y` は様々な値を取る可能性があり、`vec` もまた様々な値を取る可能性があります。それでも、 `vec` の値は全て `x` と `y` プロパティへのアクセスをサポートしていて、そのような値に対して `typeof` を実行した結果は全て `"object"` になります。

もしプログラムに `vec.z` という式が含まれていた場合、型チェッカーはサポートされていない操作とみなしてエラーを出します。型チェッカーは、開発時に式の型に対してサポートされていない操作が実行されていないことをチェックすることで、プログラムが実行時にサポートされていない操作を具体的な値に対して実行しないことを保証するのです。
## TypeScriptは何がすごいのか?

殆どの型システムではプリミティブ型は混ぜられないので、`string` か `boolean`のどちらかを保持するような変数を作ることが出来ません。ただその場合でも通常は特定の複合データ型を混ぜて値の型をテストする方法が存在します。クラスとオブジェクトのある言語では、クラス`Shape`の変数にサブクラス `Circle` や `Square` のオブジェクトを保持することができ、オブジェクトのクラスを `instanceof` （または同等のもの）でテストすることができます。また、バリアント（サムタイプやタグ付きユニオンとも言う）のある言語では、 `tree` 型の変数にバリアントアーム `Leaf` や `Node` の値を保持することができ、パターンマッチングによって値がどのアームであるかを知ることができます。

TypeScriptではどんなでも型を混ぜて使えます。もし変数  `x` が `string` もしくは `boolean` を保持することが分かっていれば、ユニオン型の `string | boolean` にできます。このようなユニオン型のテストをするとき、型チェッカーはテストの結果に応じて `if` / `else` の分岐で `x` の型の *narrow* (true ブランチでは `x` の型を `string` へ、false ブランチでは `boolean` への絞り込み)を行います。

このアイデアを推し進めれば、バリアント型の定義も出来ます。例えば葉(leaf)とノード(node)のユニオン型で出来た木型(tree type)です。

```typescript
type tree = { value: number } | { left: tree, right: tree }
```

この型によって作られた値がこちらです

```typescript
const tree = {
  left: {
    left: { value: 7 },
    right: { value: 9 }
  },
  right: { value: 11 }
}
```

 if文の条件式にて`value` プロパティの存在を確認することにより、TypeScriptは各分岐ブランチ内の型を絞り込めます(narrowing)。false ブランチでは`left` プロパティと `right` プロパティを使用しても安全だと推論出来るのです。

```typescript
function height(t: tree): number {
  if ('value' in t) return 1;
  else return 1 + Math.max(height(t.left), height(t.right));
}
```

私はユニオン型とnarrowingを使ったプログラミングが大好きです。実行時の型チェックに依存する典型的なJavaScriptのイディオムを便利にチェックできるからです。これにより殆どのJavaScriptのコードはTypeScriptへと簡単に移行が出来ます。ユニオンはクラスの階層やバリアント型に代わる、簡単で柔軟性のある魅力的な代替手段です。(この説を支持するような例をこれから記事中で挙げていきます)
## 余談: unions vs. variants

もしあなたがHaskellやOCamlにあるようなバリアント型を知っている場合、ユニオン型がそれらとはどのように違うのか疑問に思うかも知れません。OCamlでのバリアント型の定義を見てみましょう。

```ocaml
type tree =
  Leaf of { value: int } |
  Node of { left: tree; right: tree }
```

-  `tree` 型 - この型の値は `Leaf` または `Node` ですが、どちらであるかはわかりません。
- 各アームの値を作成するコンストラクター。コンストラクターは基礎となる値をタグでラップするため、実装はそれがどのアームであるかを認識できます。
- 型の値のパターン マッチングのサポート。実装では、値のタグを使用して、それがどのアームであるかを判断します。

`tree` の値は明示的なコンストラクターで記述出来ます

```ocaml
let tree = Node {
  left = Node {
    left = Leaf { value = 7 };
    right = Leaf { value = 9 }
  };
  right = Leaf { value = 11 }
}
```

パターンマッチングにより区別します

```ocaml
let rec height (tree: tree) =
  match tree with
    | Leaf { value } -> 1
    | Node { left; right } -> 1 + max (height left) (height right)
```

TypeScriptでユニオン型を定義すると、作られるのはその型だけです。前のセクションの例と同様に、ユニオン体の値を構築するには、型固有のコンストラクターを使用するのではなく、アームの 1 つの型を満たす通常の値を書き留めます。アームを識別するために、タイプ固有のパターン マッチングではなく、通常のテスト演算子と絞り込みを使用します。

ユニオン型は新しい式構文を必要としないため、柔軟で軽量です。たとえば、関数の引数が `boolean` または `string` の場合は、 関数の外で型を宣言したり、呼び出し元にコンストラクターで引数をラップするよう要求したりすることなく、引数の型内で直接実行できます。
## Reconstructing TypeScript

実のところ、実際のTypeScriptがどう動いているのかを私は知りません。どう動作するべきかを定義する[最新の仕様](https://github.com/Microsoft/TypeScript/issues/15711)は存在せず、実装を読もうとしたことはありません。また [Programmable Matter](https://github.com/jaked/programmable-matter) の言語は実際のTypeScriptとは違ったニッチな使用用途のために選びました。

このシリーズではTypeScriptの再構築を行います。非形式なドキュメンテーションや実際のTypeScriptを使った実験、関連したシステムの研究論文と型チェッカーの実装に関する予備知識、そして私が考えるあるべき姿のTypeScriptを元にして再構築したTypeScriptです。(これから実際のTypeScriptと再構築したものの間に興味深い違いがあった場合その都度取り上げていきます)

今回の記事では型チェッカーの働きについての概観的な解説を行い、次回からは実際のコードの細部を見ていきます。

## 余談: Hindley-Milner  型推論

おそらくあなたはHindley-Milner型推論という型チェック手法を聞いたことがあるかも知れません。HaskellやOCamlなど[^1]で採用されているものです。この記事で取り上げるものはHindley-Milnerではなく双方向型検査(bidirectional type checking)です。これはTypeScriptやScalaなどで採用されています。

何故Hindley-Milnerではないのでしょう？双方向型検査は実装が比較的に簡単であり、またHindley-Milnerは部分型付けやユニオン型と組み合わせるのが難しいため、TypeScriptには適していません。実際のTypeScriptの実装は双方向型検査です。
## 式から型を合成する(Synthesizing a type from an expression)

型チェッカーはプログラム中の全ての式の型を知る必要があります。それによって式の型によってサポートされていない操作をプログラムが行わないように保証することができるのです。どうやってこれらの型を算出するのでしょうか？

プログラムは木(tree)とみなせます。*抽象構文木*(*abstract syntax tree*)、略して*AST*と呼ばれるものです。[^2] 木において各式はノードであり、その部分式は子ノードであり、最小単位の式であるリテラル値は葉ノードです。

この発想を念頭に置けば、各式の型を木の葉から根へとボトムアップに算出できます

- 最小単位の式ではリテラル値に対応する型を返します。 例えば値 `"foo"` は `string` 型を持ち、値 `true` は `boolean`を持ちます。
- 複合式では部分式の型たちを探し、その型たちを式のトップレベルの操作に基づいて組み合わせます。

例として下記の式の型を算出してみましょう

```typescript
{ x: 7, y: 9 }
```

部分式である `7` と `9` の型は両方とも `number` ですよね。よって、オブジェクト式全体の型はこのようになります。

```typescript
{ x: number, y: number }
```

こういった一連の流れを、*式からの型の合成* (*synthesizing* a type *from* an expression)と呼びます。[^3]

## 部分型付け(Subtyping)

ある種類の式では、部分式の型がある意味で一致しているかチェックする必要があります。例えば関数の呼び出しでは、関数に渡される引数の型が関数が期待する引数の型と互換性があるかどうかをチェックする必要があります。ここで言う"互換性"の意味とはなんでしょう？

関数の引数の型は、その関数が引数の型に記述された操作のみを試みるという主張（型チェッカーによって検証される）に等しいです。つまり、関数に渡される式の型が引数の型と互換性があるのは、渡される式の型が関数の引数の型に記述されている操作を少なくとも**サポートしている場合であり、それ以外の操作をサポートしている場合もあります。例えば

```typescript
(vec: { x: number, y: number }) => number
```

この引数型は渡される値に `x` と `y` プロパティにアクセス出来ることを要求し、さらにそれらのプロパティが`number`型であることも要求しています。下記の型はこれらの操作(`x` と `y` プロパティへのアクセスと`number`型であること)をサポートしているので互換性があります。

```typescript
{ x: number, y: number }
{ x: number, y: number, z: number }
{ x: number, y: number, foo: string }
```

しかし下記の型は互換性がありません

```typescript
{ x: number, y: string }
{ x: number }
boolean
```

この互換性の概念を部分型と呼びます。*反射*(*reflexive*) *推移*(*transitive*)[^4]

したがって、関数呼び出しの型を合成するには、関数の型を合成します。渡された引数の型を合成します。渡された型が関数の引数の型のサブタイプであることを確認します。そして最後に関数の結果の型を返します。
## 式を型に対してチェックする Checking an expression against a type

合成とサブタイプのみを使用する型チェッカーがその役割を果たし、プログラムがサポートされていない操作を試行しないようにします。しかし、型チェッカーをより使いやすくする代替手段があります。式に期待される型がわかっている場合 (たとえば、それが関数の引数として表示され、関数の引数の型がわかっている場合)、式をチェックできます。型に対する式。

考え方は、式と型が同じ構造を持つ場合、両方を分解して、式の各部分を対応する型の部分と比較してチェックできるということです。それ以上分解できない場合は、合成とサブタイプに戻ります。

たとえば、オブジェクト式をチェックするには

```typescript
{ x: 7, y: "nine" }
```

オブジェクトタイプに対して

```typescript
{ x: number, y: number }
```

式を分解してプロパティに入力し、各プロパティ値式を対応するプロパティ タイプと照合します。`x` については、`7` を `number` と照合します。 `y` については、`"nine"` と `number` を比較します。これらをさらに分解することはできないため、合成とサブタイプに戻ります。`x` の場合、`7` から `number` を合成し、< であることを確認します。 /b9> は `number` のサブタイプです (実際そうです)。 `y` については、`"nine"` から `string` を合成し、`string` が `number` のサブタイプであることを確認します (そうでないため、型エラーが検出されます)。

これにより型チェッカーがより使いやすくなる方法の 1 つは、エラーをローカライズすることです。上記の例では、式全体の型を合成してからサブタイプをチェックすると、次のようなエラー メッセージが表示されます。

```typescript
{ x: 7, y: "nine" }
^^^^^^^^^^^^^^^^^^^

{ x: number, y: string } is not a subtype of { x: number, y: number }
```

しかし、期待される型に対して式をチェックすると、次のようなメッセージが表示されます。

```typescript
{ x: 7, y: "nine" }
           ^^^^^^
string is not a subtype of number
```

型チェッカーは式と型を可能な限り分解し、正確な部分式でエラーを報告するためです。たとえ式と型が大きくても、非常に読みやすく理解しやすいエラーに出来ます。

型チェッカーをより使いやすくするもう 1 つの方法に、必要な型の注釈を減らすことがあります。これの説明はコードに入るまで保留にします。

## Checking + synthesis = bidirectional type checking

この手法(式に対して期待される型がわかっている場合は式を型チェックをし、そうでない場合は式から型を合成する)を双方向型検査と呼びます。葉から根に対して型情報が流れる場合には型合成を行い、根から末端へと流れる場合には型チェックを行います。抽象構文木内で型情報が2方向に対して流れることからこの名前は付けられました。

[^1]: [Rust](https://rustc-dev-guide.rust-lang.org/type-inference.html)でも採用されている
[^2]: [Bidirectional Typing](https://arxiv.org/abs/1908.05839)によると、inferenceではなくsynthesisを使っているのは、program synthesisという用語との整合性を重視しているからのようです。
[^3]:

[] 式が何なのか混乱する方はJavaScript Primerの[文と式](https://jsprimer.net/basic/statement-expression/) を読むことをオススメします。

@babel/typesでは　[conditionalExpression](https://babeljs.io/docs/babel-types#conditionalexpression) と [ifStatement](https://babeljs.io/docs/babel-types#ifstatement)

[^4]:
