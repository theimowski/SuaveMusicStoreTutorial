コンテナ
========

スタイルを適用出来るようになったので、今後追加されることになるビューでも
使用できるように共通のレイアウトを切り出す事にしましょう。
まず`View`の`index`に`container`引数を追加します：

````fsharp
let index container =
    html [
    ...
````

そしてdiv「header」の直後にidが「main」のdivを追加します：

````fsharp
    divId "header" [
        h1 (aHref Path.home (text "F# Suave Music Store"))
    ]

    divId "main" container
````

以前の`index`は定数値でしたが、この変更により
`container`を引数にとる関数になりました。

次に「home」ページ用のコンテナの内容を定義します：

````fsharp
let home = [
    text "Home"
]
````

今のところは「Home」というテキストだけを含む事になります。
続いて、コンテナを引数にとってWebPartを作成できるように共通関数を切り出します。
`App`モジュールにある`browse`のWebPartの直前に以下のコードを追加します：

````fsharp
let html container =
    OK (View.index container)
````

この関数は以下のようにして呼び出せます：

````fsharp
    path Path.home >=> html View.home
````

`View`モジュール内に他のルート用のコンテナも追加しましょう：

````fsharp
let home = [
    text "Home"
]

let store = [
    text "ストア"
]

let browse genre = [
    text (sprintf "ジャンル: %s" genre)
]

let details id = [
    text (sprintf "詳細 %d" id)
]
````

`home`と`store`はいずれも定数値ですが、`browse`と`details`は
それぞれ`genre`と`id`を引数にとる関数になっている点に注意してください。

4つのビューで`html`を使用するように`App`を書き換えます：

````fsharp
let browse =
    request (fun r ->
        match r.queryParam "genre" with
        | Choice1Of2 genre -> html (View.browse genre)
        | Choice2Of2 msg -> BAD_REQUEST msg)

let webPart =
    choose [
        path Path.home >=> html View.home
        path "/store" >=> html View.store
        path "/store/browse" >=> browse
        pathScan Path.Store.details (fun id -> html (View.details id))
        
        pathRegex "(.*)\.(css|png)" >=> Files.browseHome
    ]
````
