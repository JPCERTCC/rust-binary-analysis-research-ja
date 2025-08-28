# CargoのProfile設定の変更に伴うバイナリの差分

攻撃者は実行ファイルのサイズ削減や情報の隠蔽を目的として、CargoのProfile設定を変更してビルドを行う可能性がある。そのため、本調査ではCargoのProfileで設定可能な各オプションを個別に変更した場合のファイルサイズを測定し、最も効果的にサイズを削減できるProfile設定を特定した。

さらに、各オプション単独で変更したバイナリから特有の情報やアセンブリコードの特徴を抽出し、その差異について調査を実施した。

## 調査結果

最も実行ファイルのサイズを削減できるProfileのオプションは下記のとおりである。
以降、各オプションの調査結果の詳細を記載する。

```toml
[profile.release]
opt-level = "s"
debug-assertions = false
overflow-checks = false
lto = "fat"
panic = "abort"
codegen-unit = 1
```

## 詳細

### opt-level

最適化の度合いを決めるオプションである。
サイズ削減を目的とした最適化を行うオプション`s`を指定することで、サイズが削減される。
また、最も最適化を行わないオプションであるオプション`0`を指定した場合とバイナリを比較すると下記のような差分が確認できた。

* ループ方法において`opt-level=0`ではイテレータが使用されるが`opt-level="s"`ではdec命令およびjnz命令を使用している。
* 余剰の計算方法において`opt-level=0`ではdiv命令が使用されるが`opt-level="s"`ではmul、shr、and命令を使用して表現している。
* `opt-level="s"`ではスタックを用いた無駄なデータ転送が削除されている。

### split-debuginfo

デバッグ情報をどこへ配置するかを決めるオプションである。
`split-debuginfo=off`ではデバッグ情報は実行ファイルに埋め込まれ、`split-debuginfo=packed`ではpdbファイルやdwpファイルとして実行ファイルから分離される。
Windows-MSVC環境ではpackedのみが有効となっているため、実行ファイルサイズや実行ファイルから得られる情報に影響しない。

### debug

実行ファイルまたはpdbファイルに埋め込まれるデバッグ情報の量を制限するオプションである。
Windows-MSVC環境では、実行ファイルにデバッグ情報を埋め込むことはできない。
そのため、debugオプションはWindows-MSVC環境においては実行ファイルに影響を与えず、pdbファイルのサイズにのみ影響する。

### strip

シンボルやデバッグ情報をstripするか否かを決定するオプションである。
Windows-MSVC環境では、実行ファイルにデバッグ情報を埋め込むことはできない。
そのため、本オプションは実行ファイルのサイズには影響を与えず、pdbファイルのサイズにのみ影響する。

### debug-assertions

debug_assert!()マクロの使用可否を決定するオプションである。
debug-assertionsにおいて最もサイズを削減できる設定値は`false`である。
設定値ごとのアセンブリの差分を調査した結果、`false`のバイナリではアサートを発生させるアセンブリが生成されない。

また、`true`のバイナリにのみ下記の文字列が確認できた。
文字列`unsafe precondition(s) violated:`は`assert_unsafe_precondition`マクロ関数に含まれる文字列であり、Rust特有の文字列である。
なお、本文字列は`Rust 1.78.0`以降の場合にのみ確認出来た。 
```
unsafe precondition(s) violated: usize::unchecked_mul cannot overflow

unsafe precondition(s) violated: ptr::sub_ptr requires `self >= origin`

unsafe precondition(s) violated: hint::assert_unchecked must never be called when the condition is false

unsafe precondition(s) violated: Layout::from_size_align_unchecked requires that align is a power of 2 and the rounded-up allocation size does not exceed isize::MAX

unsafe precondition(s) violated: slice::from_raw_parts requires the pointer to be aligned and non-null, and the total size of the slice not to exceed `isize::MAX`
```

これらの文字列をシグネチャとして`debug-assertions`オプションが`true`か否かを判別するYARAルールは以下のとおりである。
なお、これらの文字列は、Cargoによってデフォルトで生成されるHello Worldプログラムでは、`debug-assertions`が`true`であるにもかかわらず確認できなかった。
その理由として、これらの文字列は`unsafe`ブロック内で使用されるため、標準ライブラリを含むプログラム内で`unsafe`ブロックが使用されている場合にのみ現れると予測する。

```yara
rule Detect_DebugAssertions_Is_True
{
    strings:
        $s1 = "unsafe precondition(s) violated:"

    condition:
        Rust_Binary and $s1
}
```

### overflow-checks

整数オーバーフローのチェックを行うか否かを決定するオプションである。
`true`または`false`を指定できるが、両オプションにて実行ファイルのサイズは同じとなった。
これは`true`の際に追加されたアセンブリのサイズが`.text`セクションのアラインメントをまたがなかったことに起因すると考えられる。
本オプションで`.text`セクションのみ比較した場合は、わずかではあるが`false`のサイズの方が小さくなる。

設定値ごとのアセンブリの差分を調査した結果、`true`のバイナリの場合、演算後にJB命令（carry = 1）を用いたオーバーフローチェックとパニックを発生させるアセンブリが生成された。
一方、`false`のバイナリの場合、オーバーフローチェックやパニック処理は確認できなかった。

また、`true`のバイナリにのみ下記の文字列が確認できた。
これらの文字列は`core/num/overflow_panic.rs`に含まれるオーバーフローによってパニックが発生した際に使用されるパニックメッセージである。

```
attempt to add with overflow

attempt to subtract with overflow

attempt to multiply with overflow

attempt to divide with overflow

attempt to calculate the remainder with overflow

attempt to negate with overflow

attempt to shift right with overflow

attempt to shift left with overflow
```

これらの文字列をシグネチャとして、`overflow-checks`オプションが`true`か否かを判別するYARAルールは、以下のとおりである。

```yara
rule Detect_OverflowCheck_Is_True
{
    strings:
        $s1 = "attempt to add with overflow"
        $s2 = "attempt to subtract with overflow"
        $s3 = "attempt to multiply with overflow"
        $s4 = "attempt to divide with overflow"
        $s5 = "attempt to calculate the remainder with overflow"
        $s6 = "attempt to negate with overflow"
        $s7 = "attempt to shift right with overflow"
        $s8 = "attempt to shift left with overflow"

    condition:
        Rust_Binary and any of them
}
```

### lto

LLVMリンク時の最適化を制御するオプションである。
実行ファイルのサイズを削減できる設定値は`fat`である。

なお、`lto`オプションを`fat`に設定したバイナリを調査した結果、以下のような特徴が抽出された。

* `lang_start_internal()`や`_print()`とその内部で呼び出される関数のインライン化
* ユーザー定義main関数の呼び出しフローの簡略化

### panic

パニック発生時の挙動を決定するオプションである。
実行ファイルのサイズを削減できる設定値は`abort`である。
また、バイナリの差分としては、`Exception Directory`においてユーザー定義ではないmain関数の`Unwind Data`が削除されていた。
これはWindowsにおいて例外時にスタックトレースを取得するための情報がないことを示している。

設定値が`unwind`のバイナリにのみ、下記の文字列が確認できた。
```
fatal runtime error: Rust panics must be rethrown

fatal runtime error: Rust cannot catch foreign exceptions

Rust panics cannot be copied

_CxxThrowException
```

これらはパニック関連の文字列であり、内容は以下のとおりである。

* `fatal runtime error:`を含む文字列は、`unwind`時に発生するパニックメッセージであり、これらの文字列は`Rust 1.71.0`以降にのみ確認出来た。
* `Rust panics cannot be copied`はWindowsのSEH関連の処理を行う`define_cleanup`マクロ関数に含まれるパニックメッセージ
* `_CxxThrowException`は最終的に例外を発生させるのに使用されるWindows APIであり、`unwind`バイナリが`_CxxThrowException`を呼び出して例外を発生させているのに対して、`abort`バイナリでは`RtlFailFast(int 29H)`というシステムコール命令を用いてプログラムを終了させている。

これらの文字列をシグネチャとして、`panic`オプションが`abort`か否かを判別するYARAルールは、以下のとおりである。

```yara
rule Detect_Panic_Is_Abort
{
    metadata:
        description = "Detect Rust binary with option panic abort"
        author = "JPCERT/CC Incident Response Group"

    strings:
        $s1 = "fatal runtime error: Rust panics must be rethrown"
        $s2 = "fatal runtime error: Rust cannot catch foreign exceptions"
        $s3 = "Rust panics cannot be copied"
        $s4 = "_CxxThrowException"

    condition:
        Rust_Binary and none of them
}
```

### incremental

本オプションを有効にした場合、再コンパイル時に使用される情報を`target`ディレクトリに保存する設定であり、実行ファイルのサイズや生成されるアセンブリに影響を及ぼさないオプションである。
`true`の場合、プロジェクトフォルダーの`target\release\incremental`直下に再利用されるデータが生成される。

### codegen-units

クレートをいくつに分割してコンパイル処理を行うかを指定するオプションである。
分割数が多くなると、その分並列処理できるため、コンパイル時間の短縮につながるが生成されるコードが遅いコードとなる可能性がある。
codegen-unitsの値を`1`に設定することでバイナリサイズが削減できる。

### rpath

実行時に動的ライブラリを検索するパスを指定するためのオプションである。
これにより柔軟なライブラリ配置、カスタムディレクトリの使用が可能となる。
実行ファイルのサイズに影響を与えるオプションではない。
