# カート用のナビゲーションメニュー

ナビゲーションメニューに`カート`のタブがあった事を覚えていますか？
このタブでカートに追加されたアルバムの合計の数を表示してみるというのはどうでしょう？
そこで、まず`partNav`ビューに引数を追加します：

```fsharp
let partNav cartItems = 
    ulAttr ["id", "navlist"] [ 
        li (aHref Path.home (text "ホーム"))
        li (aHref Path.Store.overview (text "ストア"))
        li (aHref Path.Cart.overview (text (sprintf "カート (%d)" cartItems)))
        li (aHref Path.Admin.manage (text "管理者"))
    ]
```

また、`index`ビューにも`partNav`引数を追加します：

```fsharp
let index partNav partUser container = 
```

そうすると、ナビゲーションメニューを作成するために`cartItems`引数を指定しなければいけなくなりました。
この問題は`App`モジュールの`html`関数を以下のように変更することで解決できます：

```fsharp
let html container =
    let ctx = Db.getContext()
    let result cartItems user =
        OK (View.index (View.partNav cartItems) (View.partUser user) container)
        >=> Writers.setMimeType "text/html; charset=utf-8"

    session (function
    | UserLoggedOn { Username = username } -> 
        let items = Db.getCartsDetails username ctx |> List.sumBy (fun c -> c.Count)
        result items (Some username)
    | CartIdOnly cartId ->
        let items = Db.getCartsDetails cartId ctx |> List.sumBy (fun c -> c.Count)
        result items None
    | NoSession ->
        result 0 None)
```

変更点は`html`内の`result`関数で、`cartItems` と `user` の2つを引数にとるようになりました。
また、`session`を呼び出す際にすべてのケースを処理するようにしています。
それぞれのケースにおいて、それぞれ異なる引数で`result`関数が呼ばれている点に注意してください。
