# プロジェクト設定

まだインストールしていない場合は、
コンピュータに[Rustをインストール]します
（数分しかかかりません）。
その後、ターミナルを開き、
アプリケーションコードを格納したいディレクトリに移動します。

[Rustをインストール]: https://www.rust-lang.org/tools/install

プログラミングプロジェクトを保存するディレクトリで、
`cargo new grrs`
を実行することから始めます。
新しく作成された `grrs` ディレクトリを見ると、
Rustプロジェクトの典型的なセットアップが表示されます：

- `Cargo.toml` ：プロジェクトのメタデータを含むファイル
（使用する依存関係/外部ライブラリのリストを含む）。
- `src/main.rs` ：（メイン）バイナリのエントリポイントとなるファイル。

`grrs` ディレクトリで `cargo run` を実行して、
"Hello World "が表示されれば準備完了です。

## 実行例

```console
$ cargo new grrs
     Created binary (application) `grrs` package
$ cd grrs/
$ cargo run
   Compiling grrs v0.1.0 (/Users/pascal/code/grrs)
    Finished dev [unoptimized + debuginfo] target(s) in 0.70s
     Running `target/debug/grrs`
Hello, world!
```