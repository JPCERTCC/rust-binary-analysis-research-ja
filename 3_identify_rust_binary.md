# Rustバイナリの識別

Rustバイナリの識別を目的に、リリースビルドおよび最小化ビルドのRustバイナリ特有の文字列やバイト列などを調査した。

## 調査結果

調査の結果、複数の特徴的な文字列とその文字列を使用する`core::panic::Location`構造体を組み合わせることで、YARAルールでRustの最小化バイナリの検出が一部可能であることを確認した。

## 詳細

[CargoのProfile設定の変更に伴うバイナリの差分](1_profile.md)、[バイナリサイズ削減](2_minimize_binary.md)および[main関数の特定と初期化処理](6_identify_main_function.md)の調査結果から、Rustバイナリの判定を行うYARAルールを作成した。

```yara
private rule PE_Signature
{
    condition:
        uint16(0) == 0x5A4D 
}

private rule Plain_Rust_Binary
{
    metadata:
        description = "Detect plain Rust binary"
        author = "JPCERT/CC Incident Response Group"

    strings:
        $s1 = "run with `RUST_BACKTRACE=1` environment variable to display a backtrace"
        $s2 = "called `Result::unwrap()` on an `Err` value"
        $s3 = "called `Option::unwrap()` on a `None` value"
    
    condition:
        PE_Signature and all of them
}

private rule Minsized_Rust_Binary
{
    metadata:
        description = "Detect minsized Rust binary"
        author = "JPCERT/CC Incident Response Group"

    strings:
        $s1 = "<unknown>"
        $s2 = "<redacted>"
        $s3 = "failed to write whole buffer"
        $s4 = "failed to write the buffered data"
        $c1_64 = {1C 00 00 00 00 00 00 00 17 00 00 00 00 00 00 00} 
        $c1_32 = {1C 00 00 00 17 00 00 00} 
        $c2_64 = {21 00 00 00 00 00 00 00 17 00 00 00 00 00 00 00} 
        $c2_32 = {21 00 00 00 17 00 00 00} 
    
    condition:
        PE_Signature and 3 of ($s*) and 1 of ($c1*) and 1 of ($c2*) 
}

rule Rust_Binary
{
    metadata:
        description = "Detect Rust binary"
        author = "JPCERT/CC Incident Response Group"

    condition:
        Plain_Rust_Binary or Minsized_Rust_Binary
}
```