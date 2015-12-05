# アルバムの削除

この時点ではアルバムに対して何の操作もできません。
そこでまず、削除機能を追加することにしましょう：

```fsharp
let deleteAlbum albumTitle = [
    h2 "Delete Confirmation"
    p [ 
        text "以下のタイトルを削除します。よろしいですか？"
        br
        strong albumTitle
    ]
    
    form [
        submitInput "Delete"
    ]

    div [
        aHref Path.Admin.manage (text "リストへ戻る")
    ]
]
```

`deleteAlbum` は `View` モジュール内に定義します。
この関数には以下の新しいマークアップ関数が必要です：

```fsharp
let strong s = tag "strong" [] (text s)

let form x = tag "form" ["method", "POST"] (flatten x)
let submitInput value = inputAttr ["type", "submit"; "value", value]
```

- `strong` は単に強調表示するものです
- `form` は「method」属性に「POST」が指定されたHTML form要素を出力します
- `submitInput` はフォームの送信ボタンを出力します

`deleteAlbum`を実装するためにはもう少しコードが必要です。
まず`Db`モジュールに以下を追加します：

```fsharp
let getAlbum id (ctx : DbContext) : Album option = 
    query { 
        for album in ctx.``[dbo].[Albums]`` do
            where (album.AlbumId = id)
            select album
    } |> firstOrNone
```

これで`Album option`が取得出来るようになります(`AlbumDetails`ではありません)。
次に`Path`へ新しいルートを追加します：

```fsharp
module Admin =
    let manage = "/admin/manage"
    let deleteAlbum : IntPath = "/admin/delete/%d"
```

最後に`App`モジュールに以下を追加します：

```fsharp
let deleteAlbum id =
    match Db.getAlbum id (Db.getContext()) with
    | Some album ->
        html (View.deleteAlbum album.Title)
    | None ->
        never
```

```fsharp
    pathScan Path.Admin.deleteAlbum deleteAlbum
```

ここまでのコードで、「/admin/delete/%d」にアクセス出来るようにはなったのですが、まだ実際にアルバムを削除することは出来ないことに注意してください。
というのも、まだデータベースからアルバムを削除する機能を処理する部分が未実装だからです。
今のところ、GETリクエストとPOSTリクエストの両方とも、削除するアルバムを選択するためのHTMLページが返されるようになっています。

削除機能を実装するために、`Db`モジュールへ`deleteAlbum`を追加します：

```fsharp
let deleteAlbum (album : Album) (ctx : DbContext) = 
    album.Delete()
    ctx.SubmitUpdates()
```

この関数は`Album`を引数に取ります。
このインスタンスはデータベース由来の型で、`Delete()`メンバーメソッドを呼び出せます。
SQLProviderは変更を追跡するようになっているため、`ctx.SubmitUpdates()`を呼ぶと、データの変更に必要なSQL命令が実行されます。この機能はある意味「アクティブレコード(Active Record)」のコンセプトに似ています。

そうすると`App`モジュールでGETとPOSTを区別出来るようになります：

```fsharp
let deleteAlbum id =
    let ctx = Db.getContext()
    match Db.getAlbum id ctx with
    | Some album ->
        choose [ 
            GET >>= warbler (fun _ -> 
                html (View.deleteAlbum album.Title))
            POST >>= warbler (fun _ -> 
                Db.deleteAlbum album ctx; 
                Redirection.FOUND Path.Admin.manage)
        ]
    | None ->
        never
```

- `deleteAlbum` WebPartは2通りの`choose`を通るようになります
- `GET` と `POST` は受信したHTTPリクエストがGETまたはPOSTの場合に成功する(`Some x`を返す)WebPartです。
- アルバムの削除に成功すると、`POST`ケースにあるコードによって`Path.Admin.manage`ページへリダイレクトされます。

> 重要：このコードではGETおよびPOSTのハンドラを`warbler`で囲っています。そうしないと、`Some album`にマッチした時点でこれらのコードが評価されてしまい、POSTリクエストではないにもかかわらず`Db.deleteAlbum`が呼ばれてしまいます。

これでアルバムを削除するためのリンクをグリッドの列へ追加できます：

```fsharp
table [
        yield tr [
            for t in ["アーティスト";"タイトル";"ジャンル";"価格";""] -> th [ text t ]
        ]

        for album in albums -> 
        tr [
            for t in [ truncate 25 album.Artist; truncate 25 album.Title; album.Genre; formatDec album.Price ] ->
                td [ text t ]

            yield td [
                aHref (sprintf Path.Admin.deleteAlbum album.AlbumId) (text "削除")
            ]
        ]
    ]
```

- 1行目に空文字の列ヘッダを追加しました
- 各アルバムを表示する行の最後列に、アルバムを削除するためのリンクを追加しました
- ここでも`yield`を指定している点に注意してください
