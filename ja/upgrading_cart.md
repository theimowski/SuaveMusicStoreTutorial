# カートのアップグレード

ユーザーがログインしていた場合にはユーザーの名前が取得できます。
そこで、カートの項目をGUIDからユーザー名へとアップグレードさせることができます。
必要になる関数を`Db`モジュールへ追加します：

```fsharp
let getCarts cartId (ctx : DbContext) : Cart list =
    query {
        for cart in ctx.``[dbo].[Carts]`` do
            where (cart.CartId = cartId)
            select cart
    } |> Seq.toList
```

```fsharp
let upgradeCarts (cartId : string, username :string) (ctx : DbContext) =
    for cart in getCarts cartId ctx do
        match getCart username cart.AlbumId ctx with
        | Some existing ->
            existing.Count <- existing.Count +  cart.Count
            cart.Delete()
        | None ->
            cart.CartId <- username
    ctx.SubmitUpdates()
```

そして`App`モジュールの`logon`ハンドラーを変更します：

```fsharp
authenticated Cookie.CookieLife.Session false 
>=> session (function
    | CartIdOnly cartId ->
        let ctx = Db.getContext()
        Db.upgradeCarts (cartId, user.UserName) ctx
        sessionStore (fun store -> store.set "cartid" "")
    | _ -> succeed)
>=> sessionStore (fun store ->
    store.set "username" user.UserName
    >=> store.set "role" user.Role)
>=> returnPathOrHome
```

以下補足です：

- `Db.getCarts`は特定の`cartId`に対するカートの内容を単なるリストとして返します
- `Db.upgradeCarts` は `cartId` と `username` を引数に取り、(`Db.getCarts`から返された)すべてのカートを走査します。そしてそれぞれに対して：
    - `username`と`albumId`に対応するカートがデータベース内に既に見つかる場合、合計を計算した後にGUIDのカートを削除します。これはログイン中のユーザーがカートへアルバムを追加し、ログアウトした後、同じアルバムをカートへ追加し、再度ログインした場合に起こります。アルバムの数としては2になるはずです。
    - `username` と `albumId`に対応するカートが無い場合、カートの`CartId`プロパティに`username`を設定して「アップグレード」するだけです
- `logon` ハンドラーは`Db.upgradeCarts`を呼び出す必要があったので、`CartIdOnly`ケースを認識するようになりました。また、今後は`username`がカートのIDとなるので、セッションストアから`cartId`キーを削除しています。

いやはや、これでミュージックストアにカートの機能が追加されました！
これまでの変更をまとめると [Tag - cart](https://github.com/theimowski/SuaveMusicStore/tree/cart) のようになります。

