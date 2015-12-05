# ユーザー用パーシャルビュー

サイトを訪れたユーザーが自分でユーザー認証出来た方がよいと思いませんか。
そこで、ナビゲーションメニューの隣にユーザー用のパーシャルビューを追加する事にしましょう。
よくあるeコマース用サイトと同じく、ユーザーがログインするとユーザー名とともに「ログアウト」のメニューが表示されるようにします。
ログインしていない場合には「ログイン」のリンクが表示されるようにします。
まず`Path`モジュールを開いて、`Account`サブモジュール内に`logon`と`logoff`のルートを追加します：

```fsharp
module Account =
    let logon = "/account/logon"
    let logoff = "/account/logoff"
```

次に`View`モジュールで`partUser`を定義します：

```fsharp
let partUser (user : string option) = 
    divId "part-user" [
        match user with
        | Some user -> 
            yield text (sprintf "ユーザー %s としてログインしています：" user)
            yield aHref Path.Account.logoff (text "ログアウト")
        | None ->
            yield aHref Path.Account.logon (text "ログイン")
    ]
```

> 注釈：パターンマッチ内のコードなので`yield`キーワードは必須です。

そして「header」の`div`に追加します：

```fsharp
divId "header" [
    h1 (aHref Path.home (text "F# Suave Music Store"))
    partNav
    partUser (None)
]
```

`partUser`の唯一の引数は、省略可能なユーザー名です。
もし値がある場合、ユーザーは認証済みだということを表します。
ここではまだどのユーザーもログインしていないという想定なので、`None`を指定して`partUser`を呼んでいます。
