# より良いエラー報告

私たちは、エラーが起こるという事実を受け入れるしかないのです。
そして、他の多くの言語とは対照的に、
Rustを使うときにこの現実に気づき、
対処するのは非常に難しいことです：
Rustには例外がないため、
起こりうるすべてのエラー状態が、関数の戻り値の型にエンコードされていることが多いのです。

## Result型

[`read_to_string`]のような関数は、文字列を返しません。
その代わり、`String`か何らかの型のエラー
（この場合は [`std::io::Error`] ）
を含む [`Result`] 型を返します。

[`read_to_string`]: https://doc.rust-lang.org/1.39.0/std/fs/fn.read_to_string.html
[`Result`]: https://doc.rust-lang.org/1.39.0/std/result/index.html
[`std::io::Error`]: https://doc.rust-lang.org/1.39.0/std/io/type.Result.html

どのようにそれがわかるのですか？
`Result` 型は `enum` 型（列挙型）なので、
`match` を使ってどの列挙子かを確認することができます：

```rust,no_run
let result = std::fs::read_to_string("test.txt");
match result {
    Ok(content) => { println!("File content: {}", content); }
    Err(error) => { println!("Oh noes: {}", error); }
}
```

<aside>

**注：**
Enumとは何か、Rustでどのように動作するのか、よくわからない？
[Rust bookのこの章](https://doc.rust-jp.rs/book-ja/ch06-00-enums.html)を読んで、
スピードアップしてください。

</aside>

## アンラッピング

さて、ファイルの内容にアクセスすることはできましたが、
`match` ブロックの後では実際に何もすることができません。
このため、エラーケースをどうにかして処理する必要があります。
課題は、`match` ブロックのすべての分岐が同じ型のものを返す必要があることです。
しかし、それを回避するための巧妙なトリックがあります：

```rust,no_run
let result = std::fs::read_to_string("test.txt");
let content = match result {
    Ok(content) => { content },
    Err(error) => { panic!("Can't deal with {}, just exit here", error); }
};
println!("file content: {}", content);
```

この例ではmatchブロックの後の `content` でStringを使用することができます。
もし `result` がエラーだったら、Stringは存在しないことになります。
しかし、`content` を使うところに到達する前にプログラムが終了してしまうので、問題ないでしょう。

これは思い切ったように見えるかもしれませんが、
とても便利な方法です。
もしあなたのプログラムがそのファイルを読む必要があり、そのファイルが存在しなければ何もできないのであれば、
プログラムの終了は有効な戦略です。
`Results` には、`unwrap` というショートカットメソッドもあります：

```rust,no_run
let content = std::fs::read_to_string("test.txt").unwrap();
```

## panic は不要

もちろん、プログラムを中断することだけがエラーへの対処法ではありません。
`panic!` の代わりに、`return` を書くことも簡単にできます：

```rust,no_run
# fn main() -> Result<(), Box<dyn std::error::Error>> {
let result = std::fs::read_to_string("test.txt");
let content = match result {
    Ok(content) => { content },
    Err(error) => { return Err(error.into()); }
};
# Ok(())
# }
```

しかし、これによって、この関数が必要とする戻り値の型が変わってしまいます。
確かに、この例にはずっと何かが隠されていました：
それは、このコードの関数シグネチャです。
そして、この最後の `return` の例では、
それが重要になります。
以下はその *全例* です：

```rust,no_run
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let result = std::fs::read_to_string("test.txt");
    let content = match result {
        Ok(content) => { content },
        Err(error) => { return Err(error.into()); }
    };
    println!("file content: {}", content);
    Ok(())
}
```

戻り値の型は `Result` です！
このため、2番目のマッチアームで `return Err(error);` と書くことができるのです。
一番下に `Ok(())` があるのがわかりますか？
これは関数のデフォルトの戻り値で、
「ResultはOKで、中身はない」という意味です。

<aside>

**注：**
なぜ `return Ok(());` と書かないのでしょうか？
簡単に書けますし、これも完全に正しいです。
Rustのブロックの最後の式は戻り値であり、
不要な `return` は省略するのが通例です。

</aside>

## クエスチョンマーク

`.unwrap()` を呼ぶことが、
エラー分岐で `panic!` を使った `match` のショートカットであるように、
エラー分岐で `return` を使った `match` のショートカット：`?` もあります。

そうです、クエスチョンマークです。
この演算子を `Result` 型の値に付加すると、
Rustは内部的にこれを今書いた `match` と非常によく似たものに展開します。

試してみてください。

```rust,no_run
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string("test.txt")?;
    println!("file content: {}", content);
    Ok(())
}
```

非常に簡潔です！

<aside>

**注：**
ここでさらにいくつかのことが起こっていますが、
それを理解する必要はありません。
例えば、
私たちの `main` 関数でのエラータイプは `Box<dyn std::error::Error>` です。
しかし、`read_to_string` が [`std::io::Error`] を返すことは上で見たとおりです。
これは、`?`  がエラータイプを *変換* してコードを 展開するため、動作します。

`Box<dyn std::error::Error>` もまた興味深い型です。
これは、標準的な [`Error`][`std::error::Error`] trait を実装する
任意の型を含むことができる `Box` です。
つまり、基本的にすべてのエラーをこのboxに入れることができるので、
`Result` を返す通常の関数のすべてで `?` を使うことができます。

[`std::error::Error`]: https://doc.rust-lang.org/1.39.0/std/error/trait.Error.html

</aside>

## コンテキストの提供

`main` 関数で `?` を使ったときに出るエラーは大丈夫ですが、
あまりいいものではありません。
例えば：
`std::fs::read_to_string("test.txt")?` を実行したが、
ファイル `test.txt` が存在しない場合、
次のような出力が得られます：

```text
Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

コードに文字通りファイル名が含まれていない場合、
どのファイルが `NotFound` であったかを判断するのは非常に困難です。
これに対処する方法は複数あります。

例えば、独自のエラータイプを作成し、
それを使って独自のエラーメッセージを作成することができます：

```rust,ignore
{{#include errors-custom.rs}}
```

さて、
これを実行すると、独自のエラーメッセージが表示されます。

```text
Error: CustomError("Error reading `test.txt`: No such file or directory (os error 2)")
```

あまりきれいではありませんが、
後で私たちの型に合わせたデバッグ出力を簡単に行うことができます。

このパターンは、実はとてもよくあることなのです。
しかし、ひとつ問題があります：
オリジナルのエラーは保存されず、
文字列表現のみが保存されるのです。
よく使われる [`anyhow`] ライブラリは、この問題を解決してくれます：
`CustomError` 型と同様に、
その [`Context`] trait を使って説明を追加することができるのです。
さらに、このライブラリはオリジナルのエラーも保持するので、
根本的な原因を示すエラーメッセージの「連鎖」を得ることができます。

[`anyhow`]: https://docs.rs/anyhow
[`Context`]: https://docs.rs/anyhow/1.0/anyhow/trait.Context.html

まず、`Cargo.toml` ファイルの
`[dependencies]` セクションに `anyhow = "1.0"` を追加して、
`anyhow` crate をインポートしましょう。

すると、完全な例は次のようになります：

```rust,ignore
{{#include errors-exit.rs}}
```

この場合、このようにエラーが表示されます：

```text
Error: could not read file `test.txt`

Caused by:
    No such file or directory (os error 2)
```
