# *grrs* の最初の実装

「[コマンドライン引数のパース]」の章を終えて、
入力データが揃ったので、
実際のツールを書き始めることができるようになりました。
私たちの `main` 関数には、今はこの行しかありません。

[コマンドライン引数のパース]:./cli-args.html

```rust,ignore
{{#include impl-draft.rs:15:15}}
```

まず、入手したファイルを開いてみましょう。

```rust,ignore
{{#include impl-draft.rs:16:16}}
```

<aside>

**注：**
ここにある [`.expect`] メソッドを見てください。
これは、値（この場合は入力ファイル）を読み込めなかったときに、
プログラムを即座に終了させる `quit` のショートカット関数です。
これはあまりきれいなものではありません。
次の「[より良いエラー報告]」の章では、
これを改善する方法について見ていきます。

[`.expect`]: https://doc.rust-lang.org/1.39.0/std/result/enum.Result.html#method.expect
[より良いエラー報告]:./errors.html

</aside>

では、行を繰り返し、
パターンを含む行をそれぞれ表示してみましょう。

```rust,ignore
{{#include impl-draft.rs:18:22}}
```

## まとめ

これで、コードは次のようになります：

```rust,ignore
{{#include impl-draft.rs}}
```

試してみましょう： `cargo run -- main src/main.rs` はこれで動くはずです。

<aside class="exercise">

**練習問題：**
これはベストな実装とは言えません：
どんなに大きなファイルでも、
ファイル全体をメモリに読み込みます。
最適化する方法を見つけてください！
（一つのアイデアとして、 `read_to_string()` の代わりに
[`BufReader`]を使用することもできます）

[`BufReader`]: https://doc.rust-lang.org/1.39.0/std/io/struct.BufReader.html

</aside>
