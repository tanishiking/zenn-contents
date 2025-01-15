---
title: "CanonicalABI lift/lower チートシート"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm"]
published: true
---

[CanonicalABI](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md)はWITで定義されるようなリッチなインターフェース型とそれに対応するCore Wasmでの表現に加えて、Component間での関数呼び出し規則なども定義している。

コンパイラがComponentModel対応する際は主に前者を気にしてwit-bindgenなりを実装していくことになる。しかしWITの型/値とCore Wasmの型/値の対応もそこそこ複雑で忘れてしまいがちなので備忘録的にチートシートを作っておく。WASIp3に関連する`Future`, `Stream`, `ErrorContext`型は今回は無視する。

自分のメモとしてスクラップ https://zenn.dev/tanishiking/scraps/fb39909cb3f990 をまとめたものなので、他の人が読んでもわかり易くないかも知れないけど`CanonicalABI.md`の副読本として使うと便利かもしれない

## Specialized value types

`tuple`, `flags`, `enum`, `options`, `result`, `string` は[specialized value types](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md#specialized-value-types)という、他の型によって表現される特殊型として定義される

```
                    (tuple <valtype>*) ↦ (record (field "𝒊" <valtype>)*) for 𝒊=0,1,...
                    (flags "<label>"*) ↦ (record (field "<label>" bool)*)
                     (enum "<label>"+) ↦ (variant (case "<label>")+)
                    (option <valtype>) ↦ (variant (case "none") (case "some" <valtype>))
(result <valtype>? (error <valtype>)?) ↦ (variant (case "ok" <valtype>?) (case "error" <valtype>?))
                                string ↦ (list char)
```

flags/stringはrecord/listと異なるcore wasm表現を持つので例外(じゃあ何でspecialized typesとして定義してみたんだ?)だが、それ以外の特殊型はrecordとvariantと同様のlift/lowerの挙動が適用される。

## flattening

[flattening](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#flattening)の項に書いているように、基本的にはCanonicalABIではComponent間のデータのやり取りは線形メモリ上にどのようにデータをstore/loadするかという体で定義されている。しかし、関数の引数や返り値の数が少ない場合はパフォーマンス向上の為、通常のcore wasm関数と同様にスタックを通した値のやり取りをすることができるようになる(flattening)。

そのため、component関数シグネチャは、core wasmでは以下のようなシグネチャとなる


```python
MAX_FLAT_PARAMS = 16
MAX_FLAT_RESULTS = 1

def flatten_functype(opts, ft, context):
  flat_params = flatten_types(ft.param_types())
  flat_results = flatten_types(ft.result_types())
  if opts.sync:
    # 引数が多すぎる場合は、全ての引数は線形メモリを介して渡される。
    # stackには代わりにそれらの引数がstoreされているメモリ開始位置
    if len(flat_params) > MAX_FLAT_PARAMS:
      flat_params = ['i32']
    if len(flat_results) > MAX_FLAT_RESULTS:
      # resultの数が2つ以上あるケース
      # 返り値はメモリを通して返される
      match context:
        case 'lift': # 関数をexportするケース
          # 返り値は線形メモリに配置される。resultのスタックにはそのメモリのoffset
          flat_results = ['i32']
        case 'lower': # 他のcomponentから関数をimportするケース
          # パラメータにポインタ(線形メモリ上のoffset)を追加
          # calleeはこのoffset位置に返り値を配置していく。
          # callerは関数呼び出しあとにこのoffsetから値をload
          flat_params += ['i32']
          flat_results = []
    return CoreFuncType(flat_params, flat_results)
  else:
    # ... (async 呼び出しのケース、一旦無視)

def flatten_types(ts):
  # flatten_typeはWIT型をcore wasm型に変換する関数
  return [ft for t in ts for ft in flatten_type(t)]
```

上は型の変換に関する定義だが、値の変換に関しても引数/返り値の数によってスタックに乗せたり、線形メモリにstore/loadしたりの切り替えが行われる。くわしくは[lifting and lowering values](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#lifting-and-lowering-values)

以下に用語をまとめておく。

- `lower_flat`: component valueをcore wasm valueに変換(スタック上に配置)
- `lift_flat`: core wasm valueをcomponent valueに変換(スタック上に配置)
- `lower_heap`/`store`: component valueを線形メモリ上に保存
- `lift_heap`/`load`: core wasm valueを線形メモリ上から読んで component valueに変換

以下ではWITの型が `lower/lift flat/heap` でそれぞれどのような変換がされるのかをできるだけ簡単にまとめておく

## bool

- `lower_flat`: Trueなら1、Falseなら0
- `lift_flat`: 0ならFalse、1以上ならTrue
  - core type: `i32`
- `store`/`load`: 0か1に変換して線形メモリ上に1バイトで保存

## u8/u16/u32/u64

- `lower_flat`: 値の変換なし
- `lift_flat`: core value(i32 or i64)の上位ビットは無視
  - 例えば32bitのcore valueを、u8のcomponent-level valueにliftする場合は上位24bitを無視
  - core type: `i32` (u64は `i64`)
- `load`/`store`: little endianで線形メモリ上に保存。u8からu64までそれぞれ1byte-4byte

## s8/s16/s32/i64

- `lower_flat`: 2の補数表現で表したバイト列を符号なし32bit整数として解釈
- `lift_flat`: u8-u64と同様に、core valueの上位ビットを無視、次に、値がターゲット型の符号付き範囲の上限よりも大きい場合、値からターゲット型の幅を引く。これにより、値がターゲット型の有効な符号付き範囲内にあることが保証される。
  - core type: `i32` (i64は `i64`)
- `load`/`store`: little endianで線形メモリ上に保存。s8からs64までそれぞれ1byte-4byte

```python
# https://github.com/WebAssembly/component-model/blob/ee4822aacbce083599f692fc8c8efb08db8d3f3a/design/mvp/canonical-abi/definitions.py#L1572-L1578
def lift_flat_signed(vi, core_width, t_width):
  i = vi.next('i' + str(core_width))
  assert(0 <= i < (1 << core_width))
  i %= (1 << t_width)
  if i >= (1 << (t_width - 1)):
    return i - (1 << t_width)
  return i
  ```

## f32/f64

maybe_scrable_nan32(f) という関数によってエンコードされる。

これは浮動小数点数がNaNの場合は(ComponentModelのオプションによって)deterministicなNaNを表現する値か、ランダムなi32の値を生成してそれを`f32.reinterpret_i32`でf32に変換する。それ以外の場合は値の変換なし

```python
def maybe_scramble_nan32(f):
  if math.isnan(f):
    if DETERMINISTIC_PROFILE:
      f = core_f32_reinterpret_i32(CANONICAL_FLOAT32_NAN)
    else:
      f = core_f32_reinterpret_i32(random_nan_bits(32, 8))
    assert(math.isnan(f))
  return f
```

float値をliftするときはNaNかどうかを調べ、NaNなら指定されているNaNに対応する整数値を`f32.reinterpret_i32`してcomponent valueに変換

```python
DETERMINISTIC_PROFILE = False # or True
CANONICAL_FLOAT32_NAN = 0x7fc00000
CANONICAL_FLOAT64_NAN = 0x7ff8000000000000

def canonicalize_nan32(f):
  if math.isnan(f):
    f = core_f32_reinterpret_i32(CANONICAL_FLOAT32_NAN)
    assert(math.isnan(f))
  return f
```

NaNの表現については深堀りしていない

- `lower_flat`: `maybe_scramble_nan32(f)`
- `lift_flat`: `canonicalize_nan32(f)`
- `store`: `maybe_scramble_nan32(f)`した値を`i32.reinterpret_f32`でi32として解釈した整数値をstore
- `load`: 整数値をloadし、`f32.reinterpret_i32`したあと`canonicalize_nan32`

## char

charは[Unicode Scalar Value](https://unicode.org/glossary/#unicode_scalar_value)であることを前提として値の変換はなし(`i32`)

そのため`lower`も`lift`もu32と同じ

## string
Component-levelでのstringは実は以下の3つ組で表現されている。

- `src`: 実際の文字列データ(具体例ではデコードしたpythonのstrを使っているが、VMの実装依存?)
- `src_encoding`: (core wasmでの)文字列のエンコーディングを示す文字列 ("utf8", "utf16", "latin1+utf16")
- `src_tagged_code_units`: `src_encoding`での文字列のエンコードされたcode unit数

`src_encoding`は `utf8`, `utf16`, `latin1+utf16` の3つがあり、`latin1+utf16` の場合は `src_tagged_code_units` の最上位bitが1の場合は`utf16`、0の場合は`latin1`というようにコンポーネント内で2つの文字コードを使い分けることができるらしい。

core wasmでのstringは、(`src_encoding`で)encodeされて線形メモリ場に配置され、線形メモリ上のoffsetとcode unitsの2つ組(`[i32, i32]`)で表現される。

> Since strings and variable-length lists are stored in linear memory, lifting can reuse the previous definitions; only the resulting pointers are returned differently (as i32 values instead of as a pair in linear memory). Fixed-length lists are lowered the same way as tuples
> https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#flat-lowering

- `lower_flat`
  - lower先のcomponentが指定しているencoding(`src_encoding`とは違うので注意)で`src`をエンコードして線形メモリにstoreし、そのoffsetとlengthを返す
  - このとき(多分各Componentが提供する)[realloc](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md#canonical-abi)を使ってメモリを確保するのだが、 `src_encoding` と `src_tagged_code_units` がメモリを確保する際のヒントとなる。
- `lift_flat`
  - 2つ組(`[i32, i32]`)を元に線形メモリからバイト列をload
  - lift元のcomponentが指定しているencodingに従って、バイト列を文字列にデコード
  - デコードした文字列・encoding・code unit数 の3つ組をcomponent-level stringとする
- `load`/`store`
  - `lower/lift_flat`と同様に(rellocで確保した)線形メモリ上にデータをstore/loadして
  - そのoffsetとcode unit数を線形メモリ上にstore/load

> Storing strings is complicated by the goal of attempting to optimize the different transcoding cases. In particular, **one challenge is choosing the linear memory allocation size before examining the contents of the string.** The reason for this constraint is that, in some settings where single-pass iterators are involved (host calls and post-MVP adapter functions), examining the contents of a string more than once would require making an engine-internal temporary copy of the whole string, which the component model specifically aims not to do. **To avoid multiple passes, the canonical ABI instead uses a realloc approach to update the allocation size during the single copy.** A blind realloc approach would normally suffer from multiple reallocations per string (e.g., using the standard doubling-growth strategy). However, as already shown in load_string above, **string values come with two useful hints: their original encoding and byte length. From this hint data, store_string can do a much better job minimizing the number of reallocations.**
> https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#storing

詳細は [store_string](https://github.com/WebAssembly/component-model/blob/ee4822aacbce083599f692fc8c8efb08db8d3f3a/design/mvp/CanonicalABI.md#storing) と [load_string](https://github.com/WebAssembly/component-model/blob/ee4822aacbce083599f692fc8c8efb08db8d3f3a/design/mvp/CanonicalABI.md#loading)

## list

list型は以下の2つの要素でparametalizeされている

- `t`: 要素型
- `l`: リストの長さ(`None`の場合は可変長リスト)

```python
@dataclass
class ListType(ValType):
  t: ValType
  l: Optional[int] = None
```


- 固定長の場合
  - 各要素をlowerして並べた値
  - 例えば `ListType(i32, 3)` なら `[i32, i32, i32]` という型になるし
  - `ListType(string, 2)` なら `[i32, i32, i32, i32]` になる
- 可変長の場合
  - stringと同様に線形メモリに値をstore(配置は前から順番にリストの中身を置いていくだけ)
  - その先頭のoffsetとengthの2つ組(`[i32, i32]`)がcore wasm valueになる

というように固定長の場合と可変長の場合でcore wasmでの表現が異なる

- `lower_flat`
  - 上記のルールにしたがって変換するだけ
- `lift_flat`
  - liftターゲットの型が固定長の場合はそれぞれのcore valueをliftしていくだけ
  - 可変長の場合は2つ組で表現された線形メモリの範囲、要素型の`elem_size`ずつ値をloadしてliftしていく
    - `elem_size` はcomponent-level typeの、可変長リスト上でのlength
- `load`/`store`
  - 固定長: `flat`だとスタックに配置していたものが、線形メモリ上に置かれるだけ
  - 可変長: stringと同様に(reallocで確保した範囲に)データを配置し、そのoffsetとlengthの2つ組を線形メモリ上に置く

## record

record型は`FieldType`のリストでparametalizeされていて、`FieldType`はラベルの名前と要素型を持つ。

```python
@dataclass
class FieldType:
  label: str
  t: ValType

@dataclass
class RecordType(ValType):
  fields: list[FieldType]
```

**component-level valueはlabel-valueの連想配列になっている。**

### lower_flat/lift_flat

- `lower_flat`:
  - fieldsの値を順番にlower_flatするだけ
  - 例えば `{"a": u32, "b": string}` は `[i32, i32, i32]` に lower される。field名はcore wasmには渡らない
- `lift_flat`:
  - fieldsを順番に要素型で `lift_flat` して、labelづけしていくだけ
  - `[i32, i32, i32]` を `{"a": u32, "b": string}` にliftしようとすると、最初のi32をまずu32としてliftして"a"の値に割当、次に`[i32, i32]`をstringとしてlift...という感じになる

### load/store

メモリ上に配置する場合は少しむずかしくて、alignemntとelem_sizeの話が必要になってくる。なぜなら **recordの各要素について、N-byteの要素はN-byte alignmetに配置したいという話がある**

recordを線形メモリに保存する場合は簡単に書くといかのような関数で表現される

```python
# v - recordを表す連想配列
# ptr - recordをstoreする線形メモリの開始位置
# fields - recordのフィールド
def store_record(cx, v, ptr, fields):
  # 各フィールドについて
  for f in fields:
    # alignment - WITの型を受け取ってそのalignmentを返す関数
    # alignt_to - ptrとalignmentを受け取って、ptrがそのalignment境界に達するようにpadding
    # まずそのフィールドの型のalignment境界までptrをpaddingしてptrを動かす
    ptr = align_to(ptr, alignment(f.t))
    # alignment境界までptrを動かしたらそこに値を保存
    store(cx, v[f.label], f.t, ptr)
    # elem_size - WITの型を受け取ってそのバイト数を返す関数
    # 保存した値の分だけptrを動かす
    ptr += elem_size(f.t)
```

例えば `(record (field "a" u32) (field "b" u8) (field "c" u16) (field "d" u8))` なレコードについて

- u32: alilgnment 4、要素サイズ 4
- u8: algnment 1、要素サイズ 1
- u16: alignment 2、要素サイズ 2
- record全体: alignment 4 (最も大きいフィールドのalignmentと同じ)、要素サイズ12 (後述)

(それぞれのalignmentと要素サイズについては [alignment](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#alignment) と [elem_size](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#element-size) を参照、recordのelem_sizeは下の説明を読んだあとだとわかりやすいかも)

- a (s = 0)
  - `align_to(0, 4) = 0` なのですでに alignment に offset が来ている
  - `s += 4`
- b (s = 4)
  - `align_to(4, 1) = 4`
  - `s += 1`
- c (s = 5)
  - `align_to(5, 2) = 6` 2バイト要素は2バイトalignmentに配置されてほしいので、align_toにより6バイト目にalignされる
  - `s += 2`
- d (s = 8)
  - `align_to(8, 1) = 8`
  - `s += 1`

よって最終的には以下のように配置されるはず

| byte  | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|-------|---|---|---|---|---|---|---|---|---|
| field | a | a | a | a | b | - | c | c | d |

そして record の elem_size は 9のように見えるが、実は最後に、`s`をレコード全体のalignmentに揃える。これによりこのrecordの後続のデータ構造が正しくalignされていることを保証するする。

`align_to(9, 4) = 12` これで record全体のサイズは12となる。

recordを線形メモリから読むときは逆の操作をしてあげると良い

```python
def load_record(cx, ptr, fields):
  record = {}
  for field in fields:
    ptr = align_to(ptr, alignment(field.t))
    record[field.label] = load(cx, ptr, field.t)
    ptr += elem_size(field.t)
  return record
```


## variant

```python
@dataclass
class CaseType:
  label: str
  t: Optional[ValType]

@dataclass
class VariantType(ValType):
  cases: list[CaseType]
```

例えば `(variant (case "a" u32) (case "b" string))` のようなvariant型について

- component-level valueは `{"a": 42}` や `{"b": "foo"}` のような1要素のみを持つ辞書配列(keyにはvariant型のcaseラベルを持ち、valueにはそのcaseの型に対応する値を持つ)
- core valueは `[<case_indexのi32>, <lower values>...]`
  - 先頭に `i32`: case_index。例えば `{"a": 42}` の場合は0番目のcaseにマッチするので0、`{"b": "foo"}`は1番目のcaseにマッチするので1
  - その後にvalueのlowered valueになる。例えば42(`i32`)はcore wasmでもi32。文字列は`[i32,i32]`で表現されるのだった。
  - なので、例えば`[0, 42]` (0番目のcaseにマッチして、値はi32型の42)や、`[1, 100, 3]` (1番目のcaseにマッチし、値はoffset100に格納された長さ3の文字列)など

### 0-埋め
値がcore valueでどう変換されるかは分かったが、型はどうなるのだろうか?というのもcomponent-level typeとcore typeは1:1対応していてほしい。variantがどのケースかによって`[0, 42]`と`[1,100,3]`とで型が違うのでは具合が悪い。

どうするのかというと、一番valueの型が長いものに合わせる。今回の場合`(case "b" string)`のケースが一番長い型`[i32,i32,i32]`となるので、`(variant (case "a" u32) (case "b" string))`に対するcore typeは`[i32,i32,i32]`となる。`(case "a" u32)`などの長さが足りないケースは型が合うように末尾が0埋めされる(`[0,42,0]`となる)

### approximation
上記のケースのように値の型がある程度一致するなら良いのだけれど、`(variant (case "a" f64) (case "b" string))`とか値のcore wasm表現が長さだけでなく型も異なる場合はどうすればよいのか? `a: [f64]` vs `b: [i32, i32]` のとき `f64` と `i32` がtype mismatchしている。こういう場合は両方を表現可能なtightestな型を選択する、具体的には

```python
def join(a, b):
  if a == b: return a
  if (a == 'i32' and b == 'f32') or (a == 'f32' and b == 'i32'): return 'i32'
  return 'i64'
```

- `i32` vs `f32` -> i32 (`f32.reinterpret_i32`などで相互変換できる)
- その他の場合i64
  - `f64` vs `i64` なら `f64.reinterpret_i64` などで相互変換できるし
  - それ以外の任意のケースも `i64` なら全て表現できる

そのため `(variant (case "a" f64) (case "b" string))` の core wasm type は `[i32, i64, i32]` となる。

> Variant flattening is more involved due to the fact that each case payload can have a totally different flattening. Rather than giving up when there is a type mismatch, **the Canonical ABI relies on the fact that the 4 core value types can be easily bit-cast between each other** and defines a join operator to pick the tightest approximation. What this means is that, regardless of the dynamic case, all flattened variants are passed with the same static set of core types, which may involve, e.g., reinterpreting an f32 as an i32 or zero-extending an i32 into an i64.
> https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#flattening

reference typeどうしようもなくないかこれ



### lower_flat/lift_flat

- `lower_flat`
  - 値にマッチするvariantのcase indexを取得
  - 値をlower_flatする。variantのcore wasm型とtype mistmatchがあった場合はapproximationで型を合わせていく
    - f32の値を`i32.reinterpret`したり
  - 最後に長さがvariantの型に合うように0埋め
- `lift_flat`
  - core wasm valueの最初の値はマッチするケースのindexと、それに対する`label`を取得しておく
  - variant では `f32` が `i32` で保存されてたりするのでf32にdecodeしたりしていく。全部読んだらそれを `lift_flat` してそれを `value` とする
  - 0埋めの部分は読み捨て
  - `{label: value}`

### load/store

基本的には `lower_flat` で変換した値をそのまま線形メモリに変換するだけなのだけれど、少し違うところがある

`discriminant_type(cases)` は cases の数に応じて、先頭のcase_indexを保持するのに最低限必要なサイズを計算する。256以下なら1バイトで十分なのでu8、257-65536は16バイトで十分なのでu16

```python
def discriminant_type(cases):
  n = len(cases)
  assert(0 < n < (1 << 32))
  match math.ceil(math.log2(n)/8):
    case 0: return U8Type()
    case 1: return U8Type()
    case 2: return U16Type()
    case 3: return U32Type()
```

variantを線形メモリに保存するときは

- まず `discriminant_type` で計算した型の範囲に、マッチするcaseのインデックスを保存
- variantのケースの中で最長のケースのalignment境界までptrを移動
  - 何故ならvariantのpayloadは、すべてのケースのうち最長のものに合わせたサイズが必要となるため

```python
def store_variant(cx, v, ptr, cases):
  case_index, case_value = match_case(v, cases)
  disc_size = elem_size(discriminant_type(cases))
  store_int(cx, case_index, ptr, disc_size)
  ptr += disc_size
  ptr = align_to(ptr, max_case_alignment(cases))
  c = cases[case_index]
  if c.t is not None:
    store(cx, case_value, c.t, ptr)

def match_case(v, cases):
  [label] = v.keys()
  [index] = [i for i,c in enumerate(cases) if c.label == label]
  [value] = v.values()
  return (index, value)
```

load_variantはこの逆の操作をするだけ

## flags

例えば `(flags "a" "b" "c") `という flag型に対して

- component-level value: `{"a": False, "b": False, "c": True}` など
- core wasm value: `0b100` (cがTrueで、abはFalse) なbitvector
  - labelの先頭(a)から **least** significant bit から順に0か1を置いていく。

lift_flat/lower_flatはこの対応から自明。store/loadもbitvectorを整数値に変換してそれを線形メモリにload/storeするだけ

## resource

resourceがなにかというのは [WITとRust：リソース編](https://zenn.dev/chikoski/articles/wit-and-rust-resource) なんかを見ると分かりやすそう。

- [resource.new](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#canon-resourcenew) はresource representation(現状はi32のみ利用可能)を受け取って、Componentに登録したResourceHandleを指し示すindex(i32)を返す。
  - resoruce repにはリソース実体が格納されているメモリの開始位置とかが入るんですかね
- [resource.rep](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#canon-resourcerep)
  - resource representationを直接利用したいとき、resource handle index を受け取って、resource rep を返す。
  - いまいちどういう時に使いたいのかよく分かってない
- [resource.drop](https://github.com/WebAssembly/component-model/blob/main/design/mvp/CanonicalABI.md#canon-resourcedrop)
  - resource handle index を受取、ComponentからResource Handleを削除
  - またそのリソースが所有権を持っている場合はdestructorを実行する
  - destructorでresource実体を保持していたメモリ領域を開放するみたいな感じ?reference type対応どうすればいいのかな〜

正直これらのcanonical instructionsの使い方がいまいち分かってない...。けど少なくとも他のcomponentが作ったリソースを自分たちのcore moduleで利用する場合は、resouce handle index (i32)が渡ってくることだけ分かれば今は十分かな(自分たちのwasm moduleがresourceをexportできるようにしたくなってくると困る)

> In the Component Model, handles are lifted-from and lowered-into i32 values that index an encapsulated per-component-instance handle table that is maintained by the canonical function definitions described below. In the future, handles could be backwards-compatibly lifted and lowered from [reference types] (via the addition of a new canonopt, as introduced below).

とのことで、reource handleは、そのresourceを指し示すComponentが内のResourceTableのindexにlowerされる(これをしてfile descriptorみたいなものと言っているんだね)

`lower_borrow` と `lower_own` は、それぞれ borrow/own handleを現在のコンポーネントインスタンスのハンドルテーブルに新しい`ResourceHandle`要素を追加しつつ、core wasm表現として`i32`(resource handle index)を返す。
VM的には借用したリソースと、所有権を持つリソースをlower/liftするときの挙動が違うのだけれど、コンパイラ的にはその違いは見えなくてとにかく `i32` にlowerされてくる。

- `lift_borrow` と `lift_own` はそれぞれResource Handle index(i32)を受け取って、`resource.rep`を返す。その過程でResourceHandle Tableからresourceを削除したりしなかったりする。このあたりはちゃんと調べてない。
- `load`/`store` は `(lift|lower)_(borrow|own)` を resource handle にしてその i32 を線形メモリにload/storeするだけ

## 例

[output-stream resource の blocking-write-and-flush 関数](https://github.com/WebAssembly/WASI/blob/5120d70557cc499b98ee21b19f6066c29f1400a4/wasip2/io/streams.wit#L154-L181)をimportしたときの core wasm側の型はどうなるでしょうか?

```wit
resource output-stream {
  blocking-write-and-flush: func(
      contents: list<u8>
  ) -> result<_, stream-error>;
}
```

答えは `(func (param i32 i32 i32 i32))` なぜなら

- 最初の3つのi32はそれぞれ、resource (i32), 可変長リスト(i32, i32)
- 最後のi32はresultを受け渡しするためのポインタ
  - result型が複数の core wasm typeをもつ場合は線形メモリを経由してデータをやり取りして、パラメータにi32のポインタを置くんだった
  - `result<_, stream-error>` は `(variant (case "ok") (case "error" stream-error))` stream-errorはまたvariant型、明らかに2つ以上のi32によって表されることになりますね。(じゃあ線形メモリにどう配置されるかは省略...)
