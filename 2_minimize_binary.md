# バイナリサイズ削減

攻撃者は痕跡を隠す目的で、バイナリサイズの削減を図る可能性がある。そのため、[公開情報](https://github.com/johnthagen/min-sized-rust)で紹介されているバイナリサイズ削減手法について、どの程度サイズを削減可能なのか、また削減後も消去されずに残存する情報からどのような情報を得られるのかを調査した。

## 調査結果

バイナリサイズを最小化する際に使用するビルドコマンドは以下のとおりである。

```
$env:RUSTFLAGS="-Zfmt-debug=none -Zlocation-detail=none";cargo +nightly build -Z build-std=std,panic_abort -Z build-std-features="optimize_for_size" -Z build-std-features=panic_immediate_abort --target x86_64-pc-windows-msvc --release
```

## 詳細

### Remove Location Details

`location-detail`オプションは、バックトレース情報をコントロールすることができるビルドオプションである。
本ビルドオプションを`none`に指定することで、パニック時に使用される`core::panic::Location`構造体におけるソースコードパスを示す値が`<redacted>`に置き換えられるとともに、ソースコードの行と列を示す値も0に置き換えられる。
`location-detail`オプションを`none`に指定することで、ソースコードパスが削除されるため、バイナリサイズの削減が可能である。
加えて、当該情報を削除することでバックトレースが正常に出力されなくなるほか、ライブラリの識別に関する調査に影響すると推測される。
`<redacted>`と`<redacted>`を含む`core::panic::Location`構造体をシグネチャとして、`-Zlocation-detail=none`が用いられているか否かを判別するYARAルールを下記に示す。

```yara
rule Detect_LocationDetail_Is_None
{
    metadata:
        description = "Detect Rust binary with option remove location details"
        author = "JPCERT/CC Incident Response Group"

    strings:
        $s1 = "<redacted>"
        /*
            pub struct Location<'a> {
                file: &'a str,
                line: u32,
                col:  u32,
            }
        */
        $c1 = {
            0A 00 00 00 00 00 00 00
            00 00 00 00
            00 00 00 00
        }
        $c2 = {
            0A 00 00 00 
            00 00 00 00
            00 00 00 00
        }

    condition:
        Rust_Binary and $s1 and ($c1 or $c2)
}
```

### Remove fmt::Debug

`fmt-debug`オプションを用いると、`#[derive(Debug)]`および`{:?}`の出力を操作することができる。
本ビルドオプションを`-Zfmt-debug=none`に設定すると、内部で`#[derive(Debug)]`および`{:?}`を使用している`dbg!()`、`assert!()`、`unwrap()`などのマクロの出力が一部削除される。
そのため、`-Zfmt-debug=none`を使用することでバイナリサイズの削減が可能である。

### Optimize libstd with build-std

バイナリにリンクされる標準ライブラリはデフォルトの状態であっても速度面で最適化されているが、サイズ面では最適化されていない。
本ビルドオプション`-Z build-std=std,panic_abort -Z build-std-features="optimize_for_size"`で標準ライブラリを指定した上で、サイズ面で最適化した状態に再ビルドすることが可能である。
サイズ面で最適化したバイナリをリンクした場合、実行ファイルのサイズを削減できる。

### Remove panic String Formatting with panic_immediate_abort

`Cargo`の`Profile`設定で指定可能な`panic="abort"`は標準ライブラリには適用されない。
ビルドオプション`-Z build-std-features=panic_immediate_abort`を使用することで、標準ライブラリにも`panic="abort"`を適用することができる。
本ビルドオプションを使用することで、バイナリのサイズを削減できる。

また、`panic_immediate_abort`を指定したバイナリからは、[Rustバイナリの識別](3_identify_rust_binary.md)で解説した特徴的な文字列が削除されているため、Rustバイナリとして認識できなくなる可能性がある。
加えて、パニックメッセージが削除されることから、使用しているライブラリの識別に影響すると考えられる。