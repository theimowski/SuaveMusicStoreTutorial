# 匿名ユーザー用のカート

`App`モジュールの`Session`型に再度手を加えましょう。
ビジネス要件としては、アルバムをカートに追加する際はログイン不要だけれども、購入手続きを始める際にはログインが必須だとします。
ユーザーがせっかく景気よく買い物をしようとしているのに、ログイン用フォームで萎えさせるわけにも行きませんから、これは妥当な要求でしょう。
そこでまず、`Session`型に`CartIdOnly`のケースを追加します。
このケースはまだログインしていないものの、カート内にいくつかアルバムが入っているユーザーを表します：

```fsharp
type Session = 
    | NoSession
    | CartIdOnly of string
    | UserLoggedOn of UserLoggedOnSession
```

`CartIdOnly`にはカートへ初めてアルバムを追加した際に割り当てられるGUIDが含まれます。

`Db`モジュールに戻って`Carts`テーブルの省略型を追加します：

```fsharp
type Cart = DbContext.``[dbo].[Carts]Entity``
```

`Cart`には以下のプロパティがあります：

- CartId：ユーザーがログインしていない場合はGUID、ログイン済みの場合はユーザー名
- AlbumId
- Count
