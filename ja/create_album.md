# アルバムの作成

アルバムを削除出来るようになったので、続けてアルバムを作成出来るようにしましょう。
アルバムを作成するためにはいくつか入力用のフィールドが必要になるため、前回よりもやや複雑な処理が必要になります。
しかしSuaveライブラリには、まさにそのためのヘルパーモジュールが用意されています。

> 注釈：`Suave.Form` モジュールは本書の執筆時点では`Experimental`パッケージに同梱されています。これは既に利用中の`Suave.Html`と同じです。

まず`Form`というモジュールを用意して、それをフォーム専用とします
(ご想像の通り、すぐにここへコードを追加する予定があります)。
`View.fs`の直前に`Form.fs`ファイルを追加してください。
`View` と `App` モジュールの両方が `Form` に依存します。
他のモジュール同様、モジュールの命名規則に沿った名前を付けておきましょう。

では1つめの`Album`フォームを用意します：

```fsharp
module SuaveMusicStore.Form

open Suave.Form

type Album = {
    ArtistId : decimal
    GenreId : decimal
    Title : string
    Price : decimal
    ArtUrl : string
}

let album : Form<Album> = 
    Form ([ TextProp ((fun f -> <@ f.Title @>), [ maxLength 100 ])
            TextProp ((fun f -> <@ f.ArtUrl @>), [ maxLength 100 ])
            DecimalProp ((fun f -> <@ f.Price @>), [ min 0.01M; max 100.0M; step 0.01M ])
            ],
          [])
```

`Album` 型にはフォームに必要なフィールドがすべて含まれています。
執筆時点において`Suave.Form`がサポートする型は以下の通りです：

- decimal
- string
- System.Net.Mail.MailAddress
- Suave.Form.Password

> 注釈：int型はまだサポートされていませんが、簡単にdecimalからint、あるいはその逆の変換を行う事ができます。

そしてアルバム用のフォームが続きます(`let album : Form<Album> =`)。
ここには「Props」(プロパティ: properties)と、各プロパティで想定する検証条件のリストが含まれます。

- 1行目では`Title`が最大100文字であることを指定しています
- 2行目では`ArtUrl`が同上の条件であることを指定しています
- 3行目では`Price`が0.01から100.0までの間、0.01単位の値をとることを指定しています(つまり1.001は不正な値です)

これらのプロパティはクライアントとサーバー両方で使用できます。
つまり`View`モジュールにある`album`のコードでは、HTML5の入力検証用属性がクライアント側で出力されます。
一方、サーバー側で動作するWebPartでは、リクエストから取り出したフォームフィールドの値を解析するためにこれらのプロパティを使用することになります。

> 注釈：上のコードではF#のコードクォートを使用しています。この機能の詳細は[こちら](https://msdn.microsoft.com/en-us/library/dd233212.aspx)にあります。
> 今回のチュートリアルでは、コードクォートを使用することによって、Suaveのライブラリがプロパティのgetアクセサーを呼び出す式からフィールド名を見つけ出す事が出来るようになっているというところだけ把握しておいてください。

このフォームを`View`モジュールで使用するために、`View.fs`ファイルの先頭へ`open Suave.Form`を追加します：

```fsharp
module SuaveMusicStore.View

open System

open Suave.Html
open Suave.Form
```

次にいくつかのヘルパー関数を追加します：

```fsharp
let divClass c = divAttr ["class", c]

...

let fieldset x = tag "fieldset" [] (flatten x)
let legend txt = tag "legend" [] (text txt)
```

そして以下のコードブロックを追加します：

```fsharp
type Field<'a> = {
    Label : string
    Xml : Form<'a> -> Suave.Html.Xml
}

type Fieldset<'a> = {
    Legend : string
    Fields : Field<'a> list
}

type FormLayout<'a> = {
    Fieldsets : Fieldset<'a> list
    SubmitText : string
    Form : Form<'a>
}

let renderForm (layout : FormLayout<_>) =

    form [
        for set in layout.Fieldsets -> 
            fieldset [
                yield legend set.Legend

                for field in set.Fields do
                    yield divClass "editor-label" [
                        text field.Label
                    ]
                    yield divClass "editor-field" [
                        field.Xml layout.Form
                    ]
            ]

        yield submitInput layout.SubmitText
    ]
```

このコードは長めですが、すぐ後でわかるように、何回かこのコードを再利用することになります。
`FormLayout`はフォームのレイアウトを定義する型で、以下を含みます：

- `SubmitText` は送信ボタン用の文字列です
- `Fieldsets` は `Fieldset` のリストです
- `Form` は描画されるフォームのインスタンスです

`Fieldset`はフィールドセットのレイアウトを定義する型です：

- `Legend` は一連のフィールドに対する文字列です
- `Fields` は `Field` のリストです

`Field` 型には以下が含まれます：

- `Label` はラベルとなる文字列です
- `Xml` は`Form`を受け取り`Xml`(HTMLマークアップを表すオブジェクトモデル)を返す関数です。やや面倒に見えるかもしれませんが、このシグネチャは部分適用が出来るということを見こんであります。

> 注釈：上記の型はいずれもジェネリック型です。つまり任意の種類のフォームを対象にすることが出来ますが、`FormLayout`の階層構造に沿ったものになっていなければいけません。

`renderForm`は再利用可能な関数で、`FormLayout`のインスタンス1つを引数に取り、HTMLオブジェクトモデルを返します：

- この関数はform要素を作成します
- formにはfieldsetのリストが含まれます。それぞれは
    - まずlegendを出力します
    - 続いて`Fields`を走査して、
        - フィールド用のlable要素を含んだdiv要素を出力して
        - フィールドに対応するinput要素を含んだdiv要素を出力します
- 最後に送信ボタンを出力します

`renderForm` は以下のようにして呼び出す事ができます：

```fsharp
let createAlbum genres artists = [ 
    h2 "作成"

    renderForm
        { Form = Form.album
          Fieldsets = 
              [ { Legend = "アルバム"
                  Fields = 
                      [ { Label = "ジャンル"
                          Xml = selectInput (fun f -> <@ f.GenreId @>) genres None }
                        { Label = "アーティスト"
                          Xml = selectInput (fun f -> <@ f.ArtistId @>) artists None }
                        { Label = "タイトル"
                          Xml = input (fun f -> <@ f.Title @>) [] }
                        { Label = "価格"
                          Xml = input (fun f -> <@ f.Price @>) [] }
                        { Label = "アルバムアートのURL"
                          Xml = input (fun f -> <@ f.ArtUrl @>) ["value", "/placeholder.gif"] } ] } ]
          SubmitText = "作成" }

    div [
        aHref Path.Admin.manage (text "リストへ戻る")
    ]
]
```

`Xml`の値として、`selectInput` や `input` を呼び出す関数を指定しています。
これらはいずれも、input要素を出力したいフィールドを示す関数を第1引数にとります。
`input`は追加の属性リスト(`string * string`つまりキーと値のリスト)を第2引数にとります。
`selectInput`はoptionのリスト(`decimal * string`つまり値と表示名のリスト)を第2引数にとります。
また、`selectInput`の第3引数で、選択済みの値を指定することもできます。
`None`を指定すると、初期状態として1番目の項目が選択されます。

> 注釈：今回はアルバムの`ArtUrl`を「/placeholder.gif」という値でハードコーディングしています。画像のアップロード機能については今回は実装しないため、プレースホルダー用の画像が表示されるようにしています。

`createAlbum`のビューが完成したので、WebPartハンドラーを実装していきましょう。

まず`Db`に`getArtists`を追加します：

```fsharp
type Artist = DbContext.``[dbo].[Artists]Entity``

...

let getArtists (ctx : DbContext) : Artist list = 
    ctx.``[dbo].[Artists]`` |> Seq.toList
```

そして`Path`モジュールにエントリを追加します：

```fsharp
    let createAlbum = "/admin/create"
```

`App`モジュールにWebPartを追加します：

```fsharp
let createAlbum =
    let ctx = Db.getContext()
    choose [
        GET >>= warbler (fun _ -> 
            let genres = 
                Db.getGenres ctx 
                |> List.map (fun g -> decimal g.GenreId, g.Name)
            let artists = 
                Db.getArtists ctx
                |> List.map (fun g -> decimal g.ArtistId, g.Name)
            html (View.createAlbum genres artists))
    ]

...

    path Path.Admin.createAlbum >>= createAlbum
```

今回も`warbler`が必須です。`warbler`を指定することでWebPartが先行評価されないようにしています。
また、`View.manage`に`createAlbum`へのリンクを追加します：

```fsharp
let manage (albums : Db.AlbumDetails list) = [ 
    h2 "Index"
    p [
        aHref Path.Admin.createAlbum (text "新規作成")
    ]
...
```

これで「/admin/create」を表示出来るようになりましたが、まだPOSTハンドラーが用意出来ていません。

このハンドラーを定義する前に、`App`モジュールにもう1つヘルパー関数を追加します：

```fsharp
let bindToForm form handler =
    bindReq (bindForm form) handler BAD_REQUEST
```

このコード用に以下のモジュールをオープンする必要があります：

- `Suave.Form`
- `Suave.Http.RequestErrors`
- `Suave.Model.Binding`

`bindToForm`が行っていることは以下の通りです：

- 第1引数として`Form<'a>`型のフォームを受け取ります
- 第2引数として`'a -> WebPart`型のハンドラーを受け取ります
- フォームのフィールドが適切に設定されている、つまり関連する型として適切に解析可能、かつ`Form`モジュールで定義されているすべての`Prop`を含んでいる受信リクエストを受け取ると、`'a`型の複数の値に対して`handler`引数が適用されます。
- 上記に該当しない場合、何が不正だったのかを示す情報とともに、「不正なリクエスト」を表すHTTPステータスコード 400が返されます。

アルバムを作成出来るようにするには、さらに2つの手順が必要です。

まず`Db`モジュールに`createAlbum`を追加します(作成したアルバムはその後使用しないので、`ignore`関数へ渡しています)：

```fsharp
let createAlbum (artistId, genreId, price, title) (ctx : DbContext) =
    ctx.``[dbo].[Albums]``.Create(artistId, genreId, price, title) |> ignore
    ctx.SubmitUpdates()
```

そして`createAlbum` WebPartにPOST用のハンドラーを追加します：

```fsharp
choose [
        GET >>= ...

        POST >>= bindToForm Form.album (fun form ->
            Db.createAlbum (int form.ArtistId, int form.GenreId, form.Price, form.Title) ctx
            Redirection.FOUND Path.Admin.manage)
    ]
```
