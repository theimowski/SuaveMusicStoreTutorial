CSS
===

いよいよHTMLマークアップにCSSを適用することにしましょう。
このチュートリアルはWebデザインのチュートリアルではないので、
スタイルの詳細は割愛します。
最終的なスタイルシートは [こちら][css] からダウンロードできます。
プロジェクトに`Site.css`を追加して、`出力ディレクトリにコピー` の設定を
`新しい場合にはコピーする` に忘れずに変更してください。

このスタイルシートをHTMLに組み込むために、以下のコードを
`View` に追加します：

````fsharp
let cssLink href = linkAttr [ "href", href; " rel", "stylesheet"; " type", "text/css" ]

let index =
    html [
        head [
            title "Suave Music Store"
            cssLink "/Site.css"
        ]
    ...
````

これで、CSSを指定する`href`属性を持ったlinkタグをHTMLとして出力出来るようになります。

サイトにスタイルシートを適用するより前に、やるべき作業がまだ2つ残っています。

CSSファイルをページに追加しようとすると、ブラウザは特定のURLに
再度アクセスします。一方で、これまで作成したメインの`WebPart`には
このCSSファイルを返すためのハンドラが登録されていません。
そのため、`WebPart`の`choose`として、以下のルートを追加する必要があります：

````fsharp
pathRegex "(.*)\.css" >=> Files.browseHome
````

`pathRegex` の `WebPart` は、受信したリクエストが特定のパターンと一致した場合に
`Some` を返します。
`Some` が返されると、続けて `Files.browseHome` のWebPartが適用されます。
`Files.browseHome` はSuaveから提供されている`WebPart`で、
アプリケーションのルートディレクトリにある静的ファイルを応答本文として返します。

CSSでは`logo.png`というアセットファイルを使用していますが、
このファイルは[こちら][logo]からダウンロード出来ます。

`logo.png` をプロジェクトに追加して、同じく`出力ディレクトリにコピー`の設定を
`新しい場合にはコピーする`に設定します。
ブラウザがこの画像ファイルを描画しようとする時には、やはりサーバーに対して
GETリクエストを送信するため、上の正規表現ルートを変更して`.png`ファイルも
返せるようにします：

````fsharp
pathRegex "(.*)\.(css|png)" >=> Files.browseHome
````

以上でHTMLにスタイルシートを適用出来るようになりました。

[css]: https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/Site.css
[logo]: https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/logo.png

