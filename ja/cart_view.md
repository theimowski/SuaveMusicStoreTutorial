# カート用ビュー

「ユーザーの観点からすると、カートが空のときはカートが空ではなくせるようにカートが空だということがわかるようになっていてほしい」
こういったビジネス上の要求に対応出来るよう、カートの中身が空かそうでないのかを区別できるようにしておいたほうがよいでしょう。
そこで、`View`モジュールに`emptyCart`を追加します：

```fsharp
let emptyCart = [
    h2 "カートは空です"
    text "是非我々の "
    aHref Path.home (text "ストア")
    text " で素敵な音楽を見つけてください!"
]
```

カートの中身が空でなかった場合には、カート内のアルバムをテーブル表示できるようにします：

```fsharp
let aHrefAttr href attr = tag "a" (("href", href) :: attr)

...

let nonEmptyCart (carts : Db.CartDetails list) = [
    h2 "カートの内容を確認します："
    table [
        yield tr [
            for h in ["アルバム名"; "(それぞれの)価格"; "数量"; ""] ->
            th [text h]
        ]
        for cart in carts ->
            tr [
                td [
                    aHref (sprintf Path.Store.details cart.AlbumId) (text cart.AlbumTitle)
                ]
                td [
                    text (formatDec cart.Price)
                ]
                td [
                    text (cart.Count.ToString())
                ]
                td [
                    aHrefAttr "#" ["class", "removeFromCart"; "data-id", cart.AlbumId.ToString()] (text "カートから削除") 
                ]
            ]
        yield tr [
            for d in ["合計"; ""; ""; carts |> List.sumBy (fun c -> c.Price * (decimal c.Count)) |> formatDec] ->
            td [text d]
        ]
    ]
]
```

これらのビューを元に、カートが空かどうかに応じて表示を切り替えるような、さらに汎用のビューを用意します：

```fsharp
let cart = function
    | [] -> emptyCart
    | list -> nonEmptyCart list
```

`cart`は`function`のパターンマッチ構文を使用しています。
`[]`のケースはリストが空の場合にマッチするため、2つめのケースはリストが空ではないことを表す事になります。

`nonEmptyCart`関数にいくつか補足します：

- まず列ヘッダ用の行を返します(「名前」「価格」「数量」)
- そしてリストにある`CartDetail`それぞれに対して、以下を含む行を返します：
    - アルバムのタイトルとして表示された、アルバムの詳細を表示するためのリンク
    - アルバムの単価
    - カート内にあるアルバムの数量
    - カートからアルバムを削除するためのリンク(後ほどAJAXによる更新機能を実装します)
- 最後に、カート内にあるアルバムの総計を表す行を返します

今の時点ではまだデータベースからカートの項目を取得する機能が実装出来ていないので、このビューを実際にテストすることが出来ません。
しかし`App`モジュールにハンドラーを追加すれば`emptyCart`の表示についてだけは確認できます：

```fsharp
let cart = View.cart [] |> html

...

path Path.Cart.overview >=> cart
```

ナビゲーション用のメニューも(`View.partNav`に)簡単に追加できます：

```fsharp
li (aHref Path.Cart.overview (text "カート"))
```
