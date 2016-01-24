# AJAXでの削除

アルバムをカートに追加して、アルバムの総数をナビゲーションメニューと`cart`の概要ページで確認出来るようになったので、次はカートからアルバムを削除出来るようにしましょう。
これはアプリケーションへAJAXの機能を組み込む絶好のチャンスです。
そこで、jQueryを使用してカートから選択したアルバムを削除し、カートのビューを更新するような単純なJavaScriptファイルを作成します。

jQueryを[こちら](https://jquery.com/download/)からダウンロードして、プロジェクトに追加します(圧縮/軽量版で構いません)。
なおプロパティで「出力ディレクトリにコピー」を必ず設定しておきます。

そしてプロジェクトに`script.js`というJSファイルを追加して、以下のコードを記述します：

```js
$('.removeFromCart').click(function () {
    var albumId = $(this).attr("data-id");
    var albumTitle = $(this).closest('tr').find('td:first-child > a').html();
    var $cartNav = $('#navlist').find('a[href="/cart"]');
    var count = parseInt($cartNav.html().match(/\d+/));

    $.post("/cart/remove/" + albumId, function (data) {
        $('#main').html(data);
        $('#update-message').html(albumTitle + ' がショッピングカートから削除されました。');
        $cartNav.html('Cart (' + (count - 1) + ')');
    });
});
```

コード自体の詳細についてはあまり言及しませんが、重要なポイントは以下の通りです：

- `removeFromCart`要素に対して、毎回クリックイベントを購読します
- 以下の情報を解析します：
    - アルバムのID
    - アルバムのタイトル
    - カート内のアルバムの個数
- 「/cart/remove」宛にPOSTリクエストを送信します
- POSTが正常に完了した場合、以下を更新します：
    - コンテナ要素のHTML
    - どのアルバムが削除されたのかを通知するためのメッセージ
    - アルバムの個数を減らしたナビゲーションメニュー

`update-message`のdivを`nonEmptyCart`ビューのテーブル直前に追加する必要もあります：

```fsharp
    divId "update-message" [text " "]
```

HTMLマークアップとして、空のdiv要素を用意することが出来ないので、意図的に空ではない文字列を指定しています。
jQueryと`script.js`ファイルは`nonEmptyCart`の最後、テーブルの直後で追加するようにします：

```fsharp
scriptAttr [ "type", "text/javascript"; " src", "/jquery-2.1.4.min.js" ] [ text "" ]
scriptAttr [ "type", "text/javascript"; " src", "/script.js" ] [ text "" ]
```

また、「js」ファイルを参照出来るようにハンドラーを変更する必要もあります：

```fsharp
pathRegex "(.*)\.(css|png|gif|js)" >=> Files.browseHome
```

追加したスクリプトはまだハンドラーがないルートへアクセスしようとします。
まず、先に`Db`モジュールで`removeFromCart`を追加します：

```fsharp
let removeFromCart (cart : Cart) albumId (ctx : DbContext) = 
    cart.Count <- cart.Count - 1
    if cart.Count = 0 then cart.Delete()
    ctx.SubmitUpdates()
```

そして`App`モジュールに`removeFromCart`ハンドラーを追加します：

```fsharp
let removeFromCart albumId =
    session (function
    | NoSession -> never
    | UserLoggedOn { Username = cartId } | CartIdOnly cartId ->
        let ctx = Db.getContext()
        match Db.getCart cartId albumId ctx with
        | Some cart -> 
            Db.removeFromCart cart albumId ctx
            Db.getCartsDetails cartId ctx |> View.cart |> Html.flatten |> Html.xmlToString |> OK
        | None -> 
            never)
```

最後に`choose`WebPartでハンドラーとルートをマッピングします：

```fsharp
pathScan Path.Cart.removeAlbum removeFromCart
```

`removeFromCart` WebPartにいくつか補足しておきます：

- このハンドラーは`NoSession`の場合には呼び出されるべきではありません。`never`は不要なリクエストを防ぐためのものです。
- カート内に存在しない`albumId`に対して`removeFromCart`を呼び出した場合は常に同じ動作になります(`Db.getCart`が`None`を返します)
- 適切なカートが見つかると、`Db.removeFromCart`を呼び出して、
- インラインのHTML要素が返されます。なおここでは以前と異なり、`html`ヘルパー関数を経由していない点に注意してください。その代わり、`script.js`がAJAXによって「main」のdiv要素を更新出来るよう、部分的な要素を返しています。

カートの機能についてはほとんど網羅できました。
最後に1つだけ残ったものがあります：
ユーザーがカートにアルバムを追加した後、ログインした場合には何が起きるべきなのでしょう？
