---
title: "Scala3 の実験的機能: Capture Checking / Separation Checking 入門"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["scala", "rust"]
published: false
---

このブログは2025年11月27日に開催された[Scalaわいわい勉強会 #6](https://scala-tokyo.connpass.com/event/371493/)で発表した内容をブログに書き起こしたものです。

[Capture Checking / Separation Checking 入門 - Speaker Deck](https://speakerdeck.com/tanishiking/separation-checking-ru-men)

---

大規模なソフトウェアを書くときに厄介なものの一つに、可変なデータの取り扱いが挙げられるでしょう。例えば、意図しない箇所でいつの間にかデータが書き換わっていたり、使うべきではないタイミングでリソースを使ってしまうことなど。

Rustのような言語は所有権(やライフタイム)によりこれらの課題を解決します。が、これをScalaのようなGCを前提とした言語に良い感じ(既存のプログラムに大きな影響を与えずに)に取り込むにはどうすればよいでしょうか? つまり、リソースへのアクセス権限のトラッキングはしたいけど、ライムタイム管理はそんなに頑張らなくていい。

https://x.com/qnighy/status/1976088645905023232

Scala3 の答えが [Capture Checking](https://docs.scala-lang.org/scala3/reference/experimental/cc.html) + [Separation Checking](https://dotty.epfl.ch/docs/reference/experimental/capture-checking/separation-checking.html) と呼ばれる実験的機能だという理解でいます。

## Capture って何?

Capture Checking とはどういう機能かという話をするのですが、そもそも Capture とは何なんですかね?

これは Closure の Capture (日本語だと”捕捉”?)のことで、例えば以下のプログラムでは `increment` というクロージャの内部で、クロージャの外で定義された `c` を参照しています。このとき クロージャ `increment` は `c` を capture しているということになる。

```scala
class Counter:
  def inc: Counter = ???

@main def main():
  val c = new Counter
  val increment = () => {
    c.inc()
  }
  increment()
```

## Capturing Type

Capture Checking では、この変数のcaptureを型レベルでトラッキングできるようにする機能で、そのために "Capturing Types" という型が導入されます。

`T^{x_1, ..., x_n}`

- `T`: Shape Type。この型の値が、次のCapture Setに含まれる値をキャプチャしているということになる
- `{x_1, ... x_n}`: この型の値がキャプチャできる値の集合

![Capturing Types](/images/capturing-types.png)

例えば先の例では、`increment` は以下のような capturing type をもつことになる。[^1]

```scala
val c = new Counter
val increment: (() -> Unit)^{c} = () => {
  c.inc()
}
```

[^1]: 目ざといScalaエンジニアは `() -> Unit` じゃなくて `() => Unit` じゃないの? と思うかもしれません。実は `->` は capture checking 導入にあたって追加された記法で、`=>` は任意の値をキャプチャ可能な関数。`->` は何もキャプチャしない関数を表します。詳しくは[Function Types | Capture Checking](https://docs.scala-lang.org/scala3/reference/experimental/cc.html#function-types)を読んでね。

## Capture Checking

それでは、Capture Checking は何をチェックするのでしょうか?

