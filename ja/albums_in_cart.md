# カート内のアルバム

今回もやはり`CartIdOnly`を把握する必要があります。`session`関数に戻り、コードを以下のように変更して`CartIdOnly`を処理出来るようにします：

```fsharp
match state.get "cartid", state.get "username", state.get "role" with
| Some cartId, None, None -> f (CartIdOnly cartId)
| _, Some username, Some role -> f (UserLoggedOn {Username = username; Role = role})
| _ -> f NoSession
```

つまりセッションストアが`cartid`キーを含み、`username`または`role`を含まない場合には`CartIdOnly`を引数にして`f`を呼び出します。

特定の`cartId`から`CartDetails`を取得するために、`Db`モジュールへそれ用の関数を追加しましょう：

```fsharp
let getCartsDetails cartId (ctx : DbContext) : CartDetails list =
    query {
        for cart in ctx.``[dbo].[CartDetails]`` do
            where (cart.CartId = cartId)
            select cart
    } |> Seq.toList
```

そうすると`cart`ハンドラーをきちんと実装できるようになります：

```fsharp
let cart = 
    session (function
    | NoSession -> View.emptyCart |> html
    | UserLoggedOn { Username = cartId } | CartIdOnly cartId ->
        let ctx = Db.getContext()
        Db.getCartsDetails cartId ctx |> View.cart |> html)
```

今回も同じ動作に対して異なるパターンを使用しています。
`CartIdOnly`と`UserLoggedOn`の両方の場合で、カートの詳細をデータベースから取得しています。
