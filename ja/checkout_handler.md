# 購入フォーム用ハンドラー

まず購入フォーム用のルートを(`Path.Cart`に)追加しましょう：

```fsharp
let checkout = "/cart/checkout"
```

そして`View.nonEmptyCart`にナビゲーション用のボタンを追加します：

```fsharp
let nonEmptyCart (carts : Db.CartDetails list) = [
    h2 "カートの内容を確認します："
    pAttr ["class", "button"] [
            aHref Path.Cart.checkout (text "購入 >>")
    ]

    ...
```

そして`App`モジュールに購入用のGETハンドラーを追加します：

```fsharp
let checkout =
    session (function
    | NoSession | CartIdOnly _ -> never
    | UserLoggedOn {Username = username } ->
        choose [
            GET >=> (View.checkout |> html)
        ])
```

```fsharp
path Path.Cart.checkout >=> loggedOn checkout
```

補足です：

- `checkout` はユーザーがログインしている場合にのみ呼び出せます。`NoSession` または `CartIdOnly`の場合には購入画面に遷移出来ないようにします。
- また、ルーティング用のコードで`checkout` WebPartを`loggedOn`に渡しています。こうすることによって、必要な場合にのみユーザーがリダイレクトされるようにしています。

では次に`Db`モジュールで`placeOrder`関数を実装します：

```fsharp
let placeOrder (username : string) (ctx : DbContext) =
    let carts = getCartsDetails username ctx
    let total = carts |> List.sumBy (fun c -> (decimal) c.Count * c.Price)
    let order = ctx.``[dbo].[Orders]``.Create(DateTime.UtcNow, total)
    order.Username <- username
    ctx.SubmitUpdates()
    for cart in carts do
        let orderDetails = ctx.``[dbo].[OrderDetails]``.Create(cart.AlbumId, order.OrderId, cart.Count, cart.Price)
        getCart cart.CartId cart.AlbumId ctx
        |> Option.iter (fun cart -> cart.Delete())
    ctx.SubmitUpdates()
```

このコードは以下の処理を行います：

1. 特定のユーザーに対応づけられたすべてのカートを取得します
2. 支払総額を計算します
3. 現在の時刻、総額、ユーザー名で新しい注文を作成します
4. `order`インスタンスに適切な`OrderId`が設定されるようにするために`SubmitUpdates`を呼び出します
5. カート内をすべて走査して、それぞれに対して：
    1. `OrderDetails` を作成してすべてのプロパティに値を設定します
    2. データベースから関連する`cart`を削除します。なお現在走査しているカートは`Cart`テーブルではなく、`CartDetails`ビューの行になっているので、そのままでは削除できないことに注意してください。
6. 最後に`SubmitUpdates`を呼び出します。

ユーザーがカートの購入を完了すると、以下の(`View`モジュール内の)確認ページが表示されるようにします：

```fsharp
let checkoutComplete = [
    h2 "購入手続きの完了"
    p [
        text "ご注文ありがとうございました！"
    ]
    p [
        aHref Path.home (text "ミュージックストア")
        text "でもっと買い物していきませんか？"
    ]
]
```

さてこれで購入用のPOSTハンドラーを実装する準備が整いました：

```fsharp
POST >=> warbler (fun _ ->
    let ctx = Db.getContext()
    Db.placeOrder username ctx
    View.checkoutComplete |> html)
```

`Db.placeOrder`はGETハンドラーを処理する際には呼ばれないようにしなければいけないため、warbler内で呼び出す必要があります。

ふう。ようやく完成しました！
コードは [Tag - registration_and_checkout](https://github.com/theimowski/SuaveMusicStore/tree/registration_and_checkout) から参照できます。
 のようになります。
