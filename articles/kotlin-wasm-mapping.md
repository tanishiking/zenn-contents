---
title: "Kotlin/WASM internal"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm", "webassembly", "kotlin", "compiler"]
published: false
---

[WebAssembly Advent Calendar 2023](https://qiita.com/advent-calendar/2023/webassembly) 21日目の記事です。

以前 [WasmGCで導入される型や命令のお勉強](https://zenn.dev/tanishiking/articles/learn-wasm-gc) という記事を書いて WasmGC premitives を学んだので、次は Kotlin/Wasm から生成される WAT ファイルを眺めて Kotlin の high level constructs が WasmGC にどうマッピングされるのかを調べていく(お勉強の過程は[こちら](https://zenn.dev/tanishiking/scraps/b86506b8d23d07))。

## Kotlin/Wasm から WAT を生成する

[Kotlin 1.9.20](https://kotlinlang.org/docs/whatsnew1920.html)

Kotlin/Wasm で遊ぶに当たって、Kotlin organization が提供している `kotlin-wasm-example` リポジトリを使って遊んで見ることにする。

https://github.com/Kotlin/kotlin-wasm-examples 

`nodejs-example` をいじくって出力を眺めてみることにする。(何故なら `nodejs-example` が一番 `build.gradle.kts` が簡単そうだったから)。Kotlin/Wasm が wasm ファイルに加えて wat ファイルも生成してくれるように、コンパイラに `-Xwasm-generate-wat` フラグを渡す。[^flags]

```diff
❯ git diff nodejs-example/build.gradle.kts
diff --git a/nodejs-example/build.gradle.kts b/nodejs-example/build.gradle.kts
index 6b48777..fa21751 100644
--- a/nodejs-example/build.gradle.kts
+++ b/nodejs-example/build.gradle.kts
@@ -26,3 +26,9 @@ rootProject.the<NodeJsRootExtension>().apply {
 tasks.withType<org.jetbrains.kotlin.gradle.targets.js.npm.tasks.KotlinNpmInstallTask>().configureEach {
     args.add("--ignore-engines")
 }
+
+tasks.withType<org.jetbrains.kotlin.gradle.tasks.Kotlin2JsCompile>().configureEach {
+    kotlinOptions.freeCompilerArgs += listOf(
+        "-Xwasm-generate-wat",
+    )
+}
```

[^flags]: ちなみに Kotlin コンパイラに渡せる Wasm 関連のコンパイラオプションはここから眺められる。https://github.com/JetBrains/kotlin/blob/8a863e00ba148f47d11c825faffd92c494e52ba6/compiler/cli/cli-common/src/org/jetbrains/kotlin/cli/common/arguments/K2JSCompilerArguments.kt#L594-L651

`nodejs-example/src/wasmJsMain/kotlin/Main.kt` を以下のように編集して、そのコンパイル結果を


```kotlin
// export される関数(main)から到達してDCEでbox関数が消されないようにする
// (main関数などはglueコードが含まれていて読みにくい)
fun main() {
    box()
}
fun box() {
    // box関数のbodyがどうコンパイルされるかを観察する
}
```

```sh
$ cd nodejs-example
$ ./gradlew build
$ cat ./build/js/packages/kotlin-wasm-nodejs-example-wasm-js/kotlin/kotlin-wasm-nodejs-example-wasm-js.wat
```

## Kotlin/Wasm が生成する WAT を眺める

### Hello World

Kotlin/Wasmの文字列表現は最適化のためにかなり複雑になっているので、ざっくり概要だけ

```kotlin
fun box() {
    println("Hello World")
}
```

```wasm
(type $____type_10 (func (param)))
(func $box___fun_237 (type $____type_10)
    
    ;; const string: "Hello World!"
    i32.const 91
    i32.const 2714
    i32.const 12
    call $kotlin.stringLiteral___fun_141
    call $kotlin.io.println___fun_233)
```

### tail call

```kotlin
tailrec fun gcd(a: Int, b: Int): Int =
    if (b == 0) a
    else gcd(b, a % b)
fun box() {
    gcd(24, 18)
}
```

```wasm
(func $box___fun_62 (type $____type_3)
    i32.const 24
    i32.const 18
    call $gcd___fun_61
    drop)


(type $____type_0 (func (param i32 i32) (result i32)))
func $gcd___fun_61 (type $____type_0)
   (param $0_a i32)
   (param $1_b i32) (result i32)
   (local $2_a i32)
   (local $3_b i32)
   (local $4_tmp0 i32)
   (local $5_tmp1 i32)
   local.get $0_a  ;; type: kotlin.Int
   local.set $2_a
   local.get $1_b  ;; type: kotlin.Int
   local.set $3_b
   loop
       block
           block
               local.get $3_b  ;; type: kotlin.Int
               i32.const 0
               i32.eq
               if (result i32)
                   local.get $2_a  ;; type: kotlin.Int
               else
                   local.get $3_b  ;; type: kotlin.Int
                   local.set $4_tmp0
                   local.get $2_a  ;; type: kotlin.Int
                   local.get $3_b  ;; type: kotlin.Int
                   i32.rem_s
                   local.set $5_tmp1
                   local.get $4_tmp0  ;; type: kotlin.Int
                   local.set $2_a  ;; type: kotlin.Int
                   local.get $5_tmp1  ;; type: kotlin.Int
                   local.set $3_b  ;; type: kotlin.Int
                   br 1
               end
               return
           end
           i32.const 1
           br_if 1
       end
   end
   unreachable)
```


TODO: Array, String, Character, unsigned

### class

```wasm
(func $box___fun_62 (type $____type_3)
    (local $0_foo (ref null $Foo___type_36))
    ref.null none
    i32.const 1
    call $Foo.<init>___fun_61
    local.set $0_foo)

```

```wasm
(type $Foo___type_36 (sub $kotlin.Any___type_13 (struct
    (field (ref $Foo.vtable___type_26))
    (field (ref null struct))
    (field (mut i32))
    (field (mut i32))
    (field (mut i32)))))
```

### function calling (direct)

```kotlin
fun foo(a: Int, b: Int) = a + b
fun box() {
    foo(1, 1)
}
```

```wasm
(type $____type_3 (func (param)))
(func $box___fun_62 (type $____type_3)
    i32.const 1
    i32.const 1
    call $foo___fun_61
    drop)

(type $____type_0 (func (param i32 i32) (result i32)))
(func $foo___fun_61 (type $____type_0)
    (param $0_a i32)
    (param $1_b i32) (result i32)
    local.get $0_a  ;; type: kotlin.Int
    local.get $1_b  ;; type: kotlin.Int
    i32.add
    return)
```

特に何も言うことは無さそう

### default arguments

```kotlin
fun foo(a: Int, b: Int = 0) = a + b
fun box() {
    val x = foo(1)
    foo(1, x)
}
```

```wasm
(type $____type_3 (func (param)))
;; ...
(func $box___fun_64 (type $____type_3)
    (local $0_x i32)
    i32.const 1
    ref.null none
    i32.const 2
    ref.null none
    call $foo$default___fun_63
    local.set $0_x
    i32.const 1
    local.get $0_x  ;; type: kotlin.Int
    call $foo___fun_62
    drop)

(func $foo$default___fun_63 (type $____type_91)
    (param $0_a i32)
    (param $1_b (ref null $kotlin.Int___type_40))
    (param $2_$mask0 i32)
    (param $3_$handler (ref null $kotlin.Any___type_13)) (result i32)
    local.get $2_$mask0  ;; type: kotlin.Int
    i32.const 2
    i32.and
    i32.const 0
    i32.eq
    i32.eqz
    if
        
        ;; Any parameters
        global.get $kotlin.Int.vtable___g_13
        global.get $kotlin.Int.classITable___g_26
        i32.const 480
        i32.const 0
        
        i32.const 0
        struct.new $kotlin.Int___type_40  ;; box
        local.set $1_b  ;; type: kotlin.Int?
    else
    end
    local.get $0_a  ;; type: kotlin.Int
    local.get $1_b  ;; type: kotlin.Int?
    struct.get $kotlin.Int___type_40 4  ;; name: value, type: kotlin.Int
    call $foo___fun_62
    return)

```


### class/data class


```kotlin
data class Foo(val bar: Int) {
    var baz: Int = 0
}
fun box() {
    val foo = Foo(100)
    foo.baz = 10
}
```

まず Person data class は以下のように表現される

```wasm
(type $Foo___type_36 (sub $kotlin.Any___type_13 (struct
  (field (ref $Foo.vtable___type_26))
  (field (ref null struct))
  (field (mut i32))
  (field (mut i32))
  (field (mut i32))
  (field (mut i32)))))
```

`$Foo___type_36` は `kotlin.Any___type_13` のサブタイプ。`Any` は以下のようなデータ構造で表される。

![kotlin Any](/images/kotlin-wasm-any.png "aaa")

field は上から

- vtable
- itable
- typeInfo
- hashCode
- bar
- baz

次に `box` の実装を見ていく。

```wasm
(func $box___fun_62 (type $____type_3)
    (local $0_foo (ref null $Foo___type_36))
    ref.null none
    i32.const 100
    call $Foo.<init>___fun_61
    local.tee $0_foo  ;; type: <root>.Foo
    i32.const 10
    struct.set $Foo___type_36 5  ;; name: baz, type: kotlin.Int)
```

- `i32.const 100` を引数として `call $Foo.<init>___fun_61`
- `Foo` のインスタンスへの参照(`$0_foo`)と `i32.const 10` を引数にして `struct.set $foo___type_36 5`
  - `$0_foo` の5番目のフィールド (`baz`) に `i32.const 10` をセット

`$Foo.<init>___fun_61` は constrcutor、何をする?

```wasm
(func $Foo.<init>___fun_61 (type $____type_88)
    (param $0_<this> (ref null $Foo___type_36))
    (param $1_bar i32) (result (ref null $Foo___type_36))
    
    ;; Object creation prefix
    local.get $0_<this>
    ref.is_null
    if
        
        ;; Any parameters
        global.get $Foo.vtable___g_24
        ref.null struct
        i32.const 452
        i32.const 0
        
        i32.const 0
        i32.const 0
        struct.new $Foo___type_36
        local.set $0_<this>
    end
    
    local.get $0_<this>  ;; type: <root>.Foo
    local.get $1_bar  ;; type: kotlin.Int
    struct.set $Foo___type_36 4  ;; name: bar, type: kotlin.Int
    local.get $0_<this>  ;; type: <root>.Foo
    i32.const 0
    struct.set $Foo___type_36 5  ;; name: baz, type: kotlin.Int
    local.get $0_<this>
    return)
```

- vtable, itable, typeinfo, hashcode の初期化 (itable と hashcode は未計算?)
- bar と baz に初期値を与えて、Fooへのポインタ `$0_<this>` を返す

vtable や itable がどう使われるかは、inheritance などで紹介する


### varargs
```kotlin
fun sum(vararg xs: Int): Int = xs.sum()
fun box() {
    sum(1, 2)
}
```

多分KotlinIRへのLowerする過程でvarargsはArrayに変換されてるっぽい

```wasm
(type $____type_54 (func (param (ref null $kotlin.IntArray___type_31)) (result i32)))
(func $sum___fun_65 (type $____type_54)
    (param $0_xs (ref null $kotlin.IntArray___type_31)) (result i32)
    local.get $0_xs  ;; type: kotlin.IntArray
    call $kotlin.collections.sum___fun_6
    return)

```

```wasm
(func $box___fun_66 (type $____type_3)
    
    ;; Any parameters
    global.get $kotlin.IntArray.vtable___g_12
    ref.null struct
    i32.const 96
    i32.const 0
    
    i32.const 0
    i32.const 2
    array.new_data $kotlin.wasm.internal.WasmIntArray___type_15 1
    struct.new $kotlin.IntArray___type_31
    call $sum___fun_65
    drop)

;; dataidx = 1
(data "\01\00\00\00\02\00\00\00")
(type $kotlin.wasm.internal.WasmIntArray___type_15 (array (mut i32)))
```

- `array.new_data $t $d: [i32, i32] -> [(ref $t)]`
  - `array.new_data $kotlin.wasm.internal.WasmIntArray___type_15 1` は RTT が `$kotlin.wasm.internal.WasmIntArray___type_15` な array を data section の 1つ目の data から作成する
  - operandは、data内のoffsetとsize
  - `i32.const 0` と `i32.const 2` がそれぞれ offset と size
- その上の `global.get` から `i32.const 96` `i32.const 0` は `struct.new $kotlin.IntArray___type_31` の引数


```
(type $kotlin.IntArray___type_31 (sub $kotlin.Any___type_13 (struct
    (field (ref $kotlin.IntArray.vtable___type_21))
    (field (ref null struct))
    (field (mut i32))
    (field (mut i32))
    (field (mut (ref null $kotlin.wasm.internal.WasmIntArray___type_15))))))
```

### generic function

```kotlin
fun <T> id(x: T): T = x
fun box() {
    id(1)
}
```

```wasm
(func $box___fun_63 (type $____type_3)
    
    ;; Any parameters
    global.get $kotlin.Int.vtable___g_13
    global.get $kotlin.Int.classITable___g_26
    i32.const 480
    i32.const 0
    
    i32.const 1
    struct.new $kotlin.Int___type_40  ;; box
    call $id___fun_62
    ref.cast $kotlin.Int___type_40
    struct.get $kotlin.Int___type_40 4  ;; name: value, type: kotlin.Int
    drop)

(func $id___fun_62 (type $____type_56)
    (param $0_x (ref null $kotlin.Any___type_13)) (result (ref null $kotlin.Any___type_13))
    local.get $0_x  ;; type: T of <root>.id
    return)

(type $____type_56 (func (param (ref null $kotlin.Any___type_13)) (result (ref null $kotlin.Any___type_13))))
```

- premitive type ではなく `kotlin.Int` に boxing
- kotlin type の中での type constraints を満たす top 型 (Any) を受け取る関数になる
- 多分この Lowering は早めにやられてそう

### virtual call

```kotlin
open class Base(p: Int) {
    open fun foo() { 1 }
}
class Derived(p: Int) : Base(p) {
    override fun foo() { 2 }
}

fun box() {
    val d = Derived(1)
    bar(d)
}
fun bar(f: Base) = f.foo()
```

Base の型はこちら

- `$Base___type_36` はいつも通りの vtable, itable, typeinfo, hashCode
- vtable には `Base.foo` への参照

```wasm
;; 型定義
(type $Base___type_36 (sub $kotlin.Any___type_13 (struct
    (field (ref $Base.vtable___type_26)) (field (ref null struct)) (field (mut i32)) (field (mut i32)))))
(type $Base.vtable___type_26 (sub $kotlin.Any.vtable___type_12 (struct
    (field (ref null $____type_53)))))
(type $____type_53 (func (param (ref null $kotlin.Any___type_13)) (result i32)))

;; 実体
(global $Base.vtable___g_24 (ref $Base.vtable___type_26)
    ref.func $Base.foo___fun_62
    struct.new $Base.vtable___type_26)
(func $Base.foo___fun_62 (type $____type_53)
    (param $0_<this> (ref null $kotlin.Any___type_13)) (result i32)
    i32.const 1
    return)
```

Derived の定義:

```wasm
;; 型定義
;; super class が Base___type_36、vtable が Derived.vtable なこと以外は Base___type_36 と同じ
(type $Derived___type_42 (sub $Base___type_36 (struct
    (field (ref $Derived.vtable___type_39)) (field (ref null struct)) (field (mut i32)) (field (mut i32)))))
(type $Derived.vtable___type_39 (sub $Base.vtable___type_26 (struct
    (field (ref null $____type_53)))))
(type $____type_53 (func (param (ref null $kotlin.Any___type_13)) (result i32)))

;; 実体
(global $Derived.vtable___g_25 (ref $Derived.vtable___type_39)
    ref.func $Derived.foo___fun_64
    struct.new $Derived.vtable___type_39)
(func $Derived.foo___fun_64 (type $____type_53)
    (param $0_<this> (ref null $kotlin.Any___type_13)) (result i32)
    i32.const 2
    return)
```

`box` 関数はこんな感じ。`Drived.<init>` は先程眺めたような constructor。`$bar___fun_66` に Derived instance を与える

```wasm
(func $box___fun_65 (type $____type_3)
    (local $0_d (ref null $Derived___type_42))
    ref.null none
    i32.const 1
    call $Derived.<init>___fun_63
    local.tee $0_d  ;; type: <root>.Derived
    call $bar___fun_66)
```

```wasm
(func $bar___fun_66 (type $____type_92)
    (param $0_f (ref null $Base___type_36)) (result i32)
    local.get $0_f  ;; type: <root>.Base
    local.get $0_f  ;; type: <root>.Base
    
    ;; virtual call: Base.foo
    struct.get $Base___type_36 0
    struct.get $Base.vtable___type_26 0
    call_ref (type $____type_53)
    
    return)
```

### interface dispatch

```kotlin
interface Base1 {
    fun foo(): Int
}
interface Base2 {
    fun bar(): Int
}
interface Base: Base1, Base2

class Derived: Base {
    override fun foo(): Int = 1
    override fun bar(): Int = 1
}

fun box() {
    val d = Derived()
    baz(d)
}
fun baz(b: Base) = b.foo() + b.bar()
```

```wasm
(func $Derived.<init>___fun_61 (type $____type_92)
    (param $0_<this> (ref null $Derived___type_40)) (result (ref null $Derived___type_40))
    
    ;; Object creation prefix
    local.get $0_<this>
    ref.is_null
    if
        
        ;; Any parameters
        global.get $Derived.vtable___g_24
        global.get $Derived.classITable___g_27
        i32.const 452
        i32.const 0
        
        struct.new $Derived___type_40
        local.set $0_<this>
    end
    
    local.get $0_<this>
    return)


(func $Derived.foo___fun_62 (type $____type_55)
    (param $0_<this> (ref null $kotlin.Any___type_16)) (result i32)
    i32.const 1
    return)
(func $Derived.bar___fun_63 (type $____type_55)
    (param $0_<this> (ref null $kotlin.Any___type_16)) (result i32)
    i32.const 1
    return)

(global $Derived.classITable___g_27 (ref $classITable___type_20)
    ref.func $Derived.foo___fun_62
    struct.new $Base1.itable___type_13
    ref.func $Derived.bar___fun_63
    struct.new $Base2.itable___type_14
    struct.new $Base.itable___type_15
    struct.new $classITable___type_20)

(type $classITable___type_20 (struct
    (field (ref null $Base1.itable___type_13))
    (field (ref null $Base2.itable___type_14))
    (field (ref null $Base.itable___type_15))))
(type $Base1.itable___type_13 (struct (field (ref null $____type_55))))
(type $Base2.itable___type_14 (struct (field (ref null $____type_55))))
(type $Base.itable___type_15 (struct))


```


```wasm
(func $box___fun_64 (type $____type_3)
    (local $0_d (ref null $Derived___type_40))
    ref.null none
    call $Derived.<init>___fun_61
    local.tee $0_d  ;; type: <root>.Derived
    call $baz___fun_65
    drop)
```

```wasm
(func $baz___fun_65 (type $____type_55)
    (param $0_b (ref null $kotlin.Any___type_16)) (result i32)
    local.get $0_b  ;; type: <root>.Base
    local.get $0_b  ;; type: <root>.Base
    
    ;; interface call: Base1.foo
    struct.get $kotlin.Any___type_16 1
    ref.cast $classITable___type_20
    struct.get $classITable___type_20 0
    struct.get $Base1.itable___type_13 0
    call_ref (type $____type_55)
    
    ;; ...
    
    i32.add
    return)
```


### enum と pattern match


### sealed class and when expression



## 感想
- Rustなどの生成する Wasm ではlinear memory上へのallocationや、それらの構造体へのポインタ(`i32`)に型がなかったりして、生成されたWasmコードを読むのが難しかった。WasmGCではallocationは `struct.new` とかするだけでエンジン側が面倒を見てくれるのでコードがとても読みやすくなった。
- コンパイラ側からするとターゲット言語が高級になり、実装は簡単になってそうではあるが、エンジン側の実装難易度は上がりそう。この調子でいろんな仕様が追加されると少しずつportabilityが損なわれていくのではないかという懸念はある。僕はエンジン実装することなかなか無い気がするので知らんけど
  - https://zenn.dev/ri5255/articles/845ef3dab5ab47#wasm%E3%81%AF%E3%81%AA%E3%81%9Cportable%E3%81%AA%E3%81%AE%E3%81%8B%3F
- 



## 参考文献など
- [Introducing Kotlin/Wasm by Zalim Bashorov & Sébastien Deleuze @ Wasm I/O 2023 - YouTube](https://www.youtube.com/watch?v=LCtMC_IVCKo)
  - blog ver: [Introducing Kotlin/Wasm · seb.deleuze.fr](https://seb.deleuze.fr/introducing-kotlin-wasm/)
- [Kotlin Docs | Kotlin Documentation](https://kotlinlang.org/docs/home.html)
- [kotlinlang #webassembly](https://slack-chats.kotlinlang.org/c/webassembly)
- [Kotlin/WASM のお勉強](https://zenn.dev/tanishiking/scraps/b86506b8d23d07)
- [Interface Dispatch | Lukas Atkinson](https://lukasatkinson.de/2018/interface-dispatch/)
- [How does dynamic dispatch work in WebAssembly?](https://fitzgeraldnick.com/2018/04/26/how-does-dynamic-dispatch-work-in-wasm.html)
  - Rust の dynamic dispatch の実現方法。こっちは `call_indirect` を使った実装になっているけど、WasmGC だと struct も typed function reference も導入されてるので、class ごとに vtable や itable を使った方が楽なのかも