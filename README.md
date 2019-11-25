# Rust with Swift

## Getting started
参照: [Building and Deploying a Rust library on iOS](https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-06-rust-on-ios.html)

iOS アプリ用にビルドするので、Xcode 必要  
`xcode-select --install`  
  
`rustup` インストールする。`rustup` は Rust の installer。Rust 言語のバージョン管理も兼ねる。(必ずしもこれ使って install しなくても良いみたいだけど、言語のバージョン管理系は入れてた方が良い)    
`curl https://sh.rustup.rs -sSf | sh`  
  
iOS の architecture 向けの build 作れるようにする。    
`rustup target add aarch64-apple-ios armv7-apple-ios armv7s-apple-ios x86_64-apple-ios i386-apple-ios`  
(32bit 向けのはいらない気がしてるけど、念のため入れておこう…)  
  
iOS 上で動くライブラリを生成するツールをインストールする。これ入れないで自分で作ろうとするとすごい大変だぞと書いてある。@@後で何してるか見てみよう [cargo-lipo](https://github.com/TimNN/cargo-lipo)  
`cargo install cargo-lipo`  
`cargo`: Rust の公式 installer 使うと標準で同梱されてる package manager 兼 build system.  
  
準備完了。プロジェクト作る。  
```
mkdir rust_with_swift # 動作確認するコード含めてここで作業する
cargo new rust_sample_lib # Rust のプロジェクト作る。名前は rust_sample_lib
mkdir ios # 動作確認用の iOS app のプロジェクトはここに入れる
```

Rust 側の function 作る  
実際は C 言語でブリッジングする。  
> function の先頭についてる `#[no_mangle]` は、Rust の default の mangle しないで、C 言語の関数のように外部に見せるために記述している。  

たぶん「デフォルトだと Rust の mangle 機構で実際の名前が変わってしまって Rust の外から呼び出すとき見つからなくなってしまう。`#[no_mangle]`属性つけると mangle されなくなるから、外部からそのままの名前で見つけられる」ということだと思う。  

> C 言語の関数のように外部に見せるために記述している。

mangle されると呼び出し側が Rustc の mangle 機構しらないといけないけど、mangle しないでおけば Rust であることを隠蔽できるぞ、ということかな。  

>  `extern`  
> Rustc に「外から呼ばれる function だぞ」と知らせる。  
> C calling conventions に従って呼び出せるようにする。

この場合の `extern` は `extern "C"` の省略形。  
`extern` は ABI の指定を意味するっぽい。`extern "Rust"` なら Rust の ABI に従う。(`extern` ついてないときと同じ挙動)  
参照: https://doc.rust-lang.org/reference/items/external-blocks.html#abi  
参照: [A little Rust with your C](https://rust-embedded.github.io/book/interoperability/rust-with-c.html)  

```rust
use std::os::raw::{c_char};
use std::ffi::{CString, CStr};

#[no_mangle]
pub extern fn rust_greeting(to: *const c_char) -> *mut c_char {
    let c_str = unsafe { CStr::from_ptr(to) };
    let recipient = match c_str.to_str() {
        Err(_) => "there",
        Ok(string) => string,
    };

    CString::new("Hello ".to_owned() + recipient).unwrap().into_raw()
}

#[no_mangle]
pub extern fn rust_greeting_free(s: *mut c_char) {
    unsafe {
        if s.is_null() { return }
        CString::from_raw(s)
    };
}
```
↓ 外部からこのように見える  
```c
#include <stdint.h>

const char* rust_greeting(const char* to);
void rust_greeting_free(char *);
```

iOS app project 作る

Rust で作った library を link する  
~libresolv.tbd も追加する~  <- TODO: 何用？？？調べる  

bridging header つくる  

.h ファイルないんだけど、どうするんだ…  
-> cbindgen 使って生成すれば良さそう https://github.com/eqrion/cbindgen  
header file import する  
.a ファイルを target に含める  
動いた

途中から説明通りにやっても動かなくて苦しい戦いになった

cdylib <- C dynamic library? かな。TODO: 種類の意味調べる

## References
### Getting started
* [Building and Deploying a Rust library on iOS](https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-06-rust-on-ios.html)
* [cargo-lipo](https://github.com/TimNN/cargo-lipo)
* [Rust on iOS](https://medium.com/visly/rust-on-ios-39f799b3c1dd)