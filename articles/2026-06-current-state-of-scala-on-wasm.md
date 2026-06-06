---
title: "Scala を Wasm Component として動かす"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["scala", "scalajs", "spin", "webassembly", "kubernetes"]
published: false
---

## この記事のまとめ

- [Scala.js](https://www.scala-js.org/) は Wasm backend を持っているが、現状はJavaScript host を前提
- [scala-wasm](https://github.com/scala-wasm/scala-wasm) では、JS に依存しない Wasm module や [Wasm Component Model](https://component-model.bytecodealliance.org/) への対応を実験しているfork。これにより以下のようなメリットが得られる。
  - sandbox環境での安全なプログラムの実行(AI生成コード・ユーザーにアップロードされたuntrusted code)
  - 異なる言語で書かれた component の相互運用・supply chain attackへの耐性
  - 1ms以下での cold start
- [scala-wasm/component-example](https://github.com/scala-wasm/component-example)にScala+Component Modelの利用例集
- [Spin](https://spinframework.dev/) を使ったサーバーアプリケーションの実装例: [scala-wasm/scala-wasm-spin-templates](https://github.com/scala-wasm/scala-wasm-spin-templates)。
- Limitations
  - JSに依存するlibraryは利用できない。WASIへのportが必要。[cats-effect+FS2のWASI対応進行中](https://summerofcode.withgoogle.com/programs/2026/projects/vibpm0UF)
  - 現状[spinkube](https://www.spinkube.dev/)や[Akamai Functions](https://www.akamai.com/products/akamai-functions)でWasmGC利用不可、[spinkubeは実装中](https://github.com/spinframework/containerd-shim-spin/pull/444)

## Introduction

### Wasm, WASI, and Wasm Component
近年 Wasm は AI が生成したコードを安全に実行するための sandbox や、サーバーサイドでの高速な cold start、言語非依存な plugin 実行環境などの用途で注目されています。

Wasm そのものは純粋な計算のみを行い外部環境への作用を持つことができず、ファイル読み書きや標準入出力/clock/ネットワークなどのシステム機能は host から import した interface 経由で扱います。
そのため、Wasmを実行するホスト環境が明示的に外部への作用を許可しない限り、Wasmは外部への作用を扱うことができません。その意味でWasmはsandboxとして機能すると言われています。

例えば以下の Wasm module 自体は `log` の実装を持たず。たとえば JavaScript host 側で次のように import を渡して初めて、`run` から外部へ作用を実現できます。filesystem や clock、network なども同様に、host が明示的に渡した interface を通じて扱います。

```wat
(module
  (import "env" "log" (func $log (param i32)))
  (func (export "run")
  　　i32.const 42
    call $log))
```

```js
const imports = {
  env: {
    log: value => console.log(value),
  },
};
const { instance } = await WebAssembly.instantiate(bytes, imports);
instance.exports.run(); // 42
```

[WASI](https://wasi.dev/) はそのような system interface を標準化する取り組みで、その現行バージョンであるWASI preview 2 は [Wasm Component Model](https://component-model.bytecodealliance.org/) をベースに定義されています。

Wasm Component Model は、[WIT](https://component-model.bytecodealliance.org/design/wit.html) という interface description language で定義した interface を通じて component の import / export を型付きで扱うための仕組みです。これにより、異なる言語で書かれた component 同士を、言語ごとの ABI や runtime 表現に直接依存せずに組み合わせられます。

```wit
// WIT の例
package example:greeter;
interface greeter {
  greet: func(name: string) -> string;
}
world app {
  import greeter;
}
```

また、Wasm Component は [shared nothing architecture](https://github.com/WebAssembly/component-model/blob/823aa133c2d94a328269da25fd9712bccc142d49/design/high-level/Choices.md) を採用していて、ある component が外部のメモリやデータに直接アクセスすることはできない設計になっています。component 間のやり取りは WIT で定義した interface を通るため、依存 component に悪意のある component が混入した場合でも影響範囲を component boundary の内側に閉じ込めやすく、supply chain attack への耐性を高めやすいという利点があります。

ここではWASIやComponent Modelについて詳しく解説しませんが、詳しく知りたい場合は以下のドキュメントが分かりやすいです。

- [Why the Component Model?](https://component-model.bytecodealliance.org/design/why-component-model.html)
- [Component Model Concepts](https://component-model.bytecodealliance.org/design/component-model-concepts.html)
- [WIT Reference](https://component-model.bytecodealliance.org/design/wit.html)
- [WASI.dev](https://wasi.dev/)

### Scala.js and Wasm

Scala.js は [1.17.0 から Wasm backend をサポート](https://www.scala-js.org/news/2024/09/28/announcing-scalajs-1.17.0/)しており、Scala.js のプログラムを Wasm module にコンパイルできるようになりました。ただし、現状の Wasm backend は Node.js やブラウザなどの JavaScript 環境で動かすことを前提にしていて、wasmtime など JS に依存しない Wasm runtime や Wasm Component Model を直接ターゲットにすることはできません。

そこで、`scala-wasm` という Scala.js の fork で、JS に依存しない Wasm module や Wasm Component のサポートの実験を進めています。

この取り組みは最終的に Scala.js 本体へ upstream することを目指していて、実際にその第一歩として、まずは [Minimal Wasm](https://github.com/scala-js/scala-js/pull/5353) という JS を持たない Wasm module を生成する機能を Scala.js に upstream する実装が進んでいます。

https://github.com/scala-wasm/scala-wasm

この記事では、Scala と Wasm の現状を整理しつつ、`scala-wasm` を使って Scala のコードを Wasm Component にコンパイルする方法について解説します。

## Scala を Wasm Component にコンパイルする

`scala-wasm` を使って、Scala のコードを Wasm Component にコンパイルしてみましょう。

:::message alert
`scala-wasm` は実験的なプロジェクトです。バージョンの更新で API が変更される可能性が高いです。
:::

この記事では Scala+Rustの相互運用成と、[Spin](https://spinframework.dev/) という Wasm のフレームワークを使ってTODOリストサーバーを動かす例を見ていきます。完全なコードや他の例を見たい場合は [scala-wasm/component-example](https://github.com/scala-wasm/component-example) も参考にしてください。

### Requirements

- [wasm-tools](https://github.com/bytecodealliance/wasm-tools)
- [wasmtime](https://github.com/bytecodealliance/wasmtime)
- [scala-wasm/wit-bindgen](https://github.com/scala-wasm/wit-bindgen)
  - scala-wasm 対応版の fork 
- [wkg](https://github.com/bytecodealliance/wasm-pkg-tools)
- [wac](https://github.com/bytecodealliance/wac)
- [cargo-component](https://github.com/bytecodealliance/cargo-component)
- [Spin canary](https://developer.fermyon.com/spin/v4/install)

`scala-wasm` に対応した `wit-bindgen` はまだ `bytecodealliance/wit-bindgen` に upstream されていないため、fork repository からインストールする必要があります

```sh
$ cargo install --git https://github.com/scala-wasm/wit-bindgen --tag scala-wasm-wasm.4
```

### sbt plugin

`scala-wasm` は `io.github.scala-wasm` という organization で配布されています。`project/plugins.sbt` で以下のように plugin を追加します。

```scala
addSbtPlugin("io.github.scala-wasm" % "sbt-scalajs" % "1.21.1-wasm.4")
libraryDependencies += "io.github.scala-wasm" %% "scalajs-env-wasmtime" % "0.0.2"
```

`build.sbt` では以 `scalaJSLinkerConfig` の `moduleKind` を `WasmComponent` に設定します。これにより Wasm Component として link するようになります。また、`scalaJSWitDirectory` 以下に WIT ファイルがあると、compile 時に `wit-bindgen scala` が実行され、`target/scala-*/src_managed` 下に Scala bindings が生成されます。

```scala
import org.scalajs.linker.interface.ModuleKind
enablePlugins(ScalaJSPlugin)

scalaJSWitDirectory := baseDirectory.value / "wit"
scalaJSWitWorld := Some("world-name") // 生成するWIT world名

scalaJSLinkerConfig ~= {
  val witDir = scalaJSWitDirectory.value
  val witWorld = scalaJSWitWorld.value
  _.withExperimentalUseWebAssembly(true)
    .withModuleKind(ModuleKind.WasmComponent)
    .withWasmFeatures { features => // 現状 wasmFeatures にも設定を与えないといけないが、将来的には不要にしたい
      features
        .withWitDirectory(Some(witDir.getAbsolutePath))
        .withWitWorld(witWorld)
    }
}
```

## Wasm Component を使った Scala + Rust の相互呼出

まずは Scala と Rust を Wasm Component を使って相互呼出するためのプロジェクトを作ってみましょう。ここでは Rust が実装した `greeter` interfaceを Scala が呼び出してみましょう

ここではかいつまんだコード例だけを提示しますが、詳細は[component-example/rust-compose](https://github.com/scala-wasm/component-example/tree/b7ebc627447a57b2bc80e7fbcbfff278f3edfa2b/rust-compose) を参照してください。

```wit
package scala-wasm:rust-compose;

interface greeter {
  greet: func(name: string) -> string;
}
world scala {
  export wasi:cli/run@0.2.0;
  import greeter;
}
world rust {
  export greeter;
}
```

まずは `scala` world を実装する Scala の component を実装しましょう。

```scala
import org.scalajs.linker.interface.ModuleKind
lazy val rustComposeScala = project
  .settings(
    // ...
    scalaJSWitDirectory := baseDirectory.value / "wit",
    scalaJSWitWorld := Some("scala"), // scala world を実装
    scalaJSWitPackage := Some("rustcompose"), // Scala bindings のパッケージ名
    Compile / scalaJSLinkerConfig := {
      // ...
      (Compile / scalaJSLinkerConfig).value
        .withExperimentalUseWebAssembly(true)
        .withModuleKind(ModuleKind.WasmComponent)
        // ...
    }
  )
```

コンパイル時にmanaged sourceに生成された binding 経由で `greeter.greet` を呼び出すことができます。

```scala
package rustcompose

import scala.scalajs.wit
import scala.scalajs.wit.annotation._

import rustcompose.exports.wasi.cli.Run
import rustcompose.scala_wasm.rust_compose.greeter.greet

@WitImplementation
object RunImpl extends Run {
  // wasi:cli/run の WIT 定義は `result<unit, unit>`
  override def run(): wit.Result[Unit, Unit] = {
    println(greet("Scala")) // greet 関数を呼び出して結果を出力
    new wit.Ok(())
  }
}
```

Rust 側は同じ WIT から `cargo component` で binding を生成し、`greeter` interface を実装します。ここでは受け取った文字列を [ferris-says](https://github.com/rust-lang/ferris-says) に渡して返ってきた文字列を返してみましょう。

```rust
#[allow(warnings)]
mod bindings;
use crate::bindings::exports::scala_wasm::rust_compose::greeter::Guest;
struct Component;
impl Guest for Component {
    fn greet(name: String) -> String {
        let mut buffer = Vec::new();
        let message = format!("Hello from Rust, {name}!");
        ferris_says::say(&message, 80, &mut buffer).unwrap();
        String::from_utf8(buffer).unwrap()
    }
}
bindings::export!(Component with_types_in bindings);
```

それぞれの component を作成した後、`wac plug` で Scala component の import に Rust component 合成します。

```sh
$ wkg wit fetch
$ sbt rustComposeScala/fastLinkJS
$ cd rust-compose/rust-greeter
$ cargo component build --target wasm32-wasip2 -r
$ cd ..
$ wac plug \
  --plug rust-greeter/target/wasm32-wasip1/release/rust_greeter.wasm \
  scala/target/scala-2.13/rust-compose-scala-fastopt/main.wasm \
  -o out.wasm
$ wasmtime -W gc,function-references,exceptions out.wasm
 _________________________
< Hello from Rust, Scala! >
 -------------------------
        \
         \
            _~^~^~_
        \) /  o o  \ (/
          '_   -   _'
          / '-----' \
```

この例では Scala から見るとただの関数呼び出しですが、実際には WIT で型付けされた component boundary を越えて Rust の実装を呼び出しています。

## Spin で HTTP + SQLite component を動かす

それでは次に、Wasm Component でサーバーアプリケーションを実装してみましょう。

Wasm/WASI は明示的なアクセス許可を与えない限り外部リソースにアクセスできない sandbox 的な性質を持っていて、またさまざまな環境で動作できるため、(Linux container のように user-land のレイヤを介さず) Kubernetes 上で Wasm component を安全に直接実行することができます。これにより、超高速な cold start を狙えるという利点があります。

では、Wasm で実用的なサーバーアプリケーションを実装するにはどうすればよいでしょうか? 外部へのアクセスが自由にできないなら、DB などへのアクセスはどう扱う? 現状の方法のひとつは、[Spin](https://spinframework.dev/) という Wasm framework を使うことでしょう。
ここでは spin を使ったサーバーサイドアプリケーションを Scala で実装してみます。

コードの詳細は [scala-wasm/component-example/spin-todo](https://github.com/scala-wasm/component-example/tree/main/spin-todo) や spin templates [scala-wasm/scala-wasm-spin-templates](https://github.com/scala-wasm/scala-wasm-spin-templates) を参照してください。

また、Scala.js が出力する Wasm Component は Wasm GC / exception handling などの feature を使うため、現時点では Spin canary が必要なことに注意してください。

```sh
$ curl -fsSL https://spinframework.dev/downloads/install.sh | bash -s -- -v canary
```

scala-wasm の spin templates から、プロジェクトを構築します

```sh
$ spin templates install --git https://github.com/scala-wasm/scala-wasm-spin-templates
$ spin new -t http-scala-wasm-todo spin-todo
$ cd spin-todo
```

TODO API の template は、WASI HTTP を export し、Spin の SQLite 向け interface を import する component を生成します。

```wit
package scala-wasm:spin-todo;
world todo {
  import fermyon:spin/sqlite@2.0.0;
  export wasi:http/incoming-handler@0.2.0;
}
```

`wasi:http/incoming-handler` は、WASI HTTP の incoming request を処理するための interface です。これを使ってWasmコンテナのための containerd shim が incoming request を Wasm Component に橋渡しすることができるようになります。

`spin up --build` で build と Spin での起動をしましょう。この際、`--experimental-wasm-features` で GC などの機能を enable する必要があります。

```sh
$ spin up --build \
  --experimental-wasm-feature gc \
  --experimental-wasm-feature exceptions \
  --experimental-wasm-feature function-references \
  --experimental-wasm-feature reference-types \
  --sqlite @migration.sql
```

起動したら HTTP API を呼び出せます🎉

```sh
$ curl -s http://127.0.0.1:3000/todos
$ curl -s http://127.0.0.1:3000/todos \
  -d '{"title":"Try Scala-wasm on Spin"}'
$ curl -s -X DELETE http://127.0.0.1:3000/todos/1
```

Spin で構築したアプリケーションは、通常であれば [SpinKube](https://www.spinkube.dev/) を使って Kubernetes にデプロイしたり、[Akamai Functions](https://www.akamai.com/products/akamai-functions) にデプロイして Function at Edge を実現したりできます。しかし、現時点では `scala-wasm` が生成する使ったバイナリを SpinKube にデプロイすることはできません...Scala.jsが生成するWasmはWasmGCとException Handlingなどの機能を使っているが、それらの環境がGCなどの機能をサポートしていないからです。

spinkube については[containerd-shim-spin#444](https://github.com/spinframework/containerd-shim-spin/pull/444) で対応中。Fermyon Cloud や Akamai Functions は今後の実装に期待ですね。

## FAQ

### XXX library は利用できる?

その library が JavaScript に依存している場合は利用できません。Wasm Component は JS runtime なしで動くことを前提にしているので、`scala.scalajs.js.Dynamic` や `@JSImport`などに依存している code path が残っていると link 時に失敗し次のような linking error が出るはずです。

```text
[error] Uses JS interop with a Wasm-only module kind:
[error]   at file:/.../javalib/src/main/scala/java/lang/_String.scala:416:43: this["repeat"](count)
[error]   at file:/.../javalib/src/main/scala/java/lang/_String.scala:429:29: str["substring"](0, (resultLength -[int] str.length;I()))
[error]   dispatched from java.lang.String.repeat(int)java.lang.String
[error]   called from java.util.regex.PatternSyntaxException.getMessage()java.lang.String
```

特に HTTP library や filesystem / process / socket など system に依存した library は、Wasm 単体では実装できません。WASI HTTP や WASI filesystem など、host から明示的に import した interface を使うように書き換える必要があります。

このような target ごとの実装分岐には `scala.scalajs.LinkingInfo.linkTimeIf` を使います。`linkTimeIf` は link 時に条件が解決され、使われない branch は link 対象から落ちるため、JS host 向けの実装と Wasm-without-JS 向けの実装を切り替えることができます。

```scala
import scala.scalajs.LinkingInfo.{linkTimeIf, moduleKind}
import scala.scalajs.ModuleKind.{MinimalWasmModule, WasmComponent}

def currentTimeMillis(): Long =
  linkTimeIf(moduleKind == WasmComponent) {
    wasiClockCurrentTimeMillis()
  } {
    import scala.scalajs.js
    js.Date.now().toLong
  }
```

実際に `scala-wasm` の javalib でも、`System.currentTimeMillis()` や 正規表現 engine など多くの場所でこの形の分岐を使い、Wasm Component では JS 依存のコードをリンクしないようにしています。また [cats-effectやFS2のWasm/WASI port](https://summerofcode.withgoogle.com/programs/2026/projects/vibpm0UF)も実験的に進行中です。

### Wasm 向けの library artifact を分けないの?

`linkTimeIf`はC/C++やRustのConditional Compilationみたいな機能ですが、代わりにScala.js や Scala Native のように、build 時に `wasm` 用の compile target を作る、という選択肢もあります。そうすると library が `wasm` 向け artifact を publish することになり、依存解決の時点でその library が Wasm-without-JS をサポートしているか分かる、というメリットがあります。

一方で、JS に依存していないライブラリの Scala.js IR は、そのまま `scala-wasm` で使えるのに、すべてのライブラリがWasm-without-JS向けにre-publishする必要が出てくる。などのデメリットもある。

そのため、今のところライブラリartifactは分けることは考えていません。

### いつupstreamされる?

直近の目標は Wasm Component サポートの stepping stone として、[Minimal Wasm](https://github.com/scala-js/scala-js/issues/5333) という機能を Scala.js 本体へ upstream することを目指しています。内部的には JDK API や Wasm backend から JS 依存を取り除くという大きな変更をupstreamしており、これが実現できれば、次は Wasm Component API のサポートを Scala.js 本体へ upstream することになります。
