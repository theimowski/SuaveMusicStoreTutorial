# ビューの修正

まずストア内にある全ジャンルを左側のナビゲーションメニューにしましょう。
そのほうがユーザーのお気に入りを簡単に見つけ出せるようになるはずです。

`View`モジュールに`partGenres`を追加します：

```fsharp
let partGenres (genres : Db.Genre list) =
    ulAttr ["id", "categories"] [
        for genre in genres -> 
            li (aHref 
                    (Path.Store.browse |> Path.withParam (Path.Store.browseKey, genre.Name)) 
                    (text genre.Name))
    ]
```

`partGenres`は各ジャンルへのリンクを含んだ、順序無しリストを作成します。

このリストをメインのindexビューに追加するため、新しい引数を1つ追加して、`container`の前で描画されるようにします：

```fsharp
let index partNav partUser partGenres container = 
    html [

    ...
    
        partGenres
    
        divId "main" container
        
        ...
```

これにあわせて、`html`WebPartの`result`内での呼び出し方を変更します：

```fsharp
let result cartItems user =
        OK (View.index 
                (View.partNav cartItems) 
                (View.partUser user) 
                (View.partGenres (Db.getGenres ctx))
                container)
        >>= Writers.setMimeType "text/html; charset=utf-8"
```

`View.browse`でのアルバム一覧の見た目も改善しましょうか。

```fsharp
let browse genre (albums : Db.Album list) = [
    divClass "genre" [ 
        h2 (sprintf "ジャンル: %s" genre)

        ulAttr ["id", "album-list"] [
            for album in albums ->
                li (aHref 
                        (sprintf Path.Store.details album.AlbumId) 
                        (flatten [ imgSrc album.AlbumArtUrl
                                   span (text album.Title)]))
        ]
    ]
]
```

このビューはやはり単なる順序無しリストですが、タイトルだけでなくアルバムアートを画像表示するようになりました。

ホームページもあまり立派なものではない感じがしたと思います。実際、単に「ホーム」というタイトルしか表示されていません。そこで、バナーイメージと、ベストセラーのアルバムリストを表示させてみるというのはどうでしょうか。

まず`Db`からベストセラーアルバムを取得します：

```fsharp
type BestSeller = DbContext.``[dbo].[BestSellers]Entity``
```

```fsharp
let getBestSellers (ctx : DbContext) : BestSeller list  =
    ctx.``[dbo].[BestSellers]`` |> Seq.toList
```

そして`View.home`を変更します：

```fsharp
let home (bestSellers : Db.BestSeller list) = [
    imgSrc "/home-showcase.png"
    h2 "今売れているアルバム"
    ulAttr ["id", "album-list"] [
            for album in bestSellers ->
                li (aHref 
                        (sprintf Path.Store.details album.AlbumId) 
                        (flatten [ imgSrc album.AlbumArtUrl
                                   span (text album.Title)]))
        ]
]
```

そして`App`モジュールに`home`ハンドラーを追加します：

```fsharp
let home =
    let ctx = Db.getContext()
    let bestsellers = Db.getBestSellers ctx
    View.home bestsellers |> html
```

```fsharp
    path Path.home >>= home
```

アセット「home-showcase.png」は [こちら](https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/home-showcase.png) からダウンロードできます。「出力ディレクトリにコピー」のプロパティを設定し忘れないように！

これですべて終了です。このチュートリアルが皆さんのお気に召しますように。
質問やコメントがあればGitHubでIssueを投稿してください。
作成したアプリケーションのソースコードは[こちら](https://github.com/theimowski/SuaveMusicStore)、本書のコンテンツについては[こちら](https://github.com/theimowski/SuaveMusicStoreTutorial/blob/master/SUMMARY.md)にあります。

それではお元気で！
