---
title: "wit-bindgen / cargo-component が生成するstructにderiveを追加する(小ネタ)"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "wasm"]
published: true
---

例えば以下のようなWITがあったときに

```wit
package component:testing;

world run {
  export wasi:cli/run@0.2.0;
  import tests;
}

interface tests {
  record point { x: s32, y: s32 }
  roundtrip-point: func(a: point) -> point;
}
```

`wit-bindgen` (0.17.0) はデフォルトではこういうコードを書いてくれる。

```rust
pub mod component {
  pub mod testing {
    pub mod tests {
      #[repr(C)]
      #[derive(Clone, Copy)]
      pub struct Point {
        pub x: i32,
        pub y: i32,
      }
    }
  }
}
```

けど、`assert_eq!` とか使おうと思ったら `PartialEq` も derive してほしい！

```rust
#[allow(warnings)]
mod bindings;

use crate::bindings::exports::wasi::cli::run::Guest as Run;
use crate::bindings::component::testing::tests;

struct Component;

impl Run for Component {
    fn run() -> Result<(), ()> {
      let p = tests::Point { x: 0, y: 3 };
      assert_eq!(
        tests::roundtrip_point(p), p
      );
      return Ok(());
    }
}

bindings::export!(Component with_types_in bindings);
```

https://github.com/bytecodealliance/wit-bindgen/pull/678

https://github.com/bytecodealliance/cargo-component/issues/148

このあたりで custom derive attributes を指定できるようになっていそうで、`Cargo.toml` に以下のように書くと良いっぽい。

```toml
[package.metadata.component.bindings]
derives = ["PartialEq"]
```

