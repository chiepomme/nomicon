# FFI（他言語関数インターフェース）

# イントロダクション

このガイドでは、他言語で書かれたコードに対するバインディングの書き方を紹介するために、
圧縮・解凍ライブラリである [snappy](https://github.com/google/snappy) を使用します。
Rust は現時点では C++ のライブラリを直接呼び出すことができませんが、snappy は C言語のインターフェースを持っています。
(snappy の C言語のインターフェースは [`snappy-c.h`](https://github.com/google/snappy/blob/master/snappy-c.h) でドキュメント化されています。)

## libc についてのメモ

これらのサンプルコードの多くは [`libc` クレート][libc] を使用しています。
`libc` クレートは、その機能の一部として様々な C言語の型の定義を提供してくれます。
もしサンプルコードを動かす場合には、`Cargo.toml` に `libc` を加える必要があるでしょう。

```toml
[dependencies]
libc = "0.2.0"
```

[libc]: https://crates.io/crates/libc

それから、`extern crate libc;` をクレートのルートに追加してください。
（訳注：Rust 2018 Edition では必要ありません。）

## 他言語の関数を呼び出す

これは他言語の関数を呼び出す最小のサンプルです。snappy がインストールされている場合にコンパイルできます。

```rust,ignore
extern crate libc;
use libc::size_t;

#[link(name = "snappy")]
extern {
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
}

fn main() {
    let x = unsafe { snappy_max_compressed_length(100) };
    println!("100 バイトのバッファを圧縮したときの最大サイズ: {}", x);
}
```

`extern` ブロックは、他言語のライブラリが持つ関数のシグネチャのリストです。
この場合はプラットフォームの C ABI に基づきます。

`#[link(...)]` アトリビュートは、リンカに snappy のライブラリとリンクすることを指示します。
これによってシンボルが解決できるようになります。

他言語関数は unsafe と見なされるため、 `unsafe {}` で囲う必要があります。
そうすることで、このブロックの中は完全に安全だということをコンパイラーに伝えます。

C言語のライブラリは、スレッドセーフではないインターフェースを公開していることが多くあります。
また、ポインタを引数に取るほぼ全ての関数が、どんな入力が来ても安全である、とはいえません。全てのポインタはダングリングポインタの可能性があるためです。
更に、生ポインタは Rust の安全なメモリ管理モデルから外れてしまいます。

他言語の関数に対して引数の型を宣言する場合、Rust コンパイラーはその宣言が正しいかどうかを検証できません。
正しく型を明記することは、実行時にバインディングが正しく動作するために必要なことです。

先ほどの `extern` ブロックを、snappy の API を網羅するように拡張でき、その場合、以下のようになります。

```rust,ignore
extern crate libc;
use libc::{c_int, size_t};

#[link(name = "snappy")]
extern {
    fn snappy_compress(input: *const u8,
                       input_length: size_t,
                       compressed: *mut u8,
                       compressed_length: *mut size_t) -> c_int;
    fn snappy_uncompress(compressed: *const u8,
                         compressed_length: size_t,
                         uncompressed: *mut u8,
                         uncompressed_length: *mut size_t) -> c_int;
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
    fn snappy_uncompressed_length(compressed: *const u8,
                                  compressed_length: size_t,
                                  result: *mut size_t) -> c_int;
    fn snappy_validate_compressed_buffer(compressed: *const u8,
                                         compressed_length: size_t) -> c_int;
}
# fn main() {}
```

# 安全なインターフェースを作る

メモリ安全性、それから、ベクタのような高レベルな概念を提供するためには、生の C API をラップする必要があります。
ライブラリは、安全で高レベルなインターフェースのみを公開し、安全ではない内部的な実装の詳細を隠すという選択肢もあります。

バッファが必要な関数関数をラップする際には、`slice::raw` モジュールを使用して、 Rust のベクタをメモリに対するポインタとして操作することができます。
（訳注：Rust 1.48.0 ではそのようなモジュールが見当たりませんでした。）
Rust のベクタは、連続したメモリ領域であることが保証されています。
ベクタの length は現在保持している要素の数、capacity は確保したメモリの総量を要素の数で表したものです。
length は必ず capacity 以下になります。

```rust,ignore
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_validate_compressed_buffer(_: *const u8, _: size_t) -> c_int { 0 }
# fn main() {}
pub fn validate_compressed_buffer(src: &[u8]) -> bool {
    unsafe {
        snappy_validate_compressed_buffer(src.as_ptr(), src.len() as size_t) == 0
    }
}
```

上記の `validate_compressed_buffer` のラッパーは、関数のシグネチャに `unsafe` を付けずに、
`unsafe` ブロックを使用することで、全ての入力に対して安全であると言う保証をしています。

`snappy_compress` 関数と `snappy_uncompress` 関数は、
出力を受けとるためのバッファも必要なため、より複雑になります。

`snappy_max_compressed_length` 関数は、圧縮後の最大のサイズを返すものです。
圧縮の出力を受け取るのに使用するベクタを作る際に、必要な容量を確保するために使用します。
ここで確保されたベクタは`snappy_compress` 関数に、出力のための引数として渡されます。

圧縮後の実際のサイズを受け取るのにも、出力のための引数が使用されています。

```rust,ignore
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_compress(a: *const u8, b: size_t, c: *mut u8,
#                           d: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(a: size_t) -> size_t { a }
# fn main() {}
pub fn compress(src: &[u8]) -> Vec<u8> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen = snappy_max_compressed_length(srclen);
        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        snappy_compress(psrc, srclen, pdst, &mut dstlen);
        dst.set_len(dstlen as usize);
        dst
    }
}
```

解凍も同様の手順で行えます。snappy は圧縮前のサイズ情報を圧縮後のバイナリに含んでおり、
`snappy_uncompressed_length` によって、事前に正確なサイズを知ることができるためです。


```rust,ignore
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_uncompress(compressed: *const u8,
#                             compressed_length: size_t,
#                             uncompressed: *mut u8,
#                             uncompressed_length: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_uncompressed_length(compressed: *const u8,
#                                      compressed_length: size_t,
#                                      result: *mut size_t) -> c_int { 0 }
# fn main() {}
pub fn uncompress(src: &[u8]) -> Option<Vec<u8>> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen: size_t = 0;
        snappy_uncompressed_length(psrc, srclen, &mut dstlen);

        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        if snappy_uncompress(psrc, srclen, pdst, &mut dstlen) == 0 {
            dst.set_len(dstlen as usize);
            Some(dst)
        } else {
            None // SNAPPY_INVALID_INPUT
        }
    }
}
```

次に、これらの関数の使い方を示すために、テストを書いてみましょう。

```rust,ignore
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_compress(input: *const u8,
#                           input_length: size_t,
#                           compressed: *mut u8,
#                           compressed_length: *mut size_t)
#                           -> c_int { 0 }
# unsafe fn snappy_uncompress(compressed: *const u8,
#                             compressed_length: size_t,
#                             uncompressed: *mut u8,
#                             uncompressed_length: *mut size_t)
#                             -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(source_length: size_t) -> size_t { 0 }
# unsafe fn snappy_uncompressed_length(compressed: *const u8,
#                                      compressed_length: size_t,
#                                      result: *mut size_t)
#                                      -> c_int { 0 }
# unsafe fn snappy_validate_compressed_buffer(compressed: *const u8,
#                                             compressed_length: size_t)
#                                             -> c_int { 0 }
# fn main() { }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid() {
        let d = vec![0xde, 0xad, 0xd0, 0x0d];
        let c: &[u8] = &compress(&d);
        assert!(validate_compressed_buffer(c));
        assert!(uncompress(c) == Some(d));
    }

    #[test]
    fn invalid() {
        let d = vec![0, 0, 0, 0];
        assert!(!validate_compressed_buffer(&d));
        assert!(uncompress(&d).is_none());
    }

    #[test]
    fn empty() {
        let d = vec![];
        assert!(!validate_compressed_buffer(&d));
        assert!(uncompress(&d).is_none());
        let c = compress(&d);
        assert!(validate_compressed_buffer(&c));
        assert!(uncompress(&c) == Some(d));
    }
}
```

# デストラクタ

他言語のライブラリは多くの場合、コードを呼び出すためにリソースの所有権を渡します。
このような場合には、安全性の確保とリソースの破棄を確実に行えるように Rust のデストラクタを使用する必要があります。（特にパニックする場合。）
デストラクタについて詳しくは [Drop トレイト](../std/ops/trait.Drop.html) を参照してください。

# C の関数から Rust の関数へのコールバック

外部のライブラリによっては、現在の状態や中間データを呼び出し元に返すために、コールバックを必要とする場合があります。
その際には Rust 側で定義した関数を、外部のライブラリに渡すことが可能です。
コールバック関数に、正しい呼び出し規約に基づいて `extern` という印を付けることで、C から呼び出しが可能になります。

It is possible to pass functions defined in Rust to an external library.
The requirement for this is that the callback function is marked as `extern`
with the correct calling convention to make it callable from C code.

コールバック関数は、コールバック登録関数で C ライブラリに渡し、その後 C 言語内から呼び出すといった使い方が可能でしょう。

The callback function can then be sent through a registration call
to the C library and afterwards be invoked from there.

以下は標準的なサンプルです：

Rust のコード:

```rust,no_run
extern fn callback(a: i32) {
    println!("I'm called from C with value {0}", a);
}

#[link(name = "extlib")]
extern {
   fn register_callback(cb: extern fn(i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    unsafe {
        register_callback(callback);
        trigger_callback(); // コールバックをトリガーする
    }
}
```

C 言語のコード:

```c
typedef void (*rust_callback)(int32_t);
rust_callback cb;

int32_t register_callback(rust_callback callback) {
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(7); // Rust 側の callback(7) が呼び出される。
}
```

このサンプルでは、Rust 側の `main()` が C 側の `trigger_callback()` を呼び出し、
それに応じて C 側は Rust 側の `callback()` をコールバックします。

## Rust のオブジェクトに対するコールバック

これまでのサンプルでは、グローバルな関数を C から呼び出す方法を用いていました。
しかし、特定の Rust のオブジェクトに対してコールバックを呼び出したいというのは良くあることです。

The former example showed how a global function can be called from C code.
However it is often desired that the callback is targeted to a special
Rust object.

？これは、C のオブジェクトに対するラッパーとなるオブジェクトを用いて行えます。
This could be the object that represents the wrapper for the
respective C object.

これは、オブジェクトに対する生ポインタを C のライブラリに渡すことで達成が可能です。
C のライブラリは、この Rust のオブジェクトへのポインタを通知（コールバック）に含められます。
これによって、安全ではないものの、参照された Rust のオブジェクトにコールバック関数内からアクセスすることができます。

Rust のコード：

```rust,no_run
#[repr(C)]
struct RustObject {
    a: i32,
    // Other members...
}

extern "C" fn callback(target: *mut RustObject, a: i32) {
    println!("I'm called from C with value {0}", a);
    unsafe {
        // RustObject の値をコールバックから受け取った値で更新する:
        (*target).a = a;
    }
}

#[link(name = "extlib")]
extern {
   fn register_callback(target: *mut RustObject,
                        cb: extern fn(*mut RustObject, i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    // コールバック内から参照されるオブジェクトを作成する:
    let mut rust_object = Box::new(RustObject { a: 5 });

    unsafe {
        register_callback(&mut *rust_object, callback);
        trigger_callback();
    }
}
```

C のコード：

```c
typedef void (*rust_callback)(void*, int32_t);
void* cb_target;
rust_callback cb;

int32_t register_callback(void* callback_target, rust_callback callback) {
    cb_target = callback_target;
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(cb_target, 7); // Rust 側の callback(&rustObject, 7) が呼び出される。
}
```

## 非同期コールバック

前のサンプルでは、コールバックは外部の C 言語ライブラリの関数を呼び出した時の直接的な応答として呼び出されていました。
つまり、現在のスレッドの制御は Rust から C へと切り替わった後、コールバックを実行するため Rust へと切り替わりますが、
最終的にコールバックは最初の呼び出しと同じスレッドで呼び出されます。

外部のライブラリが自前でスレッドを生成し、そこからコールバックを呼び出す場合、もっと複雑になります。
これらの場合には、コールバック内での Rust のデータ構造へのアクセスは特に危険なため、適切な同期の仕組みが必要となります。
Rust では、ミューテックスといった古典的な同期の仕組みに加えて、チャンネル（`std::sync::mpsc`）を用いることで、
コールバックを呼び出した C のスレッドから Rust のスレッドへとデータを送ることができます。

もし、非同期コールバックが Rust のアドレス空間の特定のオブジェクトを対象とする場合、
Rust のオブジェクトが破棄された後にコールバックが絶対に呼び出されないようにする必要があります。
If an asynchronous callback targets a special object in the Rust address space
it is also absolutely necessary that no more callbacks are performed by the
C library after the respective Rust object gets destroyed.

これを達成するためには、オブジェクトのデストラクタでコールバックの登録を解除し、
登録解除後にコールバックが呼び出されないことを保証できるようにライブラリを設計しなくてはいけません。


# リンク

`extern` ブロックの `link` アトリビュートは、
rustc に対して、どのようにネイティブライブラリとリンクすれば良いかを指示するための、基本構造を提供します。

The `link` attribute on `extern` blocks provides the basic building block for
instructing rustc how it will link to native libraries.

link アトリビュートは現在のところ 2通りの書き方ができます：

* `#[link(name = "foo")]`
* `#[link(name = "foo", kind = "bar")]`

どちらの場合も `foo` はリンクするネイティブライブラリの名前を表し、
2つ目の例の `bar` は、リンクするネイティブライブラリのタイプを表します。
ネイティブライブラリのタイプは、現在のところ、以下の 3つが存在します：

* ダイナミック（動的） - `#[link(name = "readline")]`
* スタティック（静的） - `#[link(name = "my_build_dependency", kind = "static")]`
* フレームワーク） - `#[link(name = "CoreFoundation", kind = "framework")]`

framework は MacOS ターゲットでのみ有効です。

`kind` の値によって、ネイティブライブラリがどのようにリンクされるかが変わります。

リンクの観点から見ると、Rust のコンパイラは 2種類の特性を持った生成物を作成します。
部分的なもの (rlib/staticli) と、最終的なもの (dylib/binary) です。

ネイティブダイナミックライブラリとフレームワークの依存関係は、最終的な生成物の領域まで伝搬します。
一方で、スタティックライブラリの依存関係は伝搬しません。
スタティックライブラリは、直接次の生成物に組み込まれるからです。

このモデルがどのように適用されるかをいくつかの例でみてみましょう：

* ネイティブのビルド依存関係の例。Rust のコードを書くときに、C/C++ のグルーコードが必要となることがあります。
  しかし、C/C++ のコードをライブラリの形式で配布するのは大変です。
  この場合、このコードは `libfoo.a` にアーカイブされ、Rust のクレート内で依存関係を宣言します。
  このときは、`#[link(name = "foo", kind = "static")]` という記述となります。

  クレートの出力の特性に関わらず、ネイティブスタティックライブラリは出力に含まれます。
  そのため、ネイティブスタティックライブラリを配布する必要はありません。

* 通常の動的な依存関係の例。`readline` のような共通のシステムライブラリは、多くのシステムで利用可能です。
  また、静的なコピーはたいていの場合見つかりません。
  この依存関係が Rust のクレートに含まれる場合、
  部分ターゲット（例えば rlib）はシステムライブラリにリンクしませんが、
  バイナリのような最終的なターゲットに rlib が含まれる時には、ネイティブライブラリとリンクします。

MacOS では、フレームワークはダイナミックライブラリと同じように動作します。

# unsafe ブロック

一部の操作、たとえば生ポインターの参照外しや、unsafe とマークされた関数を呼び出すといったものは、unsafe ブロックの内側でのみ許可されています。
unsafe ブロックは危険性を分離し、その危険性がブロックの外に漏れ出ないということをコンパイラーと約束します。
一方で、unsafe な関数は自分は危険であると外の世界に伝えます。
unsafe な関数は以下のように書かれます：

```rust
unsafe fn kaboom(ptr: *const i32) -> i32 { *ptr }
```

この関数は、`unsafe` ブロックまたは、別の `unsafe` な関数からしか呼び出せません。

# 他言語のグローバル変数にアクセスする

他言語の API はよくグローバル変数へのアクセスを提供します。これはグローバルな状態を追跡する場合などに使われます。
こういった変数にアクセスするためには、編巣を `extern` ブロック内部で `static` キーワードを付けて宣言します：

```rust,ignore
extern crate libc;

#[link(name = "readline")]
extern {
    static rl_readline_version: libc::c_int;
}

fn main() {
    println!("readline バージョン {} がインストールされています。",
             unsafe { rl_readline_version as i32 });
}
```

または、他言語インターフェースによって提供されるグローバルな状態を変更する必要があるかもしれません。
そういった場合には、`static` に `mut` を付けて宣言することで、変更が可能になります。

```rust,ignore
extern crate libc;

use std::ffi::CString;
use std::ptr;

#[link(name = "readline")]
extern {
    static mut rl_prompt: *const libc::c_char;
}

fn main() {
    let prompt = CString::new("[my-awesome-shell] $").unwrap();
    unsafe {
        rl_prompt = prompt.as_ptr();

        println!("{:?}", rl_prompt);

        rl_prompt = ptr::null();
    }
}
```

読み込み書き込みに関わらず `static mut` に対する全ての操作は unsafe です。
グローバルな変更可能な状態を扱う場合、最大の注意を払う必要があります。

# 他言語関数の呼出規約

多くの他言語コードは C 言語の ABI を提供します。
Rust は他言語関数を呼び出す場合、デフォルトでプラットフォームの C 言語の呼出規約を使用します。
他言語関数によっては、特に Windows API においては、他の呼出規約を使用する場合があります。
Rust はコンパイラに対してどの呼出規約を使うべきかを伝える手段を提供します：

```rust,ignore
extern crate libc;

#[cfg(all(target_os = "win32", target_arch = "x86"))]
#[link(name = "kernel32")]
#[allow(non_snake_case)]
extern "stdcall" {
    fn SetEnvironmentVariableA(n: *const u8, v: *const u8) -> libc::c_int;
}
# fn main() { }
```

これは `extern` ブロック全体に適用されます。
対応している ABI 呼び出し規約は以下の通りです：

* `stdcall`
* `aapcs`
* `cdecl`
* `fastcall`
* `vectorcall`
これは現在は `abi_vectorcall` ゲートの裏に隠されており、変更される可能性があります。
* `Rust`
* `rust-intrinsic`
* `system`
* `C`
* `win64`
* `sysv64`

このリストの多くの ABI 呼出規約は名前の通りですが、`system` は少しおかしく感じるかも知れません。
この呼出規約は、ターゲットとする環境のライブラリを相互運用する上で適切な ABI を選択します。
たとえば、 x86 アーキテクチャの Win32 では、`stdcall` が使用され、
x86_64 アーキテクチャの場合には、Windows では `C` 呼出規約が使用されるため、`C` が選択されます。
つまり、一つ前のサンプルにおいて、`extern "system" { ... }` を使うことにより、
x86 だけではなく、全ての Windows に対して定義をできるということです。

# 他言語コードとの相互運用性

Rust は `#[repr(C)]` アトリビュートが適用されている `struct` に限り、
メモリレイアウトがプラットフォームの C 言語での表現と互換性があることが保証されます。
`#[repr(C, packed)]` では、構造体ののメンバはパディングなしでレイアウトされます。
`#[repr(C)]` は列挙体にも適用が可能です。

Rust の、所有しているボックス（`Box<T>`）は null 非許容ポインタを、抱えているオブジェクトを指すハンドルとして使用します。
しかしながら、そういったポインタは内部のアロケータによって管理されるため、自前で作成すべきではありません。
参照は直接型を指す null 非許容ポインタと安全に見なすことができます。
しかしながら、借用チェックや変更可能性ルールを破壊することは、安全が保証されません。
そのため、

Rust's owned boxes (`Box<T>`) use non-nullable pointers as handles which point
to the contained object. However, they should not be manually created because
they are managed by internal allocators. References can safely be assumed to be
non-nullable pointers directly to the type.  However, breaking the borrow
checking or mutability rules is not guaranteed to be safe, so prefer using raw
pointers (`*`) if that's needed because the compiler can't make as many
assumptions about them.

Vectors and strings share the same basic memory layout, and utilities are
available in the `vec` and `str` modules for working with C APIs. However,
strings are not terminated with `\0`. If you need a NUL-terminated string for
interoperability with C, you should use the `CString` type in the `std::ffi`
module.

The [`libc` crate on crates.io][libc] includes type aliases and function
definitions for the C standard library in the `libc` module, and Rust links
against `libc` and `libm` by default.

# 可変長引数関数

C 言語では、関数が可変長引数を取ることができます。
Rust では他言語の関数の宣言時に引数の最後に `...` を明記することで、可変長引数関数を宣言できます：

```no_run
extern {
    fn foo(x: i32, ...);
}

fn main() {
    unsafe {
        foo(10, 20, 30, 40, 50);
    }
}
```

ただし、通常の Rust の関数は可変長引数関数には*できません*：

```ignore
// これはコンパイルできません

fn foo(x: i32, ...) { }
```

# 「null 許容ポインタ最適化」

特定の Rust の型は絶対に `null` にならないと定義されています。
これには参照（`&T`, `&mut T`）、ボックス（`Box<T>`）、関数ポインタ（`extern "abi" fn()`）が含まれます。
C とつなぎ合わせる場合、`null` になる可能性のあるポインタがよく使われますが、
そういったポインタでは、厄介な `transmute` や unsafe コードを使って、
Rust の型への、もしくは、Rust の型からの変換を扱う必要があります。
しかしながら、Rust にはこの問題に対するワークアラウンドがあります。

ある特定の場合で `enum` は「null 許容ポインタ最適化」に使用することができます。
特定の場合とは `enum` が丁度 2つの variant を持っていて、そのうち 1つはデータを持たず、
もう 1つは、先ほど述べた Null 非許容型のフィールドを持っている場合です。
これは判別のための余分なスペースが必要ないということであり、
null 非許容フィールドに `null` 値を置くことによって空の variant が表されます。？
これは「最適化」と呼ばれていますが、他の最適化と異なり、資格のある型に適用されるという保証があります。

As a special case, an `enum` is eligible for the "nullable pointer optimization" if it contains
exactly two variants, one of which contains no data and the other contains a field of one of the
non-nullable types listed above.  This means no extra space is required for a discriminant; rather,
the empty variant is represented by putting a `null` value into the non-nullable field. This is
called an "optimization", but unlike other optimizations it is guaranteed to apply to eligible
types.

Null 許容ポインタ最適化を利用するもっとも一般的な型は `Option<T>` です。`None` が `null` に対応します。
`Option<extern "C" fn(c_int) -> c_int>` は、 C 言語 ABI を使用する上で、Null 許容関数ポインタを表す正しい方法です。
（これは C の型でいう `int (*)(int)`に対応します。）

これは不自然なサンプルです。C のライブラリがコールバックを登録する機能を持っていて、特定の場面で呼び出されるとしましょう。
コールバックは関数ポインタと整数を渡し、その関数はその整数を引数として呼び出されると考えられます。
つまり、関数ポインタが FFI の境界を双方向に飛び越えることになります。

Here is a contrived example. Let's say some C library has a facility for registering a
callback, which gets called in certain situations. The callback is passed a function pointer
and an integer and it is supposed to run the function with the integer as a parameter. So
we have function pointers flying across the FFI boundary in both directions.

```rust,ignore
extern crate libc;
use libc::c_int;

# #[cfg(hidden)]
extern "C" {
    /// コールバックを登録する。
    fn register(cb: Option<extern "C" fn(Option<extern "C" fn(c_int) -> c_int>, c_int) -> c_int>);
}
# unsafe fn register(_: Option<extern "C" fn(Option<extern "C" fn(c_int) -> c_int>,
#                                            c_int) -> c_int>)
# {}

/// この極めて無意味な関数は、関数ポインタと整数を C 側から受け取り、
/// 渡された整数を引数として関数を実行し、その結果を返す。
/// もし関数が渡されなかった場合、デフォルトの動作として整数を二乗して返す。
extern "C" fn apply(process: Option<extern "C" fn(c_int) -> c_int>, int: c_int) -> c_int {
    match process {
        Some(f) => f(int),
        None    => int * int
    }
}

fn main() {
    unsafe {
        register(Some(apply));
    }
}
```

C 側のコードはこのようになります：

```c
void register(void (*f)(int (*)(int), int)) {
    ...
}
```

`transmute` は必要ありません！

# Rust のコードを C 言語から呼び出す

You may wish to compile Rust code in a way so that it can be called from C. This is
fairly easy, but requires a few things:

```rust
#[no_mangle]
pub extern "C" fn hello_rust() -> *const u8 {
    "Hello, world!\0".as_ptr()
}
# fn main() {}
```

The `extern "C"` makes this function adhere to the C calling convention, as
discussed above in "[Foreign Calling
Conventions](ffi.html#foreign-calling-conventions)". The `no_mangle`
attribute turns off Rust's name mangling, so that it is easier to link to.

# FFI and panics

It’s important to be mindful of `panic!`s when working with FFI. A `panic!`
across an FFI boundary is undefined behavior. If you’re writing code that may
panic, you should run it in a closure with [`catch_unwind`]:

```rust
use std::panic::catch_unwind;

#[no_mangle]
pub extern fn oh_no() -> i32 {
    let result = catch_unwind(|| {
        panic!("Oops!");
    });
    match result {
        Ok(_) => 0,
        Err(_) => 1,
    }
}

fn main() {}
```

Please note that [`catch_unwind`] will only catch unwinding panics, not
those who abort the process. See the documentation of [`catch_unwind`]
for more information.

[`catch_unwind`]: ../std/panic/fn.catch_unwind.html

# Representing opaque structs

Sometimes, a C library wants to provide a pointer to something, but not let you
know the internal details of the thing it wants. The simplest way is to use a
`void *` argument:

```c
void foo(void *arg);
void bar(void *arg);
```

We can represent this in Rust with the `c_void` type:

```rust,ignore
extern crate libc;

extern "C" {
    pub fn foo(arg: *mut libc::c_void);
    pub fn bar(arg: *mut libc::c_void);
}
# fn main() {}
```

This is a perfectly valid way of handling the situation. However, we can do a bit
better. To solve this, some C libraries will instead create a `struct`, where
the details and memory layout of the struct are private. This gives some amount
of type safety. These structures are called ‘opaque’. Here’s an example, in C:

```c
struct Foo; /* Foo is a structure, but its contents are not part of the public interface */
struct Bar;
void foo(struct Foo *arg);
void bar(struct Bar *arg);
```

To do this in Rust, let’s create our own opaque types:

```rust
#[repr(C)] pub struct Foo { _private: [u8; 0] }
#[repr(C)] pub struct Bar { _private: [u8; 0] }

extern "C" {
    pub fn foo(arg: *mut Foo);
    pub fn bar(arg: *mut Bar);
}
# fn main() {}
```

By including a private field and no constructor, 
we create an opaque type that we can't instantiate outside of this module.
(A struct with no field could be instantiated by anyone.)
We also want to use this type in FFI, so we have to add `#[repr(C)]`.
And to avoid warning around using `()` in FFI, we instead use an empty array,
which works just as well as an empty type but is FFI-compatible.

But because our `Foo` and `Bar` types are
different, we’ll get type safety between the two of them, so we cannot
accidentally pass a pointer to `Foo` to `bar()`.

Notice that it is a really bad idea to use an empty enum as FFI type.
The compiler relies on empty enums being uninhabited, so handling values of type
`&Empty` is a huge footgun and can lead to buggy program behavior (by triggering
undefined behavior).
