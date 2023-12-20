---
title: "Kotlin/WASM internal (その1)"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm", "webassembly", "kotlin", "compiler"]
published: false
---

[WebAssembly Advent Calendar 2023](https://qiita.com/advent-calendar/2023/webassembly) 21日目の記事です。(その1)って言ってるけど続くかは不明。

以前 [WasmGCで導入される型や命令のお勉強](https://zenn.dev/tanishiking/articles/learn-wasm-gc) という記事を書いてWasmGC primitivesを学んだので、次はKotlin/Wasmから生成されるWATファイルを眺めて Kotlinのhigh level constructsがWasmGCにどうマッピングされるのかを調べていく(お勉強の過程は[こちら](https://zenn.dev/tanishiking/scraps/b86506b8d23d07))。

今回見てないけど、そのうち調べたいものは

- exception handling
- coroutine
- unsigned xxx
- string の内部表現 (最適化されていてちょっと複雑)

## Kotlin/Wasm から WAT を生成する

https://github.com/Kotlin/kotlin-wasm-examples 

`kotlin-wasm-example/nodejs-example` をいじくって出力を眺めていきます。執筆時のバージョンは[Kotlin 1.9.20](https://kotlinlang.org/docs/whatsnew1920.html)。コンパイラに `-Xwasm-generate-wat` フラグを渡し、WATファイルを生成してくれるようにする [^flags]

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

[^flags]: ちなみに Kotlin コンパイラに渡せる Wasm 関連のコンパイラオプションはこちら https://github.com/JetBrains/kotlin/blob/8a863e00ba148f47d11c825faffd92c494e52ba6/compiler/cli/cli-common/src/org/jetbrains/kotlin/cli/common/arguments/K2JSCompilerArguments.kt#L594-L651

`nodejs-example/src/wasmJsMain/kotlin/Main.kt` を以下のように編集してビルド(main関数はglueコードが多く読みにくいため)

```kotlin
fun main() {
    box()
}
fun box() {
    // ...
}
```

```sh
$ cd nodejs-example
$ ./gradlew build
$ cat ./build/js/packages/kotlin-wasm-nodejs-example-wasm-js/kotlin/kotlin-wasm-nodejs-example-wasm-js.wat
```

## Kotlin/Wasm が生成する WAT を眺める

生成されるWATは3000行くらいあるのだけれど、見やすいように関連するコードフラグメントだけを抜粋する

### direct function calling

まずはいちばん簡単な例から

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

はい

### (data) class

```kotlin
data class Foo(val bar: Int) {
    var baz: Int = 0
}
fun box() {
    val foo = Foo(100)
    foo.baz = 10
}
```

まず `Person` class は以下のように `struct` で表現される。

```wasm
(type $Foo___type_36 (sub $kotlin.Any___type_13 (struct
  (field (ref $Foo.vtable___type_26))
  (field (ref null struct))
  (field (mut i32))
  (field (mut i32))
  (field (mut i32)) ;; bar
  (field (mut i32))))) ;; baz
```

`$Foo___type_36` は `kotlin.Any___type_13` のサブタイプになっていて、`Any` は以下のようなデータ構造。

```wasm
(type $kotlin.Any___type_13 (struct
    (field (ref $kotlin.Any.vtable___type_12)) ;; vtable
    (field (ref null struct)) ;; itable
    (field (mut i32)) ;; typeInfo
    (field (mut i32)))) ;; hashCode
```

![kotlin Any](/images/kotlin-wasm-any.png)

from https://seb.deleuze.fr/introducing-kotlin-wasm/ 

vtable や itable は dynamic dispatch のためのよく知られたデータ構造[^dispatch]。後のvirtual callの項で詳しく述べる。

[^dispatch]: https://lukasatkinson.de/2016/dynamic-vs-static-dispatch/ や https://lukasatkinson.de/2018/interface-dispatch/ を参照

次に `box` の実装や、`Foo` がどう初期化されるのかを見ていく。

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

- `val foo = Foo(100)`
  - `ref.null none` (bottom type を RTT とする null参照) と `i32.const 100` をoperandとして `call $Foo.<init>___fun_61`
- `foo.baz = 10`
  - `Foo` のインスタンスへの参照(`$0_foo`)と `i32.const 10` をoperandにして `struct.set $foo___type_36 5`(`$0_foo` の5番目のフィールド (`baz`) に `i32.const 10` をセット)

`$Foo.<init>___fun_61` は何をするのでしょう?

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

- 引数として受け取った `$0_<this>` が `null` なら
  - vtable, itable, typeinfo, hashcode の初期化 (itable と hashcode は未計算?)
  - `Foo.vtable` は global
- bar と baz に初期値を与えて、Fooへのポインタ `$0_<this>` を返す

vtable や itable がどう作られるか・使われるかは、次の項で紹介する

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

- `$Base___type_36` はいつも通りの vtable, itable, typeinfo, hashCode
- vtable には `Base.foo` への function reference
  - (`data class` じゃないので `hashCode` や `equals` の定義とかはないのね)

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

`foo` が override されているので、`Derived` の vtable には `Derived.foo` への function reference が登録されている。


`box` 関数はこんな感じ。

```wasm
(func $box___fun_65 (type $____type_3)
    (local $0_d (ref null $Derived___type_42))
    ref.null none
    i32.const 1
    call $Derived.<init>___fun_63
    local.tee $0_d  ;; type: <root>.Derived
    call $bar___fun_66)
```

- `Drived.<init>` は上で見たようなコンストラクタ。
- `$bar___fun_66` に Derived instance を与える

```wasm
(type $____type_92 (func (param (ref null $Base___type_36))))
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

- 最初に引数の `Base___type_36` 型の参照を2つ stack に乗せる (上のコードでは `Derived` のインスタンスへの参照を渡すことになる)
  - 1つは、`Base.vtable` から `foo` への function reference を取得するため
  - もう1つは、`Base.foo` に与える receiver
- `struct.get $Base___type_36 0` で引数で受け取った `$0_f: (ref null $Base___type_36)` の vtable を取得 (`Derived.vtable`)
- `struct.get $Base.vtable___type_26 0` で vtable から `foo` への function reference を取得 (`Derived.foo`)
- `call_ref` で関数呼び出し

### interface dispatch
Kotlin(やJavaなど)は単一継承だが、複数のinterfaceを実装している場合はどうだろうか?

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

まずは `box___fun` を見る

```wasm
(func $box___fun_64 (type $____type_3)
    (local $0_d (ref null $Derived___type_40))
    ref.null none
    call $Derived.<init>___fun_61
    local.tee $0_d  ;; type: <root>.Derived
    call $baz___fun_65
    drop)
```

`call $Derived.<init>` して `call $baz___fun_65`を呼び出すだけ。`Derived.<init>` を見ていく。

```wasm
(type $Derived___type_40 (sub $kotlin.Any___type_16 (struct
    (field (ref $Derived.vtable___type_30))
    (field (ref null struct))
    (field (mut i32))
    (field (mut i32)))))

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
```

`global.get Derived.classITable___g_27` で itable をセットしているのが分かる。これはなんだろう

```wasm
;; 実体
(global $Derived.classITable___g_27 (ref $classITable___type_20)
    ref.func $Derived.foo___fun_62
    struct.new $Base1.itable___type_13
    ref.func $Derived.bar___fun_63
    struct.new $Base2.itable___type_14
    struct.new $Base.itable___type_15
    struct.new $classITable___type_20)

;; 型定義
(type $classITable___type_20 (struct
    (field (ref null $Base1.itable___type_13))
    (field (ref null $Base2.itable___type_14))
    (field (ref null $Base.itable___type_15))))
(type $Base1.itable___type_13 (struct (field (ref null $____type_55))))
(type $Base2.itable___type_14 (struct (field (ref null $____type_55))))
(type $Base.itable___type_15 (struct))
(type $____type_55 (func (param (ref null $kotlin.Any___type_16)) (result i32)))

```

- `Derived` の itable には `Base1`, `Base2`, `Base` の itable への参照
- それぞれの itable には、その interface で宣言された関数型のfunction referenceを持つ
  - `Base` では関数定義がないので、空のstruct
- `global $Derived.classITable___g_27` は `Derived.foo` や `Derived.bar` への function reference を itable に登録している。

それじゃあcaller側(`baz`)の実装を見ていく

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

- 最初に引数で受け取った `(param $0_b (ref null $kotlin.Any___type_16))` (実際は `Derived` のインスタンス)をスタックに2つ積む
  - virtual call の例と同様で、1つはitableから関数参照を取得するため、もう1つはreceiver
- `struct.get $kotlin.Any___type_16 1` で `$0_b` の itable を取得
- この型を `Derived` の itable (`$classITable___type_20`) にキャスト
  - 何で vtable と違って itable の型は `(ref null struct)` で実行時にダウンキャストするんだろう?
- `struct.get $classITable___type_20 0` で `Base1` の itable を取得
- `struct.get $Base1.itable___type_13 0` で `Base1.foo` の型の function reference を取得
- `call_ref` で最初にスタックに積んでおいた `$0_b` をreceiverとして`Derived.foo`を呼び出し
- `Derived.bar` は同じなので省略


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



### enum と pattern match

```kotlin
enum class Kind { A, B }
fun box() {
    bar(Kind.A)
}
fun bar(k: Kind) =
    when(k) {
        Kind.A -> 1
        Kind.B -> 2
    }
```

```wasm
(type $Kind___type_44 (sub $kotlin.Enum___type_32 (struct
    (field (ref $Kind.vtable___type_41)) ;; vtable
    (field (ref null struct)) ;; itable
    (field (mut i32)) ;; typeInfo
    (field (mut i32)) ;; hashCode
    (field (mut (ref null $kotlin.String___type_34)))
    (field (mut i32)))))

```

最後の2つのフィールドはなんだろう?これは `Kind.<init>` を見ると分かって、文字列表現と、enumの通し番号

```wasm
(func $Kind.<init>___fun_77 (type $____type_101)
    (param $0_<this> (ref null $Kind___type_44))
    (param $1_name (ref null $kotlin.String___type_34))
    (param $2_ordinal i32) (result (ref null $Kind___type_44))
    
    ;; Object creation prefix
    local.get $0_<this>
    ref.is_null
    if
        
        ;; Any parameters
        global.get $Kind.vtable___g_30
        global.get $Kind.classITable___g_34
        i32.const 580
        i32.const 0
        
        ref.null $kotlin.String___type_34
        i32.const 0
        struct.new $Kind___type_44
        local.set $0_<this>
    end
    
    local.get $0_<this>
    local.get $1_name  ;; type: kotlin.String
    local.get $2_ordinal  ;; type: kotlin.Int
    call $kotlin.Enum.<init>___fun_18
    drop
    local.get $0_<this>
    return)

(func $kotlin.Enum.<init>___fun_18 (type $____type_67)
    (param $0_<this> (ref null $kotlin.Enum___type_32))
    (param $1_name (ref null $kotlin.String___type_34))
    (param $2_ordinal i32) (result (ref null $kotlin.Enum___type_32))
    local.get $0_<this>  ;; type: kotlin.Enum<E of kotlin.Enum>
    local.get $1_name  ;; type: kotlin.String
    struct.set $kotlin.Enum___type_32 4  ;; name: name, type: kotlin.String
    local.get $0_<this>  ;; type: kotlin.Enum<E of kotlin.Enum>
    local.get $2_ordinal  ;; type: kotlin.Int
    struct.set $kotlin.Enum___type_32 5  ;; name: ordinal, type: kotlin.Int
    local.get $0_<this>
    return)
```

```wasm
(func $Kind_initEntries___fun_76 (type $____type_3)
    global.get $Kind_entriesInitialized___g_10  ;; type: kotlin.Boolean
    if
        return
    else
    end
    i32.const 1
    global.set $Kind_entriesInitialized___g_10  ;; type: kotlin.Boolean
    ref.null none
    
    ;; const string: "A"
    i32.const 29
    i32.const 1128
    i32.const 1
    call $kotlin.stringLiteral___fun_25
    
    i32.const 0
    call $Kind.<init>___fun_77
    global.set $Kind_A_instance___g_8  ;; type: <root>.Kind?
    ref.null none
    
    ;; const string: "B"
    i32.const 30
    i32.const 1130
    i32.const 1
    call $kotlin.stringLiteral___fun_25
    
    i32.const 1
    call $Kind.<init>___fun_77
    global.set $Kind_B_instance___g_9  ;; type: <root>.Kind?)
```

```wasm
(func $box___fun_78 (type $____type_3)
    call $Kind_A_getInstance___fun_80
    call $bar___fun_79
    drop)
(func $bar___fun_79 (type $____type_102)
    (param $0_k (ref null $Kind___type_44)) (result i32)
    (local $1_tmp0_subject (ref null $Kind___type_44))
    local.get $0_k  ;; type: <root>.Kind
    local.tee $1_tmp0_subject  ;; type: <root>.Kind
    call $Kind_A_getInstance___fun_80
    local.get $1_tmp0_subject  ;; type: <root>.Kind
    
    ;; virtual call: kotlin.Any.equals
    struct.get $kotlin.Any___type_13 0
    struct.get $kotlin.Any.vtable___type_12 0
    call_ref (type $____type_62)
    
    if (result i32)
        i32.const 1
    else
        local.get $1_tmp0_subject  ;; type: <root>.Kind
        call $Kind_B_getInstance___fun_81
        local.get $1_tmp0_subject  ;; type: <root>.Kind
        
        ;; virtual call: kotlin.Any.equals
        struct.get $kotlin.Any___type_13 0
        struct.get $kotlin.Any.vtable___type_12 0
        call_ref (type $____type_62)
        
        if (result i32)
            i32.const 2
        else
            call $kotlin.wasm.internal.throwNoBranchMatchedException___fun_30
            unreachable
        end
    end
    return)

```

```wasm
(func $Kind_A_getInstance___fun_80 (type $____type_103) (result (ref null $Kind___type_44))
    call $Kind_initEntries___fun_76
    global.get $Kind_A_instance___g_8  ;; type: <root>.Kind?
    return)
```

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