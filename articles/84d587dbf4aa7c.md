---
title: "複数オブジェクトが同じCU DIEを持っててmtimeが同じだと、Darwin linkerは重複したオブジェクトのデバッグ情報を無視"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Scala", "LLVM", "linker", "dwarf"]
published: true
---

https://tanishiking24.hatenablog.com/entry/2023/09/04/142641 からの移植

https://github.com/scala-native/scala-native/issues/3458



Darwin linker はデバッグ情報を最終成果物のバイナリにリンクしない。その代わりに、リンクされたオブジェクトファイルへの絶対パスとタイムスタンプを含む `N_OSO` タイプの symbol を出力バイナリのシンボルテーブルに出力する。

このへん?
https://github.com/tpoechtrager/cctools-port/blob/c1cc75893ed1978174fdcd1b898f81e6535e82d3/cctools/ld64/src/ld/OutputFile.cpp#L5354-L5367


こういうやつ

```sh
❯ cat foo.c
int main() {
  int x = 1;
  return x;
}


❯ clang -g -o foo foo.c

❯ dsymutil --symtab foo | grep N_OSO
[     3] 00000061 66 (N_OSO        ) 00     0001   0000000064f56464 '/var/folders/xt/hyn73bjx4kl6hjj9sxnf1yzh0000gn/T/foo-e04e09.o'
```

デバッガは `N_OSO` シンボルを見て、object file 内のデバッグ情報を参照したり、`dsymutil` コマンドは `N_OSO` のエントリを参照して obect file から デバッグ情報をかき集めて `dSYM` bundle を生成する。 https://github.com/llvm/llvm-project/blob/e2cb07c322e85604dc48f9caec52b3570db0e1d8/llvm/lib/Object/ArchiveWriter.cpp#L698-L707


(ここはちょっと怪しい)


>   // For an object file, the N_OSO entries contain the absolute path
>  // path to the file, and the file's timestamp. For an object
>   // included in an archive, the path is formatted like
>  // "/absolute/path/to/archive.a(member.o)", and the timestamp is the
>  // archive member's timestamp, rather than the archive's timestamp.
>  //
>  // However, this doesn't always uniquely identify an object within
>  // an archive -- an archive file can have multiple entries with the
>  // same filename. (This will happen commonly if the original object
>  // files started in different directories.) The only way they get
>  // distinguished, then, is via the timestamp. But this process is
>  // unable to find the correct object file in the archive when there
>  // are two files of the same name and timestamp.

https://github.com/llvm/llvm-project/blob/e2cb07c322e85604dc48f9caec52b3570db0e1d8/llvm/lib/Object/ArchiveWriter.cpp#L709-L721

archive object 内の object file が同一 path かつタイムスタンプをもつ場合に Darwin linker がオブジェクトファイル(内のデバッグ情報を)区別できないという話。

archive object の話だから関係ないかなと思ったけど、同一 object file の CU DIE が同一 `DW_AT_comp_dir` と `DW_AT_name` と同じ mtime を持っている場合、明らかに重複した object file に対する `N_OSO` シンボルが生成されてない問題を見つけた。

なので、上の LLVM 内の記述は 1 source file - 1 object file でコンパイルすることを前提として書かれているのではないか? object file の path と行っているけど実際はその object file に含まれる CU DIE の属性を参照しているのでは? (推測)


---

ScalaNative 0.4.14 での実装では package ごとに .ll ファイルを生成して、.ll.o にコンパイルする。その結果 1 Scala ファイル内に複数 package があると 複数の `.o` ファイルが作られるし、それらの CU DIE はいずれも同じ Scala ファイルを指す。

```scala
// Test.scala
package A { ... } // A.ll.o whose compilation unit DIE points to Test.scala
package B { ... } // B.ll.o whose compilation unit DIE points to Test.scala
package C { ... } // same
```

これで `Test.scala` から生成される .o ファイルの mtime が同じだと重複したうち一つの .o ファイルしか `N_OSO` が最終的なバイナリのシンボルテーブルに追加されず、`dsymutil` はオブジェクトファイルの検索に失敗する。

そして `warning: (arm64)  could not find object file symbol for symbol __SM33scala.scalanative.issue1359.Main$D1fL16java.lang.ObjectEO` みたいな warning を吐いてくる。

https://github.com/llvm/llvm-project/blob/1ef1eec8db21f35286d9c7fa0a48bb0b6446fe11/llvm/tools/dsymutil/MachODebugMapParser.cpp#L578-L581



https://github.com/scala-native/scala-native/pull/3466

どうなるかな -> ディレクトリごとに分割するようにしました


## 参考



https://dkimitsa.github.io/2018/04/04/fix279-extremely-slow-dsymutil/



https://github.com/llvm/llvm-project/blob/e2cb07c322e85604dc48f9caec52b3570db0e1d8/llvm/lib/Object/ArchiveWriter.cpp#L698-L739

https://github.com/rust-lang/rust/issues/47086

---


archive 内の object file が同一 path かつタイムスタンプをもつ場合に Darwin linker がオブジェクトファイル(内のデバッグ情報を)区別できない問題を回避するため(OSXの?)arコマンドはアーカイブ内の.oファイルのmtimeをわざと1秒ごとずらす?みたいなことやるっぽい
https://github.com/llvm/llvm-project/blob/e2cb07c322e85604dc48f9caec52b3570db0e1d8/llvm/lib/Object/ArchiveWriter.cpp#L698-L739

(けど、その結果link結果がdeterministicじゃなくなって困るみたいな話もあったっぽい (これは ZERO_AR_DATE=1 で回避))

https://github.com/rust-lang/rust/issues/47086
