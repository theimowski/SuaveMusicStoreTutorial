# カートへ追加

`App`のハンドラーを実装するために、`Db`モジュールへ以下の関数を追加します：

```fsharp
let getCart cartId albumId (ctx : DbContext) : Cart option =
    query {
        for cart in ctx.``[dbo].[Carts]`` do
            where (cart.CartId = cartId && cart.AlbumId = albumId)
            select cart
    } |> firstOrNone
```

```fsharp
let addToCart cartId albumId (ctx : DbContext)  =
    match getCart cartId albumId ctx with
    | Some cart ->
        cart.Count <- cart.Count + 1
    | None ->
        ctx.``[dbo].[Carts]``.Create(albumId, cartId, 1, DateTime.UtcNow) |> ignore
    ctx.SubmitUpdates()
```

> 訳注：`Db`モジュールの先頭に`open System`があることをチェックしておいてください。

`addToCart` は `cartId` と `albumId` を引数に取ります。
データベース内に既にカートのエントリがある場合は`Count`列をインクリメントし、そうでなければ新しい行を追加します。
`getCart`はカートのエントリがデータベースに既にあるかどうかをチェックします。これは単にcartIdとalbumIdで検索しているだけです。

そして`View`モジュールの`details`関数内、「album-details」のdivの一番最後へ「カートへ追加」のボタンを追加します：

```fsharp
yield pAttr ["class", "button"] [
    aHref (sprintf Path.Cart.addAlbum album.AlbumId) (text "カートへ追加")
]
```

これで`App`モジュールにハンドラーを定義できるようになりました：

```fsharp
let addToCart albumId =
    let ctx = Db.getContext()
    session (function
            | NoSession -> 
                let cartId = Guid.NewGuid().ToString("N")
                Db.addToCart cartId albumId ctx
                sessionStore (fun store ->
                    store.set "cartid" cartId)
            | UserLoggedOn { Username = cartId } | CartIdOnly cartId ->
                Db.addToCart cartId albumId ctx
                succeed)
        >>= Redirection.FOUND Path.Cart.overview
```

```fsharp
pathScan Path.Cart.addAlbum addToCart
```

`addToCart` invokes our `session` function with two flavors:

- `NoSession`の場合には新しいGUIDが作成され、データベースに保存されます。そしてセッションストア内の「cartid」キーに対応する値が更新されます。
- `UserLoggedOn` または `CartIdOnly` の場合はユーザーのカートへアルバムが追加されるだけです。ここでは両方のケースで`cartId`文字列をバインドできることに注意してください。既に説明したように、`cartId`はユーザーがログインしていない場合にはGUID、そうでなければユーザー名になっています。
