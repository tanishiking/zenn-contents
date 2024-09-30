---
title: "Scala の Wasm バックエンドを実装した"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm", "webassembly", "scala", "compiler", "javascript"]
published: true
---

Scala.js 1.17.0 で実験的な Wasm backend がサポートされました！

https://www.scala-js.org/news/2024/09/28/announcing-scalajs-1.17.0/

リリースノートに書いてあるとおり、以下のような設定をすることでScala.jsがJSの代わりにWasmモジュールを生成することができます。

`@JSExport` によるモジュールのexportがサポートされていませんが、それ以外のsemanticsはサポートされており、既存のScala.jsアプリケーションを変更なしにWasmにビルドすることが可能なはずです。(もし何か問題があれば教えて下さい!)

```scala
// Emit ES modules with the Wasm backend
scalaJSLinkerConfig := {
  scalaJSLinkerConfig.value
    .withExperimentalUseWebAssembly(true) // use the Wasm backend
    .withModuleKind(ModuleKind.ESModule)  // required by the Wasm backend
    .withPrettyPrint(true) // (デバッグ用) これを設定すると .wat ファイルも一緒に生成してくれる
},

// Configure Node.js (at least v22) to support the required Wasm features
jsEnv := {
  val config = NodeJSEnv.Config()
    .withArgs(List(
      "--experimental-wasm-exnref", // required
      "--experimental-wasm-imported-strings", // optional (good for performance)
      "--turboshaft-wasm", // optional, but significantly increases stability
    ))
  new NodeJSEnv(config)
},
```

以下の Wasm features を利用した Wasm module を生成しているため、新し目のブラウザ・JSランタイムが必要になります(Node.js v22 など)

- Wasm Garbage Collection
- Wasm Exception Handling
- (optional) JS String Builtin
  - 利用できない場合はpollyfillによる実装にfallbackします
- Tail Call Extension

各種ブラウザ・JSランタイムのサポート状況は https://webassembly.org/features/

またJS環境で実行可能なWasmしか生成することができず、wasmtimeやWasmEdgeといったWasmランタイムはサポートしていません。

## パフォーマンス・バイナリサイズ

Wasmは早いとか言われますが、実際はWasmは最適化する余地がJSより大きいという理解で、実際現時点ではWasmバックエンドは必ずしもScala.jsのJSバックエンドより実行速度が早いコードを生成するとは限りません。

また生成されるコードのサイズも、現時点では Scala.js の JS backend + Google Closure Compiler を使って最適化したものの方が、Wasm backend よりも小さいコードを生成します。

少し前のデータになりますが、以下のブログ(英語)にベンチマーク結果を書いていますので詳しく知りたい方はそちらを参照してください。

https://dev.virtuslab.com/i/146705467/run-time-performance-analysis

また以下のツイートではJSバックエンドとWasmバックエンドで生成したLife of Gameの実行速度を比較していますが、やはりJSの方が幾分早いですね。
単純な数値計算なんかはWasmが長じているのですが、JS interopが多く発生するとそのオーバーヘッドがパフォーマンスに大きく影響してくるように感じています。

https://x.com/velvetbaldmime/status/1840009315094016489

これを見てWasm遅いじゃん！と思ってほしくはなくてScala.jsは10年以上の最適化の積み重ねにより効率的なJSコードを生成しているのに対して、Wasmバックエンドは開発開始からまだ約半年。まだまだ最適化の余地が多く残されています。

例えばまだ[wasm-opt](https://github.com/WebAssembly/binaryen)は[一部のWasmGCの機能が不足している](https://github.com/WebAssembly/binaryen/issues/6407)ため、Scala.jsが生成したWasmバイナリに対して最適化を実行することができません。(block parameter typeを`local.get`と`local.set`にlowerするパッチは手元にあるのですが...)

今後の最適化でどのくらい高速にバイナリサイズを小さく出来るか楽しみですね。

## 今後

- さらなる最適化
- JS依存のないWasmモジュールの生成。これによりwasmtimeやwasmedgeなどのstand-alone wasm runtimeで実行できるコードを生成
  - https://github.com/scala-js/scala-js/issues/4991
- Wasm Component Model (host/guest) 対応
  - WASI Preview 2 対応のためにどちらにしろhostとしての対応は必要になる
  - GCをmoduleに組み込まず、WasmGCを利用する選択をしたからこそサイズの小さいguest componentが作れるのではないか?
- [stack swtiching proposal](https://github.com/WebAssembly/stack-switching)による効率的なVirtual Threadの実装
  - ScalaJVMはProject Loom、[ScalaNativeはsetjmp/longjmpベースのdelimcc](https://dl.acm.org/doi/abs/10.1145/3679005.3685979)によりVirtual Thread APIをサポートしていますが、ScalaJSではwhole program transformationをするしか術がありませんでした。stack switching proposal により ScalaJS でも効率的な Virtual Thread API の実装が可能になるのではないかと期待しています(まだ全然調べてない)


## 感想

2024年初め頃にScala.jsのWasmバックエンド実装を始めて、やっとこさアップストリームへのリリースまでこぎつけることができました。最初のベース実装は僕がやったのですが、そこから実際にあらゆるソフトウェアを動かすことができるようにするまではScala.js作者の[@sjrd](https://github.com/sjrd)がかなり実装してくれました(ありがとう)。

https://github.com/tanishiking/scala-wasm

https://github.com/scala-js/scala-js/pull/4988

引き続き、ScalaのWasm対応頑張っていこうと思います。
