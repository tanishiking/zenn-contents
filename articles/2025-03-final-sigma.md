---
title: "toLowerCase と Final Sigma"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unicode"]
published: true
---

訳あって String.toLowerCase を自分で実装しないといけなくなり、ScalaNative や Kotlin/Wasm のコードを読んでいると Final Sigma というものに対する特別扱いを見かけた。なにこれ

どうやら大文字シグマ「Σ (`Σ`, U+03A3, Greek Capital Letter Sigma)」に対してギリシャ語には2つの異なる小文字のシグマがあるらしい。

  - **通常の小文字**: `σ` (U+03C3, Greek Small Letter Sigma) → 単語の途中や単語全体に使われる
  - **語末形（Final Sigma）**: `ς` (U+03C2, Greek Small Letter Final Sigma) → 単語の最後に使われる


# Final Sigma のルール
Unicodeでは `toLowerCase` の処理を行うとき、`Σ` (`U+03A3`) の変換結果は
 
- **単語の途中にある場合** → `σ` (`U+03C3`)
- **単語の最後にある場合** → `ς` (`U+03C2`)

なので例えば

- `ΣΟΦΟΣ` -> `σοφος`
- `ΚΑΛΟΣ` -> `καλος`

らしい。プログラム的には final sigma はどうやって判定するのかな

# Final Sigma の判定

> C is preceded by a sequence consisting of a cased letter and then zero or more case-ignorable characters, and C is not followed by a sequence consisting of zero or more case-ignorable characters and then a cased letter.
> - Before C \p{cased} (\p{Case_Ignorable})*
> - After C ! ( (\p{Case_Ignorable})* \p{cased} )

https://www.unicode.org/versions/Unicode16.0.0/core-spec/chapter-3/#G54277

らしい。つまり

  - Σの前のcharacterが
    - case ignorable letter をスキップしつつ、前に cased letter が見つかる。
    - case ignorable letter とは以下などの文字
      - `'` U+0027 APOSTROPHE
      - `.` U+002E FULL STOP
      - `:` U+003A COLON
    - case ignorable letter には ` `, `\t`, `\n` `EOF` などのホワイトスペースは含まれないので、ホワイトスペースによる単語境界にヒットしたらそこで探索は終わる
  - Σの後ろが
    - case ignorable letter をスキップしつつ、後ろにcased letterが [[ない]]
    - `ΣΟΦΟΣ.` の末尾のΣは単語末尾となるし、先頭のΣは後ろにcased letter `Ο` があるので final sigma ではない。

(ギリシャ語の word boundary って空白文字ってことでOK?)

Case_Ignorable の定義はこちら
https://www.unicode.org/Public/16.0.0/ucd/DerivedCoreProperties.txt

ここでひとつ疑問に思ったのが、前方の探索いる?
例えば `ΜΟΥΣΙΚΗ` (音楽) という単語を lower case にしようとしたときに、Σがfinal sigmaがをチェックするなら後にcased letterの存在をチェックすればいいだけでは?

どうやら後ろだけをチェックしていた場合は `Σ` のように、単語ではない語を final sigmaと誤って判定してしまうのが問題らしい (`Σ`という単語は存在しないらしい！(ほんまか?)日本語だと`い`とか`う`という単語は存在するので意外)。

cased と case ignorable は disjoint ではないことに注意も必要なようだ
> The regular-expression operator * in Table 3-17 is “possessive,” consuming as many characters as possible, with no backup. This is significant in the case of Final_Sigma, because the sets of case-ignorable and cased characters are not disjoint: for example, they both contain U+0345 COMBINING GREEK YPOGEGRAMMENI. Thus, the Before condition is not satisfied if C is preceded by only U+0345, but would be satisfied by the sequence <capital-alpha, ypogegrammeni>. Similarly, the After condition is satisfied if C is only followed by ypogegrammeni, but would not satisfied by the sequence <ypogegrammeni, capital-alpha>.

面白いね
