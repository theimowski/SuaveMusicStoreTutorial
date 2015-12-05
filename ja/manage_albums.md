# アルバムの管理

まず`View`モジュールに`manage`を追加するところから始めましょう：

```fsharp
let manage (albums : Db.AlbumDetails list) = [ 
    h2 "Index"
    table [
        yield tr [
            for t in ["アーティスト";"タイトル";"ジャンル";"価格"] -> th [ text t ]
        ]

        for album in albums -> 
        tr [
            for t in [ truncate 25 album.Artist; truncate 25 album.Title; album.Genre; formatDec album.Price ] ->
                td [ text t ]
        ]
    ]
]
```

テーブルのHTMLマークアップ用にいくつか新しいヘルパー関数を用意する必要もあります：

```fsharp
let table x = tag "table" [] (flatten x)
let th x = tag "th" [] (flatten x)
let tr x = tag "tr" [] (flatten x)
let td x = tag "td" [] (flatten x)
```

また、セルの内容が最大文字数を超えないようにするための関数`truncate`も追加します：

```fsharp
let truncate k (s : string) =
    if s.Length > k then
        s.Substring(0, k - 3) + "..."
    else s
```

以下補足です：

- 上のHTMLテーブルは1行目(`tr`)に列ヘッダー(`th`)があり、2行目以降の各セル(`td`)内でアルバムの情報を表示しています。
- このコードでは初めて`yield`キーワードを使用しています。ここでは`for album in albums ->`のリスト内包表記で得られるリストと連結した状態にする必要があるため、このキーワードを指定しなければいけません。大雑把に説明すると、リスト内包表記を使用する場合、内包表記とは別の場所でリストに含める値を返そうとする場合には常に`yield`を指定する必要があるということです。なかなか覚えづらい事ですが、心配する必要はありません。`yield`キーワードを指定し忘れたとしても、コンパイラがそれを警告してくれるようになっています。
- 入力の手間を省くために、ネストされたリスト内包表記を使用して`th`と`td`を出力しています。ここもやはり単に好みの問題で、各要素をそれぞれ個別に列挙して記述することもできます。

続いて、すべての`AlbumDetails`のリストをデータベースから取得する必要があります。
そこで`Db`モジュールに以下のクエリを追加します：

```fsharp
let getAlbumsDetails (ctx : DbContext) : AlbumDetails list = 
    ctx.``[dbo].[AlbumDetails]`` |> Seq.toList
```

これでアルバムのリストを一覧表示するハンドラを定義する準備が整いました。
`Path`に新しいサブモジュールを追加しましょう：

```fsharp
module Admin =
    let manage = "/admin/manage"
```

`Admin`サブモジュールにはアルバム管理用のパスやルートが追加されることになります。

`App`モジュールの`manage`WebPartは以下のように実装できます：

```fsharp
let manage = warbler (fun _ ->
    Db.getContext()
    |> Db.getAlbumsDetails
    |> View.manage
    |> html)
```

そしてメインの`choose`WebPartで次のように呼び出します：

```fsharp
    path Path.Admin.manage >>= manage
```

`manage`WebPartには`warbler`を指定することを忘れないようにしてください。
このWebPartには引数を指定しないので、先行評価にならないようにしなければいけません。

アプリケーションを起動して、ブラウザで「/admin/manage」にアクセスすると、ストア内のアルバムがグリッド表示されることが確認できるはずです。
