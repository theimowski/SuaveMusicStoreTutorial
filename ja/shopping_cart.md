# ショッピングカート

お店にカートが無いなんてことがありえますか？
ユーザーがストア内で買い物をする際に、アルバムをカートへ追加出来るようにしましょう。
そこで、まず`Path`モジュールに新しいルートを追加します：

```fsharp
module Cart =
    let overview = "/cart"
    let addAlbum : IntPath = "/cart/add/%d"
    let removeAlbum : IntPath = "/cart/remove/%d"
```

`View`に着手する前に、まずはデータベースビュー`CartDetails`に対する新しい省略型を追加しておきます。
`CartDetails`は`Cart`と`Album`をアルバムタイトルと価格で連結したビューです。

```fsharp
type CartDetails = DbContext.``[dbo].[CartDetails]Entity``
```
