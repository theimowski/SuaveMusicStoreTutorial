クエリ
======

型エイリアスを定義したので、いよいよ最初のクエリを作成していきます：

````fsharp
let firstOrNone s = s |> Seq.tryFind (fun _ -> true)

let getGenres (ctx : DbContext) : Genre list =
    ctx.``[dbo].[Genres]`` |> Seq.toList

let getAlbumsForGenre genreName (ctx : DbContext) : Album list =
    query {
        for album in ctx.``[dbo].[Albums]`` do
            join genre in ctx.``[dbo].[Genres]`` on (album.GenreId = genre.GenreId)
            where (genre.Name = genreName)
            select album
    }
    |> Seq.toList

let getAlbumDetails id (ctx : DbContext) : AlbumDetails option =
    query {
        for album in ctx.``[dbo].[AlbumDetails]`` do
            where (album.AlbumId = id)
            select album
    } |> firstOrNone
````

`getGenres` はすべてのジャンルを検索するための関数です。
今後、`Db`モジュールに追加していくことになる他の関数と同じく、
この関数は`DbContext`を引数にとります。
`: Genre list` の部分は型注釈(type annotation)で、
関数の返り値が`Genre`のリストであることを明記しています。
実装自体は単純で、 ```ctx.``[dbo].[Genres]`` ``` とすると
すべてのジャンルを問い合わせられるので、後は単に結果を`Seq.toList`に渡すだけです。

`getAlbumsForGenre`は(string型と推論される)`genreName` を引数にとり、
`Album`のリストを返します。
ここでは「クエリ式(query expression)」(`query { }`) を使用しています。
F#のクエリ式ははC#のクエリ式とよく似ています。
クエリ式の詳細については[こちら][queryexpression]を参照してください。
クエリ式の内部では、`Albums`と`Genres`を外部キー`GenreId`で
内部連結(inner join)した後、入力された`genreName`と`genre.Name`が
一致するかどうかという条件句を指定しています。
その後、クエリの結果を`Seq.toList`に渡しています。

`getAlbumDetails`は(int型と推論される)`id`を引数にとり、
`AlbumDetails option`型を返します。
option型になっているのは、特定のidに対応するアルバムが存在しない場合があるためです。
今回のクエリ結果は`firstOrNone`に渡しています。
`firstOrNone`はクエリの結果を`option`型に変換する処理を行っていて、
1つ以上の結果が返されたかどうかをチェックしています。
もし1つ以上の結果が返された場合は`Some x`、そうでなければ`None`が返されます。

[queryexpression]: https://msdn.microsoft.com/en-us/library/hh225374.aspx
