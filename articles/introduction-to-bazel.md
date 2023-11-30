---
title: "Bazel入門"
emoji: "🌿"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Bazel", "Scala", "build"]
published: true
---

https://tanishiking24.hatenablog.com/entry/2022/12/14/155923 から

[Scala Advent Calendar 2022](https://qiita.com/advent-calendar/2022/scala) 14日目の記事です。今日は Bazel について書きます。

(ちょっと自動翻訳っぽい日本語ですが、実際そうで、よそで英語で(自分で)書いた文章を日本語に適当に訳して投稿しています、英語の記事はこちら https://virtuslab.com/blog/introduction-to-bazel-for-scala-developers/)

---

プロジェクトのビルド時間は、チームの開発効率に大きな影響を与えます。

しかし、コードベースが大きくなればなるほど、ビルドにかかる時間は長くなり、ビルド時間が長くなればなるほど、開発者のエクスペリエンスは悪化していきます。
Scalaのデファクトスタンダードなビルドシステムであるsbtは優れたビルドツールですが、非常に大規模なプロジェクト(100万行~とか)では長いビルド時間をいい感じにすることは難しい。

![](/images/slow-compiling.png)

この記事では、シンプルなScalaアプリケーションのビルドを通して、Google-scale のリポジトリでも高速なビルドを実現するためのビルドシステム「Bazel」を紹介します。([Bazel の発音は ベィゼル](https://bazel.build/about/faq?hl=en#how_do_you_pronounce_%E2%80%9Cbazel%E2%80%9D))

## What is Bazel
Bazelは`{Fast, Correct} choose two` を謳うビルドシステムで、Artifact-Based Build Systemと呼ばれるものです。

### Artifact-Based Build System とは?
AntやMavenなどの従来のビルドシステムは、Task-Based Build System と呼ばれています。
Task-Based Build System のビルド設定では、`do taskA, then do taskB, and then do taskC` のように procedural なタスクの実行順序を記述する。一方、Buck、Pants、Bazelなどの成果物ベースビルドシステムでは、ビルドする成果物、依存関係のリスト、などを **declarative に記述する**。

**Bazelの基本的な考え方は、ビルドは純粋な関数であり、ソースと依存関係は入力、アーティファクトは出力。そこに副作用はない(よなぁ!?)というものです。**

アーティファクトベースのビルドシステムのコンセプトの詳細については、 [Chapter 18 of Software Engineering at Google](https://abseil.io/resources/swe-book/html/ch18.html) が詳しい。

### `{Fast, Correct} Choose Two`

この主張をよりよく理解するためには、Bazelの [Hermeticity](https://bazel.build/basics/hermeticity) という性質を理解するのと良い。

Bazel は 密閉ビルドシステム (hermetic build system) と呼ばれ、同じ入力ソースと構成が与えられると、同じ出力を返す。Hermeticity のおかげで、Bazelは再現性のあるビルドを提供します。つまり、同じ入力が与えられれば、**誰のコンピュータでも常に同じ出力を返す。** (OS などの platform 情報がビルドに対して暗黙的な依存として含まれるので、誰のコンピュータでもというのは少し語弊があるが...)** (これを correct build という)

また、 correct build のおかげで、Bazelはチーム内でビルドキャッシュを共有するための [リモートキャッシュ機能](https://bazel.build/remote/caching)を提供することができます。リモートキャッシュを使えば、チームメンバー間で共有されたビルドキャッシュを利用して、大規模なプロジェクトを高速にビルドすることができる。

## Bazel Basics
ということで、この記事の残りでは、Bazel を使って簡単な Scala アプリケーションをビルドしつつ、Bazelの重要なコンセプトを紹介していきます。

https://github.com/tanishiking/bazel-tutorial-scala/tree/main/01_scala_tutorial

ディレクトリ構成はこんな感じ

```sh
|-- WORKSPACE
`-- src
    `-- main
        `-- scala
            |-- cmd
            |   |-- BUILD
            |   `-- Runner.scala
            `-- lib
                |-- BUILD
                `-- Greeting.scala
```


Bazel の設定ファイルには `WORKSPACE` と `BUILD` の2つ。

- `WORKSPACE` ファイルは、外部からの情報(3rd party dependenciesなど)をBazelプロジェクトに取り込むための設定を記述するファイル
- `BUILD` ファイルは、Bazel がソースコードをどうビルドするかを記述するファイル

### WORKSPACE ファイル
WORKSPACE` ファイルには、外部の依存関係（Bazel と JVM の両方）などが記述される。例えば、Scala をコンパイルするための Bazel の拡張(build rule)である [rules_scala](https://github.com/bazelbuild/rules_scala) を ダウンロードするときは `WORKSPACE` に以下のような感じで書く。

```python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
 ...
http_archive(
    name = "io_bazel_rules_scala",
    sha256 = "77a3b9308a8780fff3f10cdbbe36d55164b85a48123033f5e970fdae262e8eb2",
    strip_prefix = "rules_scala-20220201",
    type = "zip",
    url = "https://github.com/bazelbuild/rules_scala/releases/download/20220201/rules_scala-20220201.zip",
)
```

(より詳しいインストラクションは [README](https://github.com/bazelbuild/rules_scala) 見てね)


### BUILD file

Bazel でソースコードをどうビルドするかを定義するためには、`BUILD` ファイルを書いていく。。
まずはビルドする Scala ファイルをシュッと見てみましょう。このプロジェクトでは、2つの単純なScalaファイルが異なるパッケージで入っているだけです。

```scala
// src/main/scala/lib/Greeting.scala
package lib
object Greeting {
  def sayHi = println("Hi!")
}
```


```scala
// src/main/scala/cmd/Runner.scala
package cmd
import lib.Greeting
object Runner {
  def main(args: Array[String]) = {
    Greeting.sayHi
  }
}
```

このように、 `lib.Greeting` は `sayHi` メソッドを提供する小さいライブラリモジュールであり、 `cmd.Runner` は `lib.Greeting` に依存。

それでは、これらのScalaソースをビルドするための `BUILD` ファイルの書き方を見てみましょう。


### scala_library

この例では `lib.Greeting` をビルドするために、`BUILD` ファイルを `Greeting.scala` の隣に置き (必ずしも隣に置く必要はないよ、詳しいことは [このブログとかを読んでね](https://medium.com/wix-engineering/migrating-to-bazel-from-maven-or-gradle-part-1-how-to-choose-the-right-build-unit-granularity-a58a8142c549))、以下のような設定を書いてみます。

```python
# src/main/scala/lib/BUILD
load("@io_bazel_rules_scala//scala:scala.bzl", "scala_library")
scala_library(
    name = "greeting",
    srcs = ["Greeting.scala"],
)
```

ここでは `rules_scala` が提供する [scala_library](https://github.com/bazelbuild/rules_scala/blob/master/docs/scala_library.md) というビルドルールを使って、BUILD ファイルを記述しています。

Bazel における[rule](https://bazel.build/rules)とは、コードをビルドしたりテストしたりするための指示のセットを宣言したもの。
例えば、（Bazel がネイティブでサポートしている）[Java プログラムをビルドするためのルール](https://bazel.build/reference/be/java)があったり、`rules_scala` は Scala プログラムをビルドするためのルール群を提供する。例えば[scala_library](https://github.com/bazelbuild/rules_scala/blob/master/docs/scala_library.md) は与えられた Scala のソースをコンパイルして、JAR ファイルを生成する。

`BUILD` ファイルの中身を一行ずつ説明していくと

- `load` 文は BUILD ファイルに `scala_library` ルールをインポートする。
- `scala_library` は Bazel のビルドルール。必須属性は `name` と `srcs` 
  - `name` は この `target` の一意な識別子 (`rule` のインスタンスを `target` と呼ぶ)。
  - srcs` はビルドする Scala ファイルのリスト。


### bazel build
ではビルドファイルもかけたことなので、早速ビルドしましょう。ビルドには `bazel build` コマンドを使う。

```sh
$ bazel build //src/main/scala/lib:greeting
...
INFO: Found 1 target...
Target //src/main/scala/lib:greeting up-to-date:
  bazel-bin/src/main/scala/lib/greeting.jar
INFO: Elapsed time: 0.152s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
```

やったー jar ファイルが生成されました。しかし `//src/main/scala/lib:greeting` って何だ???

### Label

`src/main/scala/lib:greeting` は Bazel では[Label](https://bazel.build/concepts/labels) と呼ばれるもので、 `src/main/scala/lib/BUILD` に定義されている `greeting` という target を指している。Bazelでは、ビルドターゲットを一意に識別するためにLabelを使用します。

Labelは3つの要素から構成される。例えば `@myrepo//my/app/main:app_binary`では、以下のようになります。

- `@myrepo//` は[リポジトリ](https://bazel.build/concepts/build-ref#repositories)名を指定します。`@myrepo` を省略することも可能で、その場合は `//` が同じ作業リポジトリを参照することになる。
- `my/app/main` はパッケージ (`BUILD` ファイルへのプロジェクトルートからの相対パス)を表す。
- `:app_binary` はターゲット名。

つまり、 `/src/main/scala/lib:greeting` は、同じワークスペースにあるターゲットを指しており、 `src/main/scala/lib` にある `BUILD` ファイルで定義されていて、そのターゲット名は `greeting` なビルドターゲットのビルドを実行する。

### Dependency
次に、`lib.Greeting`に依存する`cmd.Runner`をビルドしてみましょう。今回は `cmd.Runner` が `lib.Greeting` に依存しているため、 `deps` 属性を使用してターゲット間の依存関係を導入します。

```python
# src/main/scala/cmd/BUILD
load("@io_bazel_rules_scala//scala:scala.bzl", "scala_binary")
scala_binary(
    name = "runner",
    main_class = "cmd.Runner",
    srcs = ["Runner.scala"],
    deps = ["//src/main/scala/lib:greeting"],
)
```

前の例と違う点は

- `scala_library` の代わりに `scala_binary` を使っている。
  - `scala_binary` は [executable rules](https://bazel.build/extending/rules#executable-rules) と呼ばれるルール。executable rule は、ソースから executable をどのようにビルドするかを定義します。
  - この処理には、依存関係のリンクや、依存関係のクラスパスのリストアップが含まれたりする (一方 scala_library は thin jar を作るだけ)
- 例えば、`scala_binary`ルールはソースと依存関係から実行可能なスクリプトをビルドする。
  - 実行ファイルがビルドされたら、`bazel run` コマンドを用いて実行することができる。
- 依存関係をリストアップするために、`deps` 属性を追加している。
  - この例では、`cmd.Runner`が `lib.Greeting` に依存しているので、 `//src/main/scala/lib:greeting` というラベルを追加しています。

これでビルドできるはず...

```python
$ bazel build //src/main/scala/cmd:runner

ERROR: .../01_scala_tutorial/src/main/scala/cmd/BUILD:3:13:
in scala_binary rule //src/main/scala/cmd:runner:
target '//src/main/scala/lib:greeting' is not visible from
target '//src/main/scala/cmd:runner'.
```

だめでした。`target '//src/main/scala/lib:greeting' is not visible from target '//src/main/scala/cmd:runner'.` らしいです。

### visibility
Bazelには一般的なプログラミングに見られる [visibility](https://bazel.build/concepts/visibility) の概念があります。デフォルトでは、すべてのターゲットの可視性は `private` で、同じパッケージ内のターゲットのみがお互いにアクセスできるようになっています。

`cmd` から `lib:greeting` を見えるようにするには、`greeting` に `visibility` 属性を追加します。

```diff
 scala_library(
     name = "greeting",
     srcs = ["Greeting.scala"],
+    visibility = ["//src/main/scala/cmd:__pkg__"],
 )
```


`//src/main/scala/cmd:__pkg__` はパッケージ `//src/main/scala/cmd` へのアクセスを許可する [Visibility Specification](https://bazel.build/concepts/visibility#visibility-specifications)。

これで...

```sh
$ bazel build //src/main/scala/cmd:runner
...
INFO: Found 1 target...
Target //src/main/scala/cmd:runner up-to-date:
  bazel-bin/src/main/scala/cmd/runner.jar
  bazel-bin/src/main/scala/cmd/runner
INFO: Elapsed time: 0.146s, Critical Path: 0.01s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
```

ビルドできました！

見ての通り、`scala_binary`ルールは `runner.jar` に加えて `runner` も生成します。これは `runner.jar` のラッパースクリプトで、これを使って簡単にJARを実行することができる。

```sh
$ ./bazel-bin/src/main/scala/cmd/runner
# もしくは
$ bazel run bazel build //src/main/scala/cmd:runner
Hi!
```

### Tips
上記の例では、ターゲットのラベルを指定して1つのターゲットをビルドしていますが、面倒くさい。複数のビルドターゲットをまとめてビルドすることはできるのでしょうか?

できる！[ワイルドカードを使って、複数のターゲットを選択することができます](https://bazel.build/run/build#specifying-build-targets)。例えば、`$ bazel build //...` とすれば全てのターゲットをビルドすることができます。便利


ここまででScalaで簡単なアプリケーションを作ることを通じて、Bazelの基本を学びましたが、Bazelで3rd-partyのライブラリを使うにはどうしたらいいのだろう？


## External JVM dependencies

次は外部ライブラリを使ってちょっとだけ難しいアプリを作ってみましょう。

[scalameta](https://github.com/scalameta/scalameta) を使って Scala プログラムを解析し、[pprint](https://github.com/com-lihaoyi/PPrint) を使って AST をきれいに印刷する簡単なアプリケーションを作ってみます。これを通じてMaven からサードパーティーライブラリを使う方法を学んでみましょう。


https://github.com/tanishiking/bazel-tutorial-scala/tree/main/02_scala_maven



この例では、外部のJVM依存性を管理するための標準的なルール・セットの一つである [rules_jvm_external](https://github.com/bazelbuild/rules_jvm_external) を使用します。

(ちなみに: rules_jvm_external を使わなくても maven_jarを使用して、Mavenリポジトリからjarをダウンロードすることができますが、rules_jvm_external は transitive deps の resolve や pinning など[いろんな便利機能](https://github.com/bazelbuild/rules_jvm_external#features) を備えたデファクトスタンダードなので、今回は rules_jvm_external を使うことにします)。


今回の Scala プログラムは1ファイルだけ

```scala
// src/main/scala/example/App.scala
package example
import scala.meta._
object App {
  def main(args: Array[String]) = {
    pprint.pprintln(parse(args.head))
  }
  private def parse(arg: String) = {
    arg.parse[Source].get
  }
}
```

argument で受け取った Scala プログラムを解析して pprint で出力するだけ

Maven リポジトリから `scalameta` と `pprint` をダウンロードするために、 `rules_jvm_external` を使用します。まずは `rules_jvm_external` をダウンロードしましょう。

`rules_jvm_external` をダウンロードするには、[リリースページ](https://github.com/bazelbuild/rules_jvm_external/releases) にある setup statements をコピーして、以下のように `WORKSPACE` ファイルにコピペする。

```python
http_archive(
    name = "rules_jvm_external",
    strip_prefix = "rules_jvm_external-4.5",
    sha256 = "b17d7388feb9bfa7f2fa09031b32707df529f26c91ab9e5d909eb1676badd9a6",
    url = "https://github.com/bazelbuild/rules_jvm_external/archive/refs/tags/4.5.zip",
)
...
```

そして、利用するライブラリ一覧を同じく `WORKSPACE` に書いていく。


```python
load("@rules_jvm_external//:defs.bzl", "maven_install")
maven_install(
    artifacts = [
        "org.scalameta:scalameta_2.13:4.5.13",
        "com.lihaoyi:pprint_2.13:0.7.3",
    ],
    repositories = [
        "https://repo1.maven.org/maven2",
    ],
)
```

こういう感じで依存するライブラリをダウンロードはできるのですが、どうやって使えばいいのだろう?

ダウンロードした依存関係を使用するには、`scala_library` などの `deps` 属性に依存関係を追加する必要があります。`rules_jvm_external` は `@maven` リポジトリ以下のライブラリのターゲットを以下のフォーマットで自動生成します。

> The default label syntax for an artifact `foo.bar:baz-qux:1.2.3` is `@maven//:foo_bar_baz_qux`
> https://github.com/bazelbuild/rules_jvm_external#usage

したがって、`com.lihaoyi:pprint_2.13:0.7.3` を `@maven//:com_lihaoyi_pprint_2_13` というラベルで参照できるようになります。そこで、以下の BUILD ファイルを `App.scala` の隣に書いていく

```python
# src/main/scala/example/BUILD
scala_binary(
    name = "app",
    main_class = "example.App",
    srcs = ["App.scala"],
    deps = [
        "@maven//:com_lihaoyi_pprint_2_13",
        "@maven//:org_scalameta_scalameta_2_13",
    ],
)
```

そしてビルド、実行してみましょう

```
$ bazel build //src/main/scala/example:app
...
INFO: Found 1 target...
Target //src/main/scala/example:app up-to-date:
  bazel-bin/src/main/scala/example/app.jar
  bazel-bin/src/main/scala/example/app
INFO: Elapsed time: 0.165s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
...

$ bazel-bin/src/main/scala/example/app "object main { println(1) }"
Source(
  stats = List(
    Defn.Object(
      ...
    )
  )
)
```

良かったですね。


## まとめ

今回は、Bazel が大規模リポジトリでの高速ビルドを可能にすることを紹介し、簡単な Scala アプリケーションのビルドを通して、Bazel の基本概念と使用方法を紹介しました。この記事がBazelを使い始める最初の一歩になれば嬉しいです。

Bazelは、sbtやMavenなどの他のビルドツールに比べて、多くのビルド設定を管理する必要があることが、今回の小さな例でも見て取れたかなと思います。しかしこれは再現可能なビルドやリモートキャッシュといったBazelの長所によるスケーラブルなビルド速度のためのトレードオフです。

Bazelについてもっと知りたいなら、まずは[公式のガイド](https://bazel.build/start)に目を通して、実際にいくつか小さなアプリケーションをビルドしてみたりすることをおすすめします。
