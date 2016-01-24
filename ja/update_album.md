# アルバムの更新

削除と作成が出来るようになったので、残るは更新だけです。
この機能は作成機能とかなり共通するものがあるので、比較的簡単です(`Form`モジュールで定義したフォームを再利用出来ます)。

`View`に`editAlbum`を追加します：

```fsharp
let editAlbum (album : Db.Album) genres artists = [ 
    h2 "編集"
        
    renderForm
        { Form = Form.album
          Fieldsets = 
              [ { Legend = "アルバム"
                  Fields = 
                      [ { Label = "ジャンル"
                          Xml = selectInput (fun f -> <@ f.GenreId @>) genres (Some (decimal album.GenreId)) }
                        { Label = "アーティスト"
                          Xml = selectInput (fun f -> <@ f.ArtistId @>) artists (Some (decimal album.ArtistId))}
                        { Label = "タイトル"
                          Xml = input (fun f -> <@ f.Title @>) ["value", album.Title] }
                        { Label = "価格"
                          Xml = input (fun f -> <@ f.Price @>) ["value", formatDec album.Price] }
                        { Label = "アルバムアートのURL"
                          Xml = input (fun f -> <@ f.ArtUrl @>) ["value", "/placeholder.gif"] } ] } ]
          SubmitText = "変更を保存" }

    div [
        aHref Path.Admin.manage (text "リストへ戻る")
    ]
]
```

パスを追加します：

```fsharp
    let editAlbum : IntPath = "/admin/edit/%d"    
```

`View`の`manage`でリンクを追加します：

```fsharp
for album in albums -> 
        tr [
            ...

            yield td [
                aHref (sprintf Path.Admin.editAlbum album.AlbumId) (text "編集")
                text " | "
                aHref (sprintf Path.Admin.deleteAlbum album.AlbumId) (text "削除")
            ]
        ]
```

`Db`モジュールに`updateAlbum`を追加します：

```fsharp
let updateAlbum (album : Album) (artistId, genreId, price, title) (ctx : DbContext) =
    album.ArtistId <- artistId
    album.GenreId <- genreId
    album.Price <- price
    album.Title <- title
    ctx.SubmitUpdates()
```

`App`モジュールに`editAlbum` WebPartを追加します：

```fsharp
let editAlbum id =
    let ctx = Db.getContext()
    match Db.getAlbum id ctx with
    | Some album ->
        choose [
            GET >=> warbler (fun _ ->
                let genres = 
                    Db.getGenres ctx 
                    |> List.map (fun g -> decimal g.GenreId, g.Name)
                let artists = 
                    Db.getArtists ctx
                    |> List.map (fun g -> decimal g.ArtistId, g.Name)
                html (View.editAlbum album genres artists))
            POST >=> bindToForm Form.album (fun form ->
                Db.updateAlbum album (int form.ArtistId, int form.GenreId, form.Price, form.Title) ctx
                Redirection.FOUND Path.Admin.manage)
        ]
    | None -> 
        never
```

そして最後にメインの`choose` WebPartで`pathScan`を追加します：

```fsharp
    pathScan Path.Admin.editAlbum editAlbum
```

前述のコードについてコメントしておきます：

- `editAlbum` ビューは`createAlbum`とかなり似ています。ただし、いずれのフィールドもあらかじめ値が設定されるという大きな違いがあります。
- `Db.updateAlbum`ではプロパティのsetアクセサーを使用しています。`Album`の値はSQLProviderにより値を変更できるように定義されていますが、`SubmitUpdates()`を呼び出す前の変更は追跡されています。
- `editAlbum`のGETハンドラーでは先行評価を避けるために`warbler`が必須です。
- 一方、POSTでは受信リクエストを解析する必要があり、解析が正常に完了するまでは評価が遅延されるため、POST側には指定する必要がありません。
- アルバムが更新されると`manage`へのリダイレクトが行われます

> 注釈：SQLProviderにより作成された`Album`のプロパティはオブジェクトを生成した後でもその値を変更できます。これは関数型プログラミングのパラダイムにおいて一般的な、不変性の概念に反するものです。しかし覚えておいていただきたいのは、F#は純粋関数型プログラミング言語ではなく、「関数型を第一義とする(functional first)」言語であるということです。つまり関数型のスタイルでコードを記述することを推奨しつつも、オブジェクト指向なコードも記述できるのです。このことはたとえばパフォーマンスを改善する必要があるような場合に有利に働きます。

最後のとっておきとして、`View.manage`にアルバムの詳細を表示するためのリンクを追加しましょう：

```fsharp
aHref (sprintf Path.Admin.editAlbum album.AlbumId) (text "編集")
text " | "
aHref (sprintf Path.Store.details album.AlbumId) (text "詳細")
text " | "
aHref (sprintf Path.Admin.deleteAlbum album.AlbumId) (text "削除")
```

やれやれ。この章はかなりのボリュームでしたが、学ぶところもたくさんありました。アプリケーションの重要な機能が動作するようになったはずです！
これまでの変更をまとめると [Tag - crud_and_forms](https://github.com/theimowski/SuaveMusicStore/tree/crud_and_forms) のようになります。
[Tag - Suave0.28.1](https://github.com/SuaveIO/suave/tree/v0.28.1)
