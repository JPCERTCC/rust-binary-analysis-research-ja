# TLS Directory

`TLS Directory`について、Microsoftが定める`TLS Directory`に準拠する構造となっているか否か、またデフォルトで存在している`TLS Callback`の処理内容を
明確にすることを目的として調査した。

## 調査結果

調査の結果、Rust製バイナリであってもMicrosoftが定める`TLS Directory`の構造と一致していることが判明した。
なお、`TLS Directory`の構造は公式サイトで説明されている。

https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#the-tls-section

加えて、デフォルトで存在している`TLS Callback`はスレッドまたはプロセスのデタッチ時にTLS変数のドロップ等のクリーンアップ処理を実行するために存在している。

## 詳細

省略