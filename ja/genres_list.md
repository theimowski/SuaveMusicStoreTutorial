# ジャンル一覧

`DbContext`のインスタンスをもっと簡単に取得できるように、
`Db`モジュールへ小さなヘルパー関数を追加することにしましょう：

```fsharp
let getContext() = Sql.GetDataContext()
```

これでようやく`App`モジュール内で実際のデータを読み取る事ができるようになります：

```fsharp
let overview =
    Db.getContext() 
    |> Db.getGenres 
    |> List.map (fun g -> g.Name) 
    |> View.store 
    |> html

...

    path Path.Store.overview >=> overview
```

`overview`はWebPartで...
ちょっと待った。本当に説明要ります？
今回のように、パイプ演算子を使用するとデータの流れが簡潔に把握できるようになります。
各行が各ステップを表しています。
DbContextから始まって、1つの関数の返値が続く関数に渡され、
最終的にはWebPartが返されます。
これは関数プログラミングにおいては関数を「組み合わせて」
機能させることができるという一例です。

また、`overview` のWebPartを`warbler`でラップする必要もあります：

```fsharp
let overview = warbler (fun _ ->
    Db.getContext() 
    |> Db.getGenres 
    |> List.map (fun g -> g.Name) 
    |> View.store 
    |> html)
```

これは`overview`のWebPartがある意味で静的(static)、
つまり結果に影響を与えるような引数を取らないからです。
`warbler`を指定することにより、新しいリクエストを受け取る度に毎回データベースから
ジャンル一覧を取得できるようになります。
`warbler`を指定しなかった場合、ジャンル一覧はアプリケーションの起動時に1回だけしか
取得されなくなります。
そうすると、リストが更新されても一覧が古いままになってしまいます。
他のWebPartはどうでしょうか？

- `browse` はジャンル名を引数にとります。
  各リクエストの度にデータベースへ問い合わせます。
- `details` はidを引数に取ります。
  上に同じです。
- `home` は問題ありません。
  今のところ完全に静的なコンテンツで、データベースに問い合わせる必要がありません。
