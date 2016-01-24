# アルバムの詳細

アルバムの詳細をデータベースから読み取りましょう。
まず`View`モジュールの`details`を変更します：

```fsharp
let details (album : Db.AlbumDetails) = [
    h2 album.Title
    p [ imgSrc album.AlbumArtUrl ]
    divId "album-details" [
        for (caption, t) in [ "ジャンル：",album.Genre; "アーティスト：", album.Artist; "価格：", formatDec album.Price ] ->
            p [
                em caption
                text t
            ]
    ]
]
```

上のコードでは、`View`に追加した以下のヘルパー関数を使用しています：

```fsharp
let imgSrc src = imgAttr [ "src", src ]
let em s = tag "em" [] (text s)

let formatDec (d : Decimal) = d.ToString(Globalization.CultureInfo.InvariantCulture)
```

なおファイルの先頭で`System`名前空間をオープンする必要もあります。

> 注釈：`System`名前空間は常にファイルの先頭でオープンするように
> しておくとよいでしょう。実際、役に立つケースが多々あります。

`details`関数では、タプルのリスト(`["ジャンル：",album.Genre;...`)を
インラインに含むようなリスト内包表記を使用しています。
こうすることで、各プロパティに対して`p`要素を3回入力する手間を省く事ができます。
もしお気に召さないようであればこの記法を止めてしまっても問題ありません。

このようにして、必要な属性をたった1ステップで(明示的にjoinすることもなく)
取得出来てしまうわけなので、`AlbumDetails` データベースビューの
ありがたみがようやく確認できました。

`App`モジュール内でアルバムの詳細を読み取るには以下のようにします：

```fsharp
let details id =
    match Db.getAlbumDetails id (Db.getContext()) with
    | Some album ->
        html (View.details album)
    | None ->
        never
```

```fsharp
    pathScan Path.Store.details details
```

上のコードを簡単に説明します：

- `details` は `id` を引数に取りWebPartを返します
- IntPath型の値`Path.Store.details` は型安全な状態で処理を実行できます
- `Db.getAlbumDetails` は指定されたidに該当するアルバムが無かった場合`None`を返します
- アルバムが見つかった場合、コンテナの中身が`View.details`であるhtml WebPartを返します
- アルバムが見つからなかった場合、`never`によって`None`のWebPartを返します

今回はパイプ演算子を使用しませんでしたが、`details` のWebPartをパイプ演算子で
書き換える方法については読者への課題としておきます。

アプリケーションをテストする前に、"placeholder.gif"の画像ファイルを
アセットに追加しておきます。
このファイルは
[こちら](https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/placeholder.gif)
からダウンロードできます。
ファイルのプロパティを開いて「出力ディレクトリにコピー」を設定すること、
また`App`モジュールの`pathRegex`で拡張子を追加しておくことを忘れないでください。

既にお気づきかもしれませんが、存在しないリソースにアクセスしようとした場合
(たとえば適当なアルバムidを指定してアルバムの詳細画面を表示しようとする等)、
レスポンスが全く返されません。
この問題を解決するには、メインにある`choose` WebPartの一番最後の候補として
「ページが見つかりません」を表示させるようにします：

```fsharp
let webPart = 
    choose [
        ...

        html View.notFound
    ]
```

`View.notFound`は以下のようにします：

```fsharp
let notFound = [
    h2 "ページが見つかりません"
    p [
        text "要求されたリソースは見つかりませんでした"
    ]
    p [
        aHref Path.home (text "ホーム")
        text "へ戻る"
    ]
]
```

これまでの変更をまとめると
[Tag - Database](https://github.com/theimowski/SuaveMusicStore/tree/database)
のようになります。
([Suave 0.28.1](https://github.com/SuaveIO/suave/tree/v0.28.1))
