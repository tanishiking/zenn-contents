---
title: "Scala.js + Javy で Scala を WebAssembly 上で動かす"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Scala", "webassembly", "javascript", "scalajs", "wasm"]
published: true
---

[Scala Advent Calendar 2023](https://qiita.com/advent-calendar/2023/scala) 1日目の記事です。

最近はScalaのWebAssembly対応やりたいな〜と思ってWebAssemblyの勉強をしています。

## (前置き) Scala の WebAssembly 対応 (2023)

ScalaからWebAssemblyを生成する方法はいくつかあるのですが、[Kotlin/WASM](https://kotlinlang.org/docs/wasm-overview.html) のようにScalaコンパイラがWebAssemblyを直接生成ことは今のところできません。

いくつかの方法というのは例えば

### TeaVM
https://teavm.org/

TeaVMはJVMバイトコードをJSやWASMにAOTコンパイルしてくれるツールです。sbtからも使える🎉

https://xuwei-k.hatenablog.com/entry/2023/11/08/100944
WebAssembly 対応は experimental とのことで、以前試したときは確かにいろいろ動かなかった気がする

### Scala Native
https://scala-native.org/en/stable/

- Scala Native は LLVM をバックエンドにしたコンパイラ
- LLVM 吐き出せるなら [Emscripten](https://emscripten.org/index.html) や [WASI-SDK](https://github.com/WebAssembly/wasi-sdk) で WebAssembly 吐けるやん
  - 実験会場 https://github.com/WojciechMazur/scala-native/tree/scala-wasm
- WASI-SDK で exception handling 今のところなし [RFC: Add Wasm exception support by whitequark · Pull Request #198 · WebAssembly/wasi-sdk](https://github.com/WebAssembly/wasi-sdk/pull/198)
- LLVM から Emscripten や WASI-SDK 使って wasm gc primitive を生成する方法今のところ無さそう?

### 新しいバックエンドが必要

以上の方法では実用的なWASMを生成することができません。WASMをちゃんとサポートするなら、ScalaコンパイラがWASMを吐き出す新しいバックエンドを作るのが良さそうというのが[Scalaコンパイラ開発チームの今のところの(うちうちの)見解となっています](https://github.com/scala-native/scala-native/issues/603#issuecomment-1819660009)。

とはいえ仮にScalaがwasm gcやWASIバックエンドを実装したとしても、wasmtime・wasmedge・wasmer(?)はまだgc[^gc]もexception handling[^eh]も実装されておらず、browser-embedding はまだしも、WASI対応はまだまだ遠い話になりそう...[^web]

もしくは AssemblyScript なんかみたいに自前でGCを実装するか、だけどなんかwasmgcが上記のruntimeで使えるようになったらいらなくなるだろうしな〜どうしよう

[^eh]: https://github.com/bytecodealliance/wasmtime/issues/3427 https://github.com/wasmerio/wasmer/issues/3100 LLVMベース言語仲間のCrystalさんも困ってます https://github.com/crystal-lang/crystal/issues/13130

[^gc]: https://github.com/bytecodealliance/rfcs/pull/31 https://github.com/WasmEdge/WasmEdge/issues/1122

[^web]: (ところでGC言語で WASM for browser-embedding って、[Blazer](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor)や[Flutter on the Web](https://flutter.dev/multi-platform/web) みたいなその言語だけでWebアプリケーション作るぜ！ってplatform以外だと嬉しいことあるのかな...?)

## Javy

そうしたら[Javy](https://github.com/bytecodealliance/javy)というJSをWASM上で実行するツールをShopifyが開発しているという記事を見つけました。

https://shopify.engineering/javascript-in-webassembly-for-shopify-functions

詳しくは上の記事を見ると良いのですが、JavyはJavaScriptをWebAssembly（厳密には `wasm32-wasi`）上で動かすためのツールチェーンです。

Javy は、JSをWASMにコンパイルするわけではなく、WASMにコンパイルされたCで書かれた小さくて埋め込み可能なJavascriptエンジンである `QuickJS` を使用して、WASM上で(`QuickJS`バイトコードにコンパイルされた)JSを埋め込まれたQuickJSエンジンを使った実行する形です。

そのため、例外処理やPromiseのような高レベルの機能も問題なく動作するし、`QuickJS`のGCのおかげでメモリを食いつぶすこともない。

## Scala.js + Javy

ところで、ScalaにはScala.jsという恐ろしく完成度の高いScala->JSコンパイラバックエンドがありまして...

https://blog.3qe.us/entry/2023/10/02/221036

じゃあ `Scala->(scala.js)->JS->(Javy)->WASI` で Scalaで書いたコードがだいたいWASM上で動かせるんじゃない...? ということでやってみました。

https://github.com/tanishiking/scala-js-javy-playground

### Exception Handling

例外が動くか見るために以下のような簡単なコードをコンパイルしてみよう

```scala
// Hello.scala
import scala.scalajs.js
import java.lang.Throwable

object Hello:
  def main(args: Array[String]): Unit =
    val console = js.Dynamic.global.console
    try
      throw new Error("test")
    catch
      case e: Throwable => console.log(e.getMessage)
```

これを [scala-cli](https://scala-cli.virtuslab.org/)を使ってJSにコンパイルし、JavyでWASMにコンパイルし、[wasmtime](https://wasmtime.dev/)で実行してみる

```console
$ scala-cli package --js Hello.scala -o build/hello.js --force

$ javy compile build/hello.js -o destination/hello.wasm

$ wasmtime destination/hello.wasm
java.lang.Error: test
```

できました🎉

### Future / Promise

Scalaの`Future`はJS上では[event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop)を使って実装されています。(参考: [JavaScriptでScalaのFutureを表現する](https://zenn.dev/mox692/articles/4d1df32f508f00))

しかし、Javyは(現時点では)デフォルトではQuickJSのevent loopを有効化していません。

> we haven’t enabled the event loop in the QuickJS instance that Javy uses. That means that async/await, Promises, and functions like setTimeout are syntactically available but never trigger their callbacks. We want to enable this functionality in Javy, but have to clear up a couple of open questions, like if and how to integrate the event loop with the host system or how to implement setTimeout() from inside a WASI environment.
> https://shopify.engineering/javascript-in-webassembly-for-shopify-functions

なので、以下のようなコードを書いて

```scala
// Promise.scala
import scala.concurrent._
import scala.util.Success
import concurrent.ExecutionContext.Implicits.global

object Promise:
  def fetchData(): Future[String] = Future { "some data!" }
  def main(args: Array[String]): Unit =
    val f = fetchData()
    f.onComplete:
      case Success(data) => println(data)
```

WASMにビルドして普通にJavyで実行しようとすると失敗してしまう。

```console
$ scala-cli package --js Promise.scala -o build/promise.js --force
$ javy compile build/promise.js -o destination/promise.wasm
$ wasmtime destination/promise.wasm
Error while running JS: Adding tasks to the event queue is not supported
Error: failed to run main module `destination/promise.wasm`

Caused by:
    0: failed to invoke command default
    1: error while executing at wasm backtrace:
           0: 0x5cfa1 - <unknown>!<wasm function 104>
           1: 0x6f59c - <unknown>!<wasm function 165>
           2: 0xb6630 - <unknown>!<wasm function 1005>
    2: wasm trap: wasm `unreachable` instruction executed
```

### experimental_event_loop

どうやらEvent Loopは `experimental_event_loop` フラグで有効化できるようなので、試してみる。(Javyをそのフラグ付きでビルドする)


```console
$ cargo build --features experimental_event_loop -p javy-core --target=wasm32-wasi -r
$ cargo install --path crates/cli

$ javy compile build/promise.js -o destination/promise.wasm
$ wasmtime destination/promise.wasm
some data!
```

動きました🎉 (どんなリスクがあるのか分かってないけど！)

## 他のプラットフォームでの活用

例えば

- WASMベースのmicroserviceを実装するためのフレームワーク[spin](https://www.fermyon.com/spin)の、[spin-js-sdk](https://github.com/fermyon/spin-js-sdk)もJavyを使って実装されています。
- またWASMによるuniversal plugin systemを実装する[Extism](https://extism.org/)の、[js plugin development kit](https://github.com/extism/js-pdk)もJavyを使って実装されています。

Scala.jsでは[ScalablyTyped](https://scalablytyped.org/docs/readme.html)というツールを使うことで簡単にTSライブラリのScalaバインディングが生成可能なので、同じ要領でScalaでWASMベースのmicroserviceやpluginを実装することができます。

### Scala on Fermyon Cloud

実際にspin-js-sdk、scala.js、ScalablyTypedを使って、Scalaで書いたコードをWASM上で動くマイクロサービスにしてみました。

https://github.com/tanishiking/spin-scalajs-example

ここでは詳しく書かず、また別の記事で書くことにします。

## Cons

これでScalaをWASM上で動かすことができました！

しかし、デメリットもあります。というのも残念ながら、KotlinやDart、そしてもちろんC++やRustによるWASMと比べると、実行速度は劣るのではないか(ちゃんと計測してない)。その理由は

- (1) 一つは、WASMモジュールに組み込まれたJSエンジン上でコンパイルしたJSコードを実行するだけだからです。WASMを直接吐き出してそれを実行するほうが早そうに思える
  - QuickJSをサイドモジュールとして動的リンクすることで、WASM moduleのサイズを小さくすることもできますが、static linkするとそのぶんmodule sizeも大きくなる
- (2) もう1つの理由は、QuickJSは小さく組み込み可能である代わりに、V8やSpiderMonkeyのような他の大規模JSエンジンよりも遅いからです。
  - 実際、QuickJSの公式ベンチマークによると、JITが有効なV8はQuickJSの最大30倍速くなるそうです。https://bellard.org/quickjs/bench.html。

## Pros

しかし、良い部分もあります。WASM化はすべて実行パフォーマンスのためと考えると、Javy+Scala.jsでScalaをWASM上で動かすメリットはあまり感じられないかもしれません。

しかし、下記ブログで述べられているように、WASMの利点は実行速度だけでなく、起動パフォーマンスが高いこと、セキュリティ、ポータビリティなどもあります。

https://blog.anatoo.jp/2023-01-18


特に、Scala.jsとJavyを使ってScalaをコンパイルするメリットは、`wasm32-unkown-unknown`ではなく、GCや例外機構などの高水準な機能を備えた`wasm32-wasi`にコンパイルできることだと思います。これにより、ScalaをWASMベースのプラットフォームで利用できるようになる


- Shopify Functions・Extism・dprintのなどのためのをScalaで記述できる
- また、Scalaコードをfermyon cloud や wasm worker server などの wasm ベースのマイクロサービスプラットフォームにデプロイできる
- edge computingプラットフォーム上でScalaを実行する（ただし、エッジに置けるようにバイナリサイズを縮小する必要があるが...）
- [Near](https://near.org/)などのWebAssemblyを実行するWeb3プラットフォームでScalaを利用できる

## ScalaとWASMの今後

(個人の意見、今開発チームとお話している最中です)
Scala.js + Javy でScalaをWASM上で実行できるようになった。[^wasi_shim]

[^wasi_shim]: ビルドターゲットはwasm32-wasiだが、ブラウザで実行したければ(そんなことある?)[browser_wasi_shim](https://github.com/bjorn3/browser_wasi_shim)を使えばよいだろう。

じゃあ、もうScalaのWASMサポートはこれでおしまい？かというと当然そんなことはない。[Kotlin/WASMがそうしたように](https://www.publickey1.jp/blog/23/kotlin_190webassembly101k2.html)自前のWASMバックエンドを実装することで生成されるWASMバイナリサイズを大きく削減することができるだろう。またScalaの[wit-bindgen](https://github.com/bytecodealliance/wit-bindgen)も必要になってくるだろう。

WASMバックエンドは、WASM GCに乗っかるべきだろうか?それともAssemblyScriptなどのように自分たちでlinear memoryに対するGC機構を実装するべきだろうか? - 個人的には WASM GCに乗っかっていくと良さそうな気がしている。

最近は多くのプログラミング言語(KotlinやOCamlやbinaryen IR)がWASM GC primitiveを実装し、また[V8](https://v8.dev/blog/wasm-gc-porting)などもWASM GCをサポートしている。長い目で見れば、wasmtimeやwasmedgeのようなランタイムはいずれwasm gcをサポートするでしょう...(知らんけど)[^wasmgc]

[^wasmgc]: wasmtime 上で WASM GC を実装するための RFC はすでにマージされています https://github.com/bytecodealliance/rfcs/pull/31 またwasmedgeもwasm gcの実装に励んでいるらしい https://github.com/WasmEdge/WasmEdge/issues/1122

それらのプラットフォームがwasm gcを実装してくれれば、我々がGCを自前で実装する必要もなくなるだろう。

当面の間は今回紹介した方法でWASMにコンパイルしつつ、(WASMサポートを少しでも簡単にするためにも)自分たちでGCを実装するのではなくWASM GCにコンパイルすることに集中するべきかもしれない
