# アウトプット

## "Hello World" の出力

```rust
println!("Hello World");
```

まあ、簡単でしたね。
では、次のトピックに移りましょう。

## `println!` マクロを使う

`println!` マクロを使えば、
かなり好きなように表示することができます。
このマクロは非常に素晴らしい機能を持っていますが、
特殊な構文も持っています。
このマクロは、最初のパラメータとして、
プレースホルダーを含む文字列リテラルを書くことを想定しており、
プレースホルダーはその後に続くパラメータの値で埋められます。

例えば、こんな感じです：

```rust
let x = 42;
println!("My lucky number is {}.", x);
```

これは、このように表示されます。

```console
My lucky number is 42.
```

上の文字列の中括弧（ `{}` ）は、プレースホルダーの一つで、
与えられた値を人間が読めるように表示しようとするデフォルトのプレースホルダー型です。
数値や文字列の場合、これは非常にうまく機能しますが、
すべての型がそうできるわけではありません。
このため、プレースホルダーの中括弧を
`{:?}` のように埋めることで得られる「デバッグ表現」も用意されています。

例えば、こんな感じです。

```rust
let xs = vec![1, 2, 3];
println!("The list is: {:?}", xs);
```

これは、このように表示されます。

```console
The list is: [1, 2, 3]
```

デバッグやログ用に独自のデータ型を出力したい場合は、
ほとんどの場合、その定義の上に `#[derive(Debug)]` を追加することができます。

<aside>

**注：**
"ユーザーフレンドリー "な出力は [`Display`] トレイトを使用し、
デバッグ出力（人間が読めるが開発者向け）は [`Debug`] トレイトを使用します。
`println!` で使用できる構文については、
[std::fmt モジュールのドキュメント][std::fmt]でより多くの情報を得ることができます。

[`Display`]: https://doc.rust-lang.org/1.39.0/std/fmt/trait.Display.html
[`Debug`]: https://doc.rust-lang.org/1.39.0/std/fmt/trait.Debug.html
[std::fmt]: https://doc.rust-lang.org/1.39.0/std/fmt/index.html

</aside>

## エラーの出力

エラーの出力は、
ユーザーや他のツールがファイルや他のツールに出力をパイプするのを容易にするために、
`stderr` （標準エラー出力）を介して行われるべきです。

<aside>

**注：**
ほとんどのOSでは、
プログラムは `stdout` と `stderr` という2つの出力ストリームに書き込むことができます。
`stdout` はプログラムの実際の出力用で、
`stderr` はエラーやその他のメッセージを `stdout` とは別に保持することができます。
これにより、
エラーはユーザーに表示しながら、出力をファイルに保存したり、
他のプログラムへパイプで送ったりすることができます。

</aside>

Rustでは、`println!` と `eprintln!` でこれを実現し、
前者は `stdout` に、
後者は `stderr` に出力します。

```rust
println!("This is information");
eprintln!("This is an error! :(");
```

<aside>

**注意点：** [エスケープコード] を出力すると、
ユーザーのターミナル画面が変な状態になるなど、危険なことがあります。
手動で出力する場合は、常に注意してください。

[エスケープコード]: https://en.wikipedia.org/wiki/ANSI_escape_code

理想的には、あなた（とあなたのユーザー）の生活を楽にするために、
生のエスケープコードを扱うときは `ansi_term` のようなクレートを使用すべきです。

</aside>

## 出力パフォーマンスに関する注意点

ターミナル画面への出力は意外と遅い！
`println!` などをループで呼び出すと、
せっかく高速なプログラムでもボトルネックになりがちです。
これを高速化するために、
2つのことができます。

まず、
実際にターミナル画面を「書き換える」書き込みの数を減らした方がいいかもしれません。
`println!` は、毎回ターミナル画面を書き換えるようにシステムに指示します。
もしその必要がなければ、`stdout` ハンドルを [`BufWriter`] でラップすることができ、
デフォルトで 8 kB までバッファリングできます。
(すぐに印刷したい場合は、この `BufWriter` に対して `.flush()` を呼び出すことができます)。

```rust
use std::io::{self, Write};

let stdout = io::stdout(); // グローバル stdout エンティティを取得
let mut handle = io::BufWriter::new(stdout); // オプション： そのハンドルをバッファに格納
writeln!(handle, "foo: {}", 42); // エラーを気にする場合は、`?` を追加
```

次に、
`stdout` (または `stderr` )をロックし，
`writeln!` を使って `stdout` に直接出力することが有効です。
これにより、システムが何度も `stdout` をロックしたりアンロックしたりするのを防ぐことができます。

```rust
use std::io::{self, Write};

let stdout = io::stdout(); // グローバル stdout エンティティを取得
let mut handle = stdout.lock(); // そのハンドルをロックする
writeln!(handle, "foo: {}", 42); // エラーを気にする場合は、`?` を追加
```

また、この2つのアプローチを組み合わせることも可能です。

[`BufWriter`]: https://doc.rust-lang.org/1.39.0/std/io/struct.BufWriter.html

## プログレスバーの表示

CLIアプリケーションの中には、1秒以内に動作するものもあれば、
数分、数時間かかるものもあります。
もしあなたが後者のタイプのプログラムを書いているならば、
何かが起こっていることをユーザーに示したいと思うかもしれません。
そのためには、有用なステータス更新状況を表示するようにすべきですが、
それらは簡単に表示できる形式であることが理想的です。

[indicatif] クレートを使うと、
プログレスバーや小さなスピナーをプログラムに追加することができます。
以下に簡単な例を示します：

```rust,ignore
{{#include output-progressbar.rs:1:9}}
```

詳しくは、
[ドキュメント][indicatif docs]
と[サンプル][indicatif examples]
をご覧ください。

[indicatif]: https://crates.io/crates/indicatif
[indicatif docs]: https://docs.rs/indicatif
[indicatif examples]: https://github.com/console-rs/indicatif/tree/main/examples

## ロギング

プログラムで何が起こっているのかを理解しやすくするために、
ログ文を追加したいと思うかもしれません。
これは通常、アプリケーションを書いているときには簡単なことです。
しかし、半年後にこのプログラムを再び実行するときには、とても役に立つでしょう。
ある意味では、
ロギングは `println!` を使うのと同じですが、
メッセージの重要度を指定することができます。
通常使用できるレベルは、*error* 、*warn* 、*info* 、*debug* 、*trace* です
（ *error* が最も優先度が高く、*trace* が最も低い）。

アプリケーションに簡単なロギングを追加するには、
[log] クレート（ログレベルにちなんだ名前のマクロを含む）と、
実際にログ出力を有用な場所に書き出す *アダプタ* の2つが必要です。
ログアダプタを使用するための機能は、非常に柔軟です：
例えば、ターミナルだけでなく、[syslog] や中央ログサーバーにも
ログを書き込むことができます。

[log]: https://crates.io/crates/log
[syslog]: https://en.wikipedia.org/wiki/Syslog

今はCLIアプリケーションを書くことしか考えていないので、
使いやすいアダプタは [env_logger] になります。
"env" ロガーと呼ばれるのは、
環境変数を使ってアプリケーションのどの部分をログに記録するか
（どのレベルで記録するか）を指定できるからです。
また、ログメッセージの先頭にタイムスタンプとログメッセージの送信元モジュールを付加します。
ライブラリも `log` を使うことができるので、
そのログ出力も簡単に設定できます。

[env_logger]: https://crates.io/crates/env_logger

簡単な例を挙げます：

```rust,ignore
{{#include output-log.rs}}
```

このファイルを `src/bin/output-log.rs` とすると、
Linux と macOS では次のように実行できます：
```console
$ env RUST_LOG=info cargo run --bin output-log
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
     Running `target/debug/output-log`
[2018-11-30T20:25:52Z INFO  output_log] starting up
[2018-11-30T20:25:52Z WARN  output_log] oops, nothing implemented!
```

Windows PowerShell では、このように実行できます：
```console
$ $env:RUST_LOG="info"
$ cargo run --bin output-log
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
     Running `target/debug/output-log.exe`
[2018-11-30T20:25:52Z INFO  output_log] starting up
[2018-11-30T20:25:52Z WARN  output_log] oops, nothing implemented!
```

Windows のコマンドプロンプトでは、次のように実行できます：
```console
$ set RUST_LOG=info
$ cargo run --bin output-log
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s
     Running `target/debug/output-log.exe`
[2018-11-30T20:25:52Z INFO  output_log] starting up
[2018-11-30T20:25:52Z WARN  output_log] oops, nothing implemented!
```

`RUST_LOG` は、ログ設定を行うために使用できる環境変数の名前です。
`env_logger` にはビルダーも含まれているので、
プログラムでこれらの設定を調整することができ、
例えば、*info* レベルのメッセージをデフォルトで表示することもできます。

ロギング・アダプターの代替品はたくさんあります、
そしてまた、`log` の代替や拡張もあります。
もし、あなたのアプリケーションが多くのログを記録することが分かっているのであれば、
必ずそれらをレビューし、
ユーザーの生活を楽にするようにしてください。

<aside>

**ヒント：**
経験上、多少便利なCLIプログラムであっても、何年も使い続けられることになります。
(一時的な解決策のつもりだった場合は特に。) 
アプリケーションが動作せず、
そして誰か（例えば、将来のあなた）がその原因を突き止める必要がある場合、
`--verbose` オプションを渡して追加のログ出力を得ることができれば、
デバッグにかかる時間を数時間から数分に短縮することができます。
[clap-verbosity-flag] クレートには、
`clap` を使ってプロジェクトに `--verbose` を追加する簡単な方法が含まれています。

[clap-verbosity-flag]: https://crates.io/crates/clap-verbosity-flag

</aside>
