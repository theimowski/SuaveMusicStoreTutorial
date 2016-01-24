HTMLマークアップ
================

ではコンテナに置いてあったプレースホルダー用の単なるテキストを
意味のあるコンテンツに置き換えていきましょう。
まずレベル2のHTMLヘッダを出力出来るように、`View`モジュールで`h2`関数を定義します：

````fsharp
let h2 s = tag "h2" [] (text s)
````

そして4つのコンテナそれぞれに置かれた`text`を`h2`に置き換えます。

「/store」のルートではMusic Store内のすべてのジャンルに対するリンクを
出力させようと思います。
そこで、`Path`モジュールに以下の関数を追加して、キーバリュー引数を持つような
HTTP URLを作成出来るようにします：

````fsharp
let withParam (key, value) path = sprintf "%s?%s=%s" path key value
````

`withParam` 関数は1番目の引数にタプル `(key, value)`、
2番目の引数に `path` をとり、適切にフォーマットされたURLを文字列として返します。
タプル(tupleあるいは組(pair))はF#のあちこちで多用されています。
タプルを作成する構文は `(item1, item2)` という感じで、
これはたとえばC#のような多くの言語において引数を指定する方法とよく似ています。
タプルの詳細については[こちら][tuple]を参照してください。

そして「/store/browse」のURL引数のキーとなる文字列を
`Path.Store`モジュールに追加します：

````fsharp
let browseKey = "genre"
````

このキーは今後、`App`モジュール内で以下のように使用することになります：

````fsharp
    match r.queryParam Path.Store.browseKey with
    ...
````

次に順序無しリスト(`ul`)とリストの項目(`li`)のHTMLを出力する
関数を追加します：

````fsharp
let ul xml = tag "ul" [] (flatten xml)
let li = tag "li" []
````

`flatten`は`Xml`のリストをとり、単一の「平らになった(flatten)」
`Xml`オブジェクトモデルを返します。
`store`用のコンテナの中身は以下のようになります：

````fsharp
let store genres = [
    h2 "ジャンル一覧"
    p [
        text (sprintf "%d 個のジャンルから選択してください：" (List.length genres))
    ]
    ul [
        for g in genres ->
            li (aHref (Path.Store.browse |> Path.withParam (Path.Store.browseKey, g)) (text g))
    ]
]
````

このコードの要点は以下の通りです：

* `store`はジャンルのリストを引数にとるようになりました
  (繰り返しますが、型はコンパイラによって推論されます)
* `[ for g in genres -> ... ]` の構文は「リスト内包表記」と呼ばれるものです。
  ここでは`genres`内にある、ジャンルを表す文字列をリスト項目にマッピングしています。
* リスト項目内で使用している`aHref`は「genre」引数をとるような`Path.Store.browse`の
  URLにリンクしています。ここでは先ほど定義した`Path.withParam`関数を使用しています。

`App`モジュールでは、ひとまず`View.store`を使用できるように、
以下のようなハードコードされた`genres`リストを指定しておくことにします：

````fsharp
path Path.Store.overview >=> html (View.store [ "ロック"; "ダンス"; "ポップ"])
````

これまでの変更をまとめると [Tag - Viwe][tag_view] のようになります。
([Suave 0.28.1](https://github.com/SuaveIO/suave/tree/v0.28.1))

[tuple]: http://fsharpforfunandprofit.com/posts/tuples/
[tag_view]: https://github.com/theimowski/SuaveMusicStore/tree/view
