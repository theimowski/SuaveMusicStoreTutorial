Indexビュー
===========

`View.fs` ファイルに最初のビューを追加しましょう：

````fsharp
module SuaveMusicStore.View

open Suave.Html


let divId id = divAttr ["id", id]
let h1 xml = tag "h1" [] xml
let aHref href = tag "a" ["href", href]

let index =
    html [
        head [
            title "Suave Music Store"
        ]

        body [
            divId "header" [
                h1 (aHref "/" (text "F# Suave Music Store"))
            ]

            divId "footer" [
                text "built with "
                aHref "http://fsharp.org" (text "F#")
                text " and "
                aHref "http://suave.io" (text "Suave.IO")
            ]
        ]
    ]
    |> xmlToString
````

このコードは今回のアプリケーションで共通するレイアウトを構成しています。
いくつか補足して説明します：

* HTMLマークアップを生成する関数を呼ぶために、`Suave.Html` モジュールをオープンします
* 続いて3つのヘルパー関数があります
  * `divId` は文字列の属性`id`を持つ`div`要素を追加します
  * `h1` はレベル1のHTMLヘッダを生成する関数で、内部要素を引数にとります
  * `aHref` は文字列の属性`href`と、`a`要素の内部HTMLを引数にとります
* `tag` 関数はSuaveで定義されたもので、`string -> Attribute list -> Xml -> Xml`
  という型になっています。最初の引数はHTML要素の名前で、2つめは属性のリスト、
  3つめは内部のマークアップです。
* `Xml` はSuave内で定義された型で、HTMLマークアップに対するオブジェクトモデル
  になっています
* `index` はHTMLマークアップの全体です
* `html` はその他のタグのリストを引数にとる関数です。
  今回は `head` と `body` です。
* `text` はHTML要素内でプレーンテキストを出力します。
* `xmlToString` はオブジェクトモデルを変換して、生のHTML文字列を返します。

> 注釈：Suaveの `tag` 関数は3引数関数です。しかし上で定義した `aHref` 関数では
> `tag` に2引数しか指定していないにもかかわらず、
> コンパイラは何も文句を言いませんでした。これは何故でしょう？
> この概念は「部分適用」と呼ばれるもので、引数の一部だけを指定しても関数を
> 呼び出す事が出来るというものです。
> 引数の一部だけを指定してある関数を呼び出すと、その関数は残りの引数をとる関数を
> 返します。今回の場合、`aHref`の型は`string -> Xml -> Xml`です。
> したがって`aHref`の「隠された」2番目の引数は`Xml`型です。
> 部分適用の詳細については [こちら][partialapplication] を参照してください。

上のコードでは「パイプ」演算子 `|>` も使用しています。
この演算子はUNIXの経験がある方にはなじみやすいものでしょう。
F#における`|>`演算子の意味は、左辺の値をとり、それに対して右辺の関数を
適用するというものです。
今回のコードでは非常に単純で、HTMLオブジェクトモデルに対して
`xmlToString`関数を呼び出すという意味です。

`index`ビューを`App.fs`でテストしてみましょう：

````fsharp
path "/" >>= (OK View.index)
````

アプリケーションのルートURLにブラウザでアクセスすると、
設定した通りのHTMLが返されることが確認出来るはずです。

[partialapplication]: http://fsharpforfunandprofit.com/posts/partial-application/

