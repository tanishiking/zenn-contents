---
title: "Wasm Component Model に対するもやもやが晴れてきた"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm"]
published: true
---

## はじめに

先日以下のようなブログを書いた

https://zenn.dev/tanishiking/articles/2024-11-idea-wasm-component

要約すると

- Wasm Component Modelが提供するWITによるリッチなインターフェースを提供すること、それを実現するためのCanonicalABIの策定は良いと思う。
- しかし、コンポーネントを合成したいということのユースケースがよく分からない。
  - ホストを介した他言語関数呼び出しじゃ駄目なの?
- コンポーネントを合成したいという要求のために、既存のモジュールを内包する形でComponentという新しいExecutable and Linkable Formatを作っていて壮大すぎる。もうちょっと小さく進められんのか?
- まだphase1のproposalをベースにWASIp2の仕様策定するマジすか

これをベースにXでごちゃごちゃ言っていたところいろんなコメントを頂き、自分の中でComponent Modelについて腑に落とすことができました(ありがとうございます！！)

頂いたコメントでなるほどな〜と思ったコメントを勝手ながら紹介させてもらいつつ、アップデートしたComponent Modelへの理解をまとめておこうかなと思います。([元ツイートはこちら](https://x.com/tanishiking/status/1858902551485444423))

## Xで頂いたコメント

https://x.com/ruccho_dev/status/1859032070137541001?conversation=none

僕が抱えていたもやもやを的確に言葉にしてくれました。新しくComponentとかいうバイナリ仕様作るみたいなことしないでカスタムセクションでなんとかできれば、既存のWasm仕様から少ない変更で、少なくともIDLやABIの策定はできたんじゃないんだろうか?

https://x.com/chikoski/status/1859038950767424043

そもそも僕はComponent Model proposalっていうのは、(他のproposalと同様?に)既存のWasm言語の課題に対して最小の変更でその問題を解決しようというものだと思っていたので、新しくComponentのバイナリ仕様を持ってくるのはやり過ぎでは?と思っていたのですが

Component Modelは"オープンなエコシステムの構築を目指している"というような態度をとっているような印象が確かにあって、一つの問題解決のためのproposalではなく、もっと理想的にどうなっていて欲しいかを突き詰めたproposalなんだなと気付かされました。

https://x.com/mizchi/status/1859243862000341124?conversation=none

https://x.com/mizchi/status/1859806686145646700?conversation=none

module-host間の呼び出しオーバーヘッドが大きくコンポーネントで静的リンクできると速いという話。module-host間の呼び出しオーバーヘッドが大きいことはScala.jsでも一つの悩みのためではありましたが、ホスト側からモジュールの関数を呼び出すのにもこんなオーバーヘッドあるんですね〜

https://x.com/tamaroning/status/1859084134871953707

https://x.com/tamaroning/status/1859084211904545269?conversation=none

Componentはmemoryをexportできないのですが、確かにそれによってattack surfaceを減らせるというのはでかそう！

と言っていたらComponent Model championであるLuke Wagnerさんの関連しそうなコメントを見つけた

## Luke Wagnerさんのコメント

> Part of what enables all the cross-language/language-agnostic tooling that makes the component model valuable in the first place is that there is no raw shared memory between a component and the outside world: this is what enables writing an interface once (in WIT) and generating bindings in N languages and a bunch of other scenarios as well. With raw public memory, there is no ABI and thus everyone would have to revert to what everyone does in wasm today, which is to roll your own custom thing and produce host-dependent code, which is the state of affairs that the Component Model is seeking to provide an alternative to.
https://github.com/WebAssembly/component-model/issues/275#issuecomment-1828479116

WITを書いてそこから色んな言語のバインディングを生成して、他言語間の呼び出しができるようにするためには、そもそもComponentがmemoryをexportしないことが必要。生のメモリを露出させてしまうとそこにはABIも何もなく(というかABIによる制限を強制することができないって話かな?)今のWasmのようにホスト依存の方法で多言語関数呼び出しをすることになってしまう。とのこと。

確か Choices.md に shared-nothing architecture の話があったな

> The component model adopts a **shared-nothing architecture** in which component instances fully encapsulate their linear memories, tables, globals and, in the future, GC memory. Component interfaces contain only immutable copied values, opaque typed handles and immutable uninstantiated modules/components. While handles and imports can be used as an indirect form of sharing, the dependency use cases enable this degree of sharing to be finely controlled.
https://github.com/WebAssembly/component-model/blob/ee4822aacbce083599f692fc8c8efb08db8d3f3a/design/high-level/Choices.md

そうか、WIT経由で提供した以外でComponentの状態を読み書きできる方法を制限することによって、tamaroningさんが言っていたように悪意のあるcomponentを合成してしまった場合でもlinear memoryを書き換えられることによってプログラムがめちゃくちゃになってしまうのを防いでいるのか。

そう考えるとComponentっていう安全キャップを被せてFFIを提供しようねっていうのはとても納得のいくデザインなような気がしてきた。

## (考えの)まとめ

- 複数言語間でのFFIを安全に提供しようと思ったら、Componentという新しいバイナリ仕様を作ってでもshared-nothing architectureでの合成は納得感のいく方法
- Host-Module間だけの多言語関数呼び出しなら、ホストコードは自分たちのコードのはずなのでModuleがメモリをexportしてても問題?はない
  - Wasm Module1 と Wasm Module2 がホストを介してやり取りしようと思うと、片方のModuleのメモリをもう片方のModuleに渡すことになり、悪意のあるモジュールがメモリをめちゃくちゃにしたりデータを読みとったりする恐れがある。
  - そんな色んな言語から作ったモジュール同士で関数呼び出しするケースあるの? -> Component Modelでそんな世界を作るんだよ！！！
- 別にshared-nothing architectureじゃなくても、既存のWasm Moduleをインクリメンタルに改良してカスタムセクションとか使ってIDLとABIは提供できるはず(だよね?)
  - けどそれは敢えてやってないってこと?何故なら安全じゃないから
- めちゃくちゃビッグドリームやん
- それはそれとしてWASIp1のことも見捨てないでください...
