# コマンドライン引数のパース

CLIツールの典型的な呼び出しは次のようなものです：

```console
$ grrs foobar test.txt
```

このプログラムは、`test.txt` を見て、
`foobar` を含む行をプリントアウトすることを期待しています。
しかし、この2つの値はどのように取得するのでしょうか？

プログラム名の後のテキストは、しばしば
「コマンドライン引数」または
「コマンドラインフラグ」
（特に `--this` のように見える場合）
と呼ばれます。
大まかに言えば、それらはスペースで区切られているので、
内部的には、オペレーティングシステムは通常、
これらを文字列のリストとして表します。

このような引数について、
どのように解析して、
より作業しやすいものにするかは、いろいろな考え方があります。
また、プログラムの利用者に、
どのような引数をどのような形式で与える必要があるかを伝える必要があります。

## 引数の取得

標準ライブラリには、与えられた引数の[イテレータ]を取得する関数
[`std::env::args()`]が含まれています。
最初のエントリ(インデックス `0` )は、あなたのプログラムが呼び出されたときの名前（例： `grrs` ）で、
その後はユーザーがその後に書いたものが続きます。

[`std::env::args()`]: https://doc.rust-lang.org/1.39.0/std/env/fn.args.html
[イテレータ]: https://doc.rust-lang.org/1.39.0/std/iter/index.html

この方法で引数を取得するのは非常に簡単です（ファイル `src/main.rs` の `fn main() {` の後に以下を記述）：

```rust,ignore
{{#include cli-args-struct.rs:10:11}}
```

## データ型としてのCLI引数

CLI引数は、テキストの束として考えるのではなく、
プログラムへの入力を表すカスタムデータ型と考えるのが得策です。

`grrs foobar test.txt` を見てください。
引数は2つで、
最初に `pattern`（探す文字列）、
次に `path`（探すファイル）となっています。

これ以上、何を語ればいいのでしょうか。
まず第一に、これらは両方とも必須であるということです。
デフォルトの値については何も話していないので、
ユーザーは常に2つの値を提供することが期待されています。
さらに、それらの型についても少し述べることができます：
patternは文字列、
2番目の引数はファイルへのパスです。

Rustでは、扱うデータを中心にプログラムを構成するのが一般的なので、
このCLI引数の見方はとてもしっくりきます。まず、こんなところから始めてみましょう
（ファイル `src/main.rs` の、`fn main() {` の前に以下を記述）：

```rust,ignore
{{#include cli-args-struct.rs:3:7}}
```

これは、データを格納するための2つのフィールド（`pattern` と `path`）を持つ
新しい構造体（[`struct`]）を定義するものです。

[`struct`]: https://doc.rust-lang.org/1.39.0/book/ch05-00-structs.html

<aside>

**注：**
[`PathBuf`]は[`String`]のようなものですが、クロスプラットフォームで動作するファイルシステム・パス用のデータ型です。

[`PathBuf`]: https://doc.rust-lang.org/1.39.0/std/path/struct.PathBuf.html
[`String`]: https://doc.rust-lang.org/1.39.0/std/string/struct.String.html

</aside>

さて、私たちのプログラムが得た実際の引数をこのような形に変換する必要があります。
選択肢の一つは、オペレーティングシステムから得た文字列のリストを手動で解析し、
自分自身で構造を構築することです。
それは以下のようなものになります：

```rust,ignore
{{#include cli-args-struct.rs:10:15}}
```

これは動作しますが、あまり便利ではありません。
`--pattern="foo"` や `--pattern "foo"` をサポートする必要がある場合、
どのように対処するのでしょうか？
`--help` はどのように実装するのでしょうか？

## Clapを使ったCLI引数のパース

もっといい方法は、利用可能な多くのライブラリのうちの一つを使うことです。
コマンドライン引数をパースするための最も一般的なライブラリは
[`clap`]と呼ばれるものです。
サブコマンドのサポート、[シェルの補完]、優れたヘルプメッセージなど、
期待されるすべての機能を備えています。

[`clap`]: https://docs.rs/clap/
[シェルの補完]: https://docs.rs/clap_complete/

まず、`Cargo.toml` ファイルの `[dependencies]` セクションに
`clap = { version = "4.0", features = ["derive"] }` を追加して、
`clap` をインポートしましょう。

これで、コードに `use clap::Parser;` と書いて、
`struct Cli` のすぐ上に `#[derive(Parser)]` を追加することができるようになりました。
ついでにドキュメントのコメントも書いておきましょう。

次のようになります（ファイル `src/main.rs` の `fn main() {` の前に以下を記述）：

```rust,ignore
{{#include cli-args-clap.rs:3:12}}
```

<aside class="node">

**注：**
フィールドに追加できるカスタム属性はたくさんあります。
例えば、
`-o` や `--output` の後の引数にこのフィールドを使いたい場合は、
`#[arg(short = 'o', long = "output")]` と追加します。
詳しくは、[clapのドキュメント][`clap`]をご覧ください。

</aside>

`Cli` 構造体のすぐ下に、このテンプレートの `main` 関数があります。
プログラムが開始されると、この関数が呼び出されます。
最初の行は：

```rust,ignore
{{#include cli-args-clap.rs:14:16}}
```

これは、`Cli` 構造体に引数をパースしようとするものです。

しかし、それが失敗したらどうするか？
それがこの方法の良さです：
Clapは、どのフィールドが必要で、
どのような形式が必要なのかを知っています。
また、`--help` メッセージを自動的に生成したり、
間違えて `--putput` と書いたときに `--output` を渡すよう示唆するような
素晴らしいエラーも出力してくれます。

<aside class="note">

**注：**
`parse` メソッドは、`main` 関数で使用することを想定しています。
失敗した場合は、
エラーメッセージやヘルプメッセージを出力し、直ちにプログラムを終了します。
他の場所で使わないでください！

</aside>

## まとめ

これで、コードは次のようになります：

```rust,ignore
{{#include cli-args-clap.rs}}
```

引数なしで実行するとこうなります：

```console
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 10.16s
     Running `target/debug/grrs`
error: The following required arguments were not provided:
    <pattern>
    <path>

USAGE:
    grrs <pattern> <path>

For more information try --help
```

`cargo run` を直接使う場合は、`--` の後に引数を書けば、引数を渡すことができます：

```console
$ cargo run -- some-pattern some-file
    Finished dev [unoptimized + debuginfo] target(s) in 0.11s
     Running `target/debug/grrs some-pattern some-file`
```

ご覧の通り、
出力はありません。
これは良いことです：
これは、エラーがなく、プログラムが終了したことを意味します。

<aside class="exercise">

**練習問題：**
このプログラムに引数を出力させろ！

</aside>