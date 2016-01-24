# 管理者ログイン

「/admin」ハンドラー用の認証機能をセットアップするためにはもう少しヘルパー関数が必要です。
`App`モジュールに以下を追加します：

```fsharp
open Suave.Cookie

...

let reset =
    unsetPair Auth.SessionAuthCookie
    >=> unsetPair StateCookie
    >=> Redirection.FOUND Path.home

let redirectWithReturnPath redirection =
    request (fun x ->
        let path = x.url.AbsolutePath
        Redirection.FOUND (redirection |> Path.withParam ("returnPath", path)))


...

let loggedOn f_success =
    Auth.authenticate
        Cookie.CookieLife.Session
        false
        (fun () -> Choice2Of2(redirectWithReturnPath Path.Account.logon))
        (fun _ -> Choice2Of2 reset)
        f_success

let admin f_success =
    loggedOn (session (function
        | UserLoggedOn { Role = "admin" } -> f_success
        | UserLoggedOn _ -> FORBIDDEN "管理者専用です"
        | _ -> UNAUTHORIZED "ログインしていません"
    ))
```

以下補足です：

- `reset` は認証およびユーザー状態を表すクッキー値をクリーンアップした後、ホームページへリダイレクトさせるWebPartです。ユーザーがログアウトした際に使用することになります。
- `redirectWithReturnPath` はクエリ引数「returnPath」が追加された特定のURLへユーザーをリダイレクトします。認証が必要な特定のアクションを行う際に、ユーザーをログインページへリダイレクトさせる際に使用します。
- `loggedOn` はユーザーが認証されていた場合に実行される関数 `f_success` を引数に取るWebPartです。この関数ではSuaveライブラリの関数`Auth.authenticate`を呼び出していて、この関数の最後の引数に `f_success` が渡されています。その他の引数はそれぞれ以下の通りです：
    - `CookieLife`には今回は`Session`を指定します
    - `httpsOnly`にはこのチュートリアルではHTTPSを使用しないためfalseを指定します(ただしSuaveはHTTPSをサポートしています)
    - 第3引数はcookieが見つからなかった場合に実行される関数です。この時に、「returnPath」をしつつユーザーをログインページへリダイレクトするようにします。
    - 第4引数は復号化エラーが発生した場合に実行される関数です。実際のアプリケーションにおいては、不正なリクエストを受け取ったことを示すものではありますが、今回は単に状態をリセットするだけにしています。このようにしておくことで、開発中にいつでもサーバーを再起動することが出来るようになり、なおかつ古いサーバーのキーで暗号化されたクッキーの値をブラウザが送りつけてきたとしても問題無くアクセスさせることができます(Suaveはサーバーを再起動するたびにサーバーキーを新しく生成します)。
- `admin` も同じく `f_success` WebPartを引数に取ります。ここでは`session`とインライン関数を使用して`loggedOn`を呼び出しています。興味深いのは`session`に渡している関数です：
    - `function | ... -> ` という構文は単に `match x with | ... -> ` という構文の省略バージョンですが、引数`x`が省略されています。また`x`の型は`Session`です。
    - 1つめのパターンはパターンマッチのテクニックの見せ所です。`f_success`はユーザーがログイン中で、なおかつユーザーのロール(Role)が「admin」だった場合にのみ実行されます(後ほど「admin」と「user」のロールを区別します)
    - 2つめのパターンはユーザーがログイン中だけれども、ロールが異なる場合にマッチし、403 Forbiddenのエラーが返されます。
    - 最後の「その他」(`_`) については既に`loggedOn`の内側にいるわけなので、通常は起こりえません。単に安全のために用意しています。

結構な量でしたが、学ぶべきことも多くありました。そしてようやく「/admin」アクションを保護できます：

```fsharp
path Path.Admin.manage >=> admin manage
path Path.Admin.createAlbum >=> admin createAlbum
pathScan Path.Admin.editAlbum (fun id -> admin (editAlbum id))
pathScan Path.Admin.deleteAlbum (fun id -> admin (deleteAlbum id))
```

「/admin/manage」にアクセスしようとした時に何が起こるか確認してみてください。

ユーザーがログインした場合に備えられるよう、`partUser`を更新する必要もあります(ユーザー名に決め打ちで`None`を指定していたことを思い出してください)。
そこで`View.index`の引数として`partUser`を渡せるようにします：

```fsharp
let index partUser container = 

...

            divId "header" [
                h1 (aHref Path.home (text "F# Suave Music Store"))
                partNav
                partUser
            ]
```

そして`App`モジュールの`html`WebPartでどのユーザーがログインしたのか把握できるようにします：

```fsharp
let html container =
    let result user =
        OK (View.index (View.partUser user) container)
        >=> Writers.setMimeType "text/html; charset=utf-8"

    session (function
    | UserLoggedOn { Username = username } -> result (Some username)
    | _ -> result None)
```

補助関数として、`user`を引数にとる`result`関数を定義しています。
`session`ではユーザーの状態を知ることができます。
実際には常に`result`関数を呼んでいますが、`user`引数の値はユーザーの状態によって異なります。

最後の変更箇所は`logoff`です。メインの`choose`WebPartに追加しましょう：

```fsharp
path Path.Account.logoff >=> reset
```

`logoff` には個別のWebPartを用意する必要がありません。`reset`を再利用できます。

`Suave`ライブラリに用意された認証およびセッション機能を巡る旅はここで終了です。
次の章でもこれらのコンセプトを振り返りますが、ほとんどの実装が再利用可能です。
これまでの変更をまとめると [Tag - auth_and_session](https://github.com/theimowski/SuaveMusicStore/tree/auth_and_session) のようになります。
