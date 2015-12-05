# ジャンル

次は「browse」のWebPartです。
まず`View`モジュールを書き換えていきます：

```fsharp
let browse genre (albums : Db.Album list) = [
    h2 (sprintf "ジャンル: %s" genre)
    ul [
        for a in albums ->
            li (aHref (sprintf Path.Store.details a.AlbumId) (text a.Title))
    ]
]
```

ジャンルの名前(文字列)と、ジャンルに属するアルバムのリストを引数にとるようにします。
それぞれのアルバムに対しては、アルバムの詳細を表示するためのリンクをリスト項目として表示するようにしています。

> 注釈：ここでは`IntPath`型の値`Path.Store.details`と`sprintf`関数を使用して、
> リンクのURLを作成しています。
> ここでもやはり静的に型付けされた状態で安全にコードを記述できます。

そして`browse`のWebPart自体を変更します：

```fsharp
let browse =
    request (fun r ->
        match r.queryParam Path.Store.browseKey with
        | Choice1Of2 genre -> 
            Db.getContext()
            |> Db.getAlbumsForGenre genre
            |> View.browse genre
            |> html
        | Choice2Of2 msg -> BAD_REQUEST msg)
```

今回もパイプ演算子を使う事により、クエリ引数として指定された`genre`が処理されていく様子が分かりやすくなっています。

> 注釈：上のコードでは`Db.getAlbumsForGenre` と `View.browse`に対して「部分適用」を行っています。
> パイプ演算子の結果はそれに続く関数の最終引数として渡されるため、このように記述できます。

ブラウザで「/store/browse?genre=Latin」を開くと、一部の文字列が正しく表示されていないことがわかります。
そこで、適切な文字セットが指定された「Content-Type」ヘッダを各HTTPレスポンスに追加します：

```fsharp
let html container =
    OK (View.index container)
    >>= Writers.setMimeType "text/html; charset=utf-8"
```