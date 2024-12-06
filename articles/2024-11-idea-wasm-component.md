---
title: "Wasm Component Model に対するもやもや"
emoji: "👀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm"]
published: true
---

:::message
2024-12-06
Xで意見を募ったところ様々なコメントを頂きました。ありがとうございます。
頂いたコメントを踏まえて新しく文章を書きました: [Wasm Component Model に対するもやもやが晴れてきた](https://zenn.dev/tanishiking/articles/2024-1206-idea-wasm-component)
:::

https://zenn.dev/tanishiking/articles/2024-09-scala-wasm-backend

先日ScalaのWasmバックエンド(JS依存)をリリースし、さて次はこれをJS依存のない**スタンドアローンWasmランタイム(wasmtime, wasmedgeなど)で実行できるようにしよう**と思っている。そのためにはいくつかの標準ライブラリをWASIを利用して再実装してあげる必要がある。

## WASI preview1 と preview2

WASIにはpreview1とpreview2、2つのバージョンがあり：

- **[WASI preview1](https://github.com/WebAssembly/WASI/blob/main/legacy/preview1/docs.md)**
  - 多くのVMでサポートされていて安定している
  - WasmのimportによってWASI関数を利用でき、Wasm moduleに特別な変更を加えずに利用可能
  - インターフェース部分はすべてi32などのWasm1.0の型しか利用できず、データは主にlinear memoryを介してやり取りする

- **[WASI preview2](https://github.com/WebAssembly/WASI/tree/main/wasip2)**
  - [Wasm Component Model proposal](https://github.com/WebAssembly/component-model)をベースにした実装
  - WASIp1にはなかったHTTP関連のインターフェースも含まれている
  - Component Modelで提案されている[WIT](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md)で[インタフェースが定義されている](https://github.com/WebAssembly/WASI/tree/main/wasip2)
    - [wit-bindgen](https://github.com/bytecodealliance/wit-bindgen)などを使ってWITからbindingを生成することで、i32などに限らない言語の型を利用してインタフェースを実装できる
  - 現状[wasmtimeがサポート](https://github.com/bytecodealliance/wasmtime/issues/6370)しているのみ、[wasmedgeは開発中](https://github.com/WasmEdge/WasmEdge/issues/2939)。他のVMは未着手
  - 言語側もRustがつい最近(2024年10月)[1.82.0でWASIp2ターゲットをリリースした](https://github.com/rust-lang/rust/releases/tag/1.82.0) (それ以前はpreview1を使うadapterを噛ませていた)。他言語は未サポート


## WASIp1 か WASIp2 か

さて、どちらを使って実装しようかというところで悩んでいる。選択肢としては以下の2つがある。

- **WASIp1を使って実装し、機を見ていつかWASIp2もサポートする**
  - WASIp1は多くのVMで提供されているので、幅広い環境でScalaのWasmバイナリが利用できるようになる。
    - WasmGCやWasmEHがサポートされれば、WAMR・wazero・chicoryなどのランタイムも将来的に利用可能。
  - ただし、WASIp1ではsocketを開くインターフェースなどが定義されていない。WasmEdgeなんかは独自のWASIp1 extensionでHTTP通信を可能にしている[^socket]。ホスト依存になってしまう。
  - 将来WASIp2がメインストリームになった場合、WASIp1の実装とWASIp2の実装を両方メンテナンスする必要が出てきて二度手間になる可能性がある。

[^socket]: https://go.dev/blog/wasi

- **WASIp1はスキップして、WASIp2のみをサポートする**
  - 将来WASIp2がメインストリームになるのなら、WASIp1の実装の手間が省ける。
  - WITによりスキーマが定義されているため、すべてのWASIp1命令をライブラリ側にハードコードする必要がなく、bindgenを作るだけで良い。(実はWASIp1にもwitxってのがあって〜)
  - しかし現時点でWASIp2をサポートしているのはwasmtimeのみ。wasmedgeは少しずつ進めているが、他のVMがサポートするかどうかは分からない。
    - wasmtimeでしか動かないバイナリは、Wasmのポータビリティという利点を損なうため好ましくない。
  - WASIp2をサポートするということは、Component Modelもサポートする必要がある(と思ってるけど合ってる?)。これは実装が大変なだけでなく、全体的な変更も大きくなる。もし進めた結果うまくいかなかった場合の手戻りは大きい。

[dotnetはWASIp1サポートを切り捨て、WASIp2に賭ける選択をしている](https://github.com/dotnet/runtime/issues/65895#issuecomment-2359352783)。我々はどうするべきだろうか。結局のところ、WASIp2 / Wasm Component Model proposalをどれくらい信用し、コミットするかという判断に帰着すると思う。

正直なところ、WASIp2やWasm Component Modelの嬉しさがよく分かってないので、これがWasmの未来だ！とは思えておらず、どちらかというと便利そうな感じはするけど便利さに対して追加される変更があまりにも大きすぎるような感じを受けている。(安定するのに非常に長い時間がかかるか、コミュニティがあまりついてこないか、形が変わる可能性があるのではと思っている)。そのため、WASIp1を無視してWASIp2に進むことには抵抗感がある。

そもそもWasm Component ModelがまだPhase1(2024年11月時点)なのに、それをベースにWASI preview2をデザインしているのはなんか気が早いなーと思う、デザインするのはいいんだけどWASIp1のことも頑張ってほしい(socket openのインターフェースとか)。日和見的ではあるが、一旦WASIp1サポートから進めるべきかも...?

## Why Wasm Component

Wasm Component Modelにはさまざまな反対意見や疑念が呈されている[^1][^2][^3]のだが、個人的に腑に落ちるような批判意見ではない。WASIp1を選ぶかWASIp2を選ぶかという意思決定をするなら、自分の言葉でどうしてどちらをサポートするかを説明できるようにしたい。そのため、Wasm Component Modelに対する所感を、自分なりにまとめてみようと思う。

[^1]: https://kerkour.com/webassembly-wasi-preview2 (これは正直共感できない。WITがそんなに複雑だとは思わないし、Wasm Componentがネットワークをスコープ外としている時点でCORBAの再来ではない。Rustの影響を受けすぎという指摘も、WITの文法などを見れば些細な話だと感じる。)
[^2]: https://news.ycombinator.com/item?id=32762288 これはWasm Component Modelはこれまでのmoduleとの互換性ないしwasmtimeでしか現状動かないし、portabilityが余計失われるやんけという批判
[^3]: https://github.com/tetratelabs/wazero/issues/2289#issuecomment-2232072988 Component Model が標準化されるのにはかなり時間がかかるだろうとのこと、僕もそう思う。理由は後述

そもそもWasm Component Modelが主張するメリットとは何だろうか？[ドキュメント](https://component-model.bytecodealliance.org/design/why-component-model.html)や[仕様のUseCases](https://github.com/WebAssembly/component-model/blob/main/design/high-level/UseCases.md)を参照すると、以下の利点が挙げられている。

- リッチなインターフェース型による言語間の相互運用性の向上
- コンポーネントの合成による再利用性の向上
- Link-time Virtualization
- (Post-MVP) dynamic linking

### リッチなインターフェース型による言語間の相互運用性の向上

Wasm 2.0では、Wasmが外部とやりとりできるのはi32やf32などの単純な型に限られている。

例えば、文字列やリストをWasmモジュールとやりとりしたい場合、その言語がそれらの値をどのように受け取り表現するのかを理解し、それに基づいてlinear memoryにデータを配置し、そのポインタ(linear memoryのoffsetや長さ)を引数として渡す必要がある。

そのため、Wasmモジュールをプラグインとして利用する際には、各言語向けにプラグインSDKを提供し、エンドユーザーがそのようなシリアライゼーションルーチンを意識しなくても済むようにすることが一般的[^fastly][^proxy][^extism]。

[^fastly]: [Fastly Edge Compute SDK](https://www.fastly.com/documentation/guides/compute/custom/)
[^proxy]: [Proxy Wasm SDK](https://github.com/proxy-wasm/spec)
[^extism]: [Extism Host SDK](https://extism.org/docs/concepts/host-sdk)

Wasm Component Modelでは、[WIT](https://component-model.bytecodealliance.org/design/wit.html)というインターフェース記述言語を使用する。これにより、文字列やリスト、レコードなどの高級なデータ型が利用可能になる。また、[Canonical ABI](https://component-model.bytecodealliance.org/advanced/canonical-abi.html)が、Component Modelで定義された高級な型とCore Wasmでの低レベルな型との間の変換規則(実際にはそれだけではないが)が定義されている。

各言語は、wit-bindgenのようなツールを使い、WITからCanonical ABIを実装したホスト言語向けのソースコードを生成するコードジェネレーターを実装する。

https://github.com/bytecodealliance/wit-bindgen

一度各言語向けにwit-bindgenが実装されれば、従来のようにエンドユーザーが各言語のシリアライゼーションの仕組みを気にする必要はなくなる。SDKを一つひとつ手作りする必要もなくなる。このシリアライゼーションはすべてCanonical ABIを実装したwit-bindgenが担当するため、エンドユーザーは生成されたテンプレートコードに自分の実装を組み込むだけで済む。

良さそうな感じがする。実際Wasm Component Modelに対してよく聞く喜びの声はこの部分に対するものが多い気がする。

### コンポーネントの(静的)合成による再利用性の向上

Wasm Componentは静的に合成することができる。ここでのデータのやり取りはすべてCanonical ABIによって決められているため、任意の言語間で(Component Modelをサポートしている限り)相互に呼び出しが可能になる。

https://component-model.bytecodealliance.org/creating-and-consuming/composing.html

Wasm Component以外での他言語間関数呼び出しにはいくつかの方法がある。

例えば、ホスト言語AからWasmにコンパイルした言語Bを呼び出すことも一種の他言語呼び出しだし、言語CのWasmモジュールのexport関数を言語Bでimportすることだってできる。このやり方の問題点の一つは、上で述べたようにインターフェース定義言語がないため、各Wasmモジュールの高級データ型の表現を把握したうえで他言語関数呼び出しをしなければならない点。[^extism2]

[^extism2]: この問題は、Extismというプラグインシステム向けライブラリを使うことである程度解決できる(ExtismがFFIの煩雑さを引き受けてくれる)。https://dylibso.com/blog/why-extism/

詳しくないのだけれど、`wasm-ld`のようなリンカーを使い、モジュールのインスタンス化前の段階でリンクすることができる。
ただし、[convention](https://github.com/WebAssembly/tool-conventions/blob/main/Linking.md)に従ったモジュールを生成する必要があるらしいがよく分かってない

静的リンクをすることの利点は何？ (多少の不便さがあっても)ホスト部分で多言語関数呼び出しが可能なので、これだけで十分ではないか?

#### 静的リンクのメリット

- **標準ライブラリ等の再利用**
  - ホスト側で多(同)言語関数をimportにわたす方法の場合、それぞれのmoduleはそれぞれ標準ライブラリなどのランタイムコードがすべてlinkされたWasm module。ランタイムや標準ライブラリなどの共通部分を、リンクするmoduleが再利用することができれば、最終的に必要になるWasmモジュール群の総サイズは減らすことができそう

- **ホスト言語にWasmランタイムがない場合**
  - ホスト側で他言語関数をimportに渡す方法は、ホスト言語でWasmランタイムが利用できない場合は使えない(というかWasm実行できないんだからそれはそう)。しかし、Wasmにコンパイルした呼び出し元の言語と呼び出し先のWasmモジュールを事前にリンクしてしまえば、ホスト言語にはWasmランタイムを備えた環境があれば良くなる。
  - 実際にはホスト言語で全くWasmランタイムが使えないという状況は稀だと思う。例えば、WasmEdgeをC-FFIで実行するなどの方法がある。ただし、ネイティブ呼び出しのFFIは全般的に少し扱いが難しい上に、ライブラリやアプリケーションとしての配布も複雑になりがち。
  - ただし、これについてはむしろ、多様な言語がWasmランタイムを持つことで解決される方が良いように思える。 https://github.com/dylibso/chicory など

- **モジュールのインスタンス化部分を自由に変更できない場合**
  - 例えば、いわゆるサーバーサイドWasmのように、特定のWasmモジュールをアップロードして、モジュールのインスタンス化と関数呼び出しをクラウドなどのランタイムが担当するケース。こういった場合、ホスト側で他言語の関数をimportに渡すことができないため、他言語呼び出しをするには事前に静的リンクが必要になる。
  - ただ、このようなサーバーサイドWasmのメリットは、VMに似たセキュリティサンドボックスを提供しながら、V8のisolateのように高速なスタートアップ性能を持つ点にある。[^startup]
  - このユースケースでは、サービスが非常に短命なマイクロサービスである場合が多く、それなら、マイクロサービス間での他言語関数呼び出しは、ネットワークを介してgRPCなどで行えば十分ではないかとも思う。(マイクロサービス間通信より、モジュール内部での関数呼び出しの方が高速という利点は確かにあると思う)

[^startup]: https://notes.crmarsh.com/isolates-microvms-and-webassembly

モジュールのインスタンス化を自由にできないケースとしてサーバーサイドWasm以外に思いつかなかったが、もし他にユースケースがあれば知りたい!

### Link-time virtualization

https://wasmcloud.com/blog/how-to-virtualize-webassembly-components-with-wasi-virt

WASIp1 だと [wasi-vfs](https://github.com/kateinoigakukun/wasi-vfs) がある。このユースケースはRuby.wasmが、RubyスクリプトをインタープリタのあるWasm moduleにvirtual file systemとして埋め込んでやるという話だった気がしていて、なるほどと思った。

それ以外だと...(安全に)ファイル読み書きしたい場合はvirtualizationが欲しくなる? WASIp1だと何が難しくてWasm Component Modelだと何が嬉しいんだろう? ここは結構大事な気がするけどあまりよく読み取れなかった。

### dynamic linking

Wasm Component Model の Post-MVP として(?) Dynamic Linking についても触れられている。

https://github.com/WebAssembly/component-model/blob/main/design/mvp/examples/SharedEverythingDynamicLinking.md

Dynamic Linking のユースケースとして、例えばWasmをWeb上で利用する場合が考えられる。

Wasmファイルはサーバーからネットワークを通してダウンロードされ、ブラウザ上でモジュールの実行が行われる。このとき、Wasmが非常に大きいとWasmファイルのダウンロードに時間がかかり、ユーザーを待たせてしまう可能性がある。Dynamic Linkingが可能であれば、最初にインスタンス化するモジュールと、例えば標準ライブラリのモジュールを別途ダウンロードし、必要になったタイミングでロードすることでスタートアップを高速化できる、という話。

この仕組みはEmscriptenでも実装されている。
https://emscripten.org/docs/compiling/Dynamic-Linking.html

また、Shopify Functionは(多分)サーバーサイドWasmではあるものの、QuickJSランタイムのWasmモジュールをside moduleとして分離することで、以下のような利点を持つ:

- ユーザーがQuickJSを含んだ大きなWasmモジュールをアップロードする必要がない。
- インタープリタのバージョンやビルドをサービス提供側(Shopify)が管理できる。
https://shopify.engineering/javascript-in-webassembly-for-shopify-functions

では、Component Model Proposal は dynamic linking をどのように変えるのだろうか？ Explainer を読んだ限りでは何も分からなかった(誰か教えて〜)

### その他

WASIp1ではHTTP通信に関するインターフェースを定義するのは難しいらしく[^socket-wasi]、開いたsocketに対していろいろするインターフェースは定義されているものの、新しくsocketを開いたりするインターフェースは定義されていない。

WASIp2では[wasi:http](https://github.com/WebAssembly/wasi-http)が定義されている。preview1では何が難しかったのだろうか?確かにhttpのincoming requestのデータ構造を表現するのはWASIp1では難しく、WASIp2なら簡単にデータ構造は表現できたと思うけど。低レベルなsocket APIくらいならできないことはないのでは?と思うんだけど、どういう難しさがあったんでしょうか

[^socket-wasi]: https://www.youtube.com/live/9pLa7PUhPYA?si=rMZW8WGfr9-jLS7v&t=773

## 巨大な proposal

Explainer と CanonicalABI をじっくり読んで[^explainer][^canonicalabi]みて思ったのだが、いろんなやりたいことをまとめた一つの proposal になっている（実際 [interface-types](https://github.com/WebAssembly/interface-types) と [module-linking](https://github.com/WebAssembly/module-linking/) は少なくとも Component Model に融合されている）。

- リッチなインターフェース型を提供したいし、IDL からバインディング生成もしたい
- Wasm モジュールを合成したい
- [Async](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Async.md)
  - CanonicalABI を眺めると、Async のために ABI が複雑になっていることがわかる。

これらを一つの proposal にまとめることで、incremental に話を進めることはできなくなるが、未来（Async など）を見据えてデザインを議論していけるのは良いですね。一方で話が肥大していくので、標準化までの道は鈍じそうだ。

[^explainer]: [rust の Wasm Component Model / WASIp2 の wat を眺める](https://zenn.dev/tanishiking/scraps/7aa5bcbd6902c2)
[^canonicalabi]: [Wasm Component CanonicalABI を眺める](https://zenn.dev/tanishiking/scraps/fb39909cb3f990)

## まとめ

- 個人的に Wasm Component Model のユースケースで良いと思う点は、リッチなインターフェース型を提供してくれるところ。
  - しかしこれだけに対してComponent Modelの変更はあまりにも大きすぎる。
- Component の合成は、いまいち良いユースケースが思い浮かばなかった。
- Dynamic Linking については、今のところ Component Model が何をどう解決するのかよく分かっていない。
- Async についても、これが Component Model の上でデザインされているのは、それが必要だからなのかよく分かっていない。

なんか自分の想定している Wasm のユースケース(いろんな言語でpluginが書ける、サーバーサイドWasmでマイクロサービス)では、Component Model がどう嬉しいのか分からなかった。リッチなインターフェース型とIDLは便利だし欲しいと思うけど、ナイーブな気持ちとしてはこれだけのためにここまで大きな proposal は必要なのか？と思ってしまう。Dynamic Linking / Virtualization / Async にもっと良いユースケースがあるのだろうか? 教えて下さい...
