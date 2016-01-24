# 登録用ハンドラー

`Path.Account`にエントリーを追加しましょう：

```fsharp
let register = "/account/register"
```

`App`モジュールで登録用のGETハンドラーを実装します：

```fsharp
let register =
    choose [
        GET >=> (View.register "" |> html)
    ]
```

```fsharp
path Path.Account.register >=> register
```

そして`View.logon`にリンクを追加します：

```fsharp
let logon msg = [
    h2 "ログイン"
    p [
        text "名前とパスワードを入力してください。"
        text "アカウントが未登録の方は "
        aHref Path.Account.register (text "登録")
        text " をお願いします。"
    ]

    ...
```

これで登録用フォームに遷移出来るようになりました。
POSTハンドラーを実装するために、まず`Db`モジュールへ必要な関数を追加しましょう：

```fsharp
let getUser username (ctx : DbContext) : User option = 
    query {
        for user in ctx.``[dbo].[Users]`` do
        where (user.UserName = username)
        select user
    } |> firstOrNone
```

```fsharp
let newUser (username, password, email) (ctx : DbContext) =
    let user = ctx.``[dbo].[Users]``.Create(email, password, "user", username)
    ctx.SubmitUpdates()
    user
```

`getUser` は指定された`username`が既にデータベースに登録されているかどうかをチェックします。同じ`username`を持つユーザーは受け付けないようにします。
`newUser`は単純な関数で、新しいユーザーを作成してそれを返します。
なお新しいユーザーのロールとしては「user」と決め打ちしています。
こうすることによって、管理者ロールと区別出来るようになります。

ユーザー登録が正しく完了した後は、一度ユーザー認証を行います。
別の言い方をすると、正しくログイン出来た場合と同じロジックを走らせています。
実際のアプリケーションでは、おそらくメール認証メカニズムを導入することになるのでしょうが、今回は単純のために省略します。
ログインのPOSTハンドラーを再利用出来るようにするために、機能を別の関数に分けておきます：

```fsharp
let authenticateUser (user : Db.User) =
    Auth.authenticated Cookie.CookieLife.Session false 
    >=> session (function
        | CartIdOnly cartId ->
            let ctx = Db.getContext()
            Db.upgradeCarts (cartId, user.UserName) ctx
            sessionStore (fun store -> store.set "cartid" "")
        | _ -> succeed)
    >=> sessionStore (fun store ->
        store.set "username" user.UserName
        >=> store.set "role" user.Role)
    >=> returnPathOrHome
```

分離後の`logon`のPOSTハンドラーは以下のようになります：

```fsharp
match Db.validateUser(form.Username, passHash password) ctx with
| Some user ->
    authenticateUser user
| _ ->
    View.logon "ユーザー名またはパスワードが間違っています。" |> html
```

最終的に、登録用のハンドラーは以下のような実装になります：

```fsharp
let register =
    choose [
        GET >=> (View.register "" |> html)
        POST >=> bindToForm Form.register (fun form ->
            let ctx = Db.getContext()
            match Db.getUser form.Username ctx with
            | Some existing -> 
                View.register "申し訳ありませんが、同名のユーザーが既に登録されています。別の名前を使用してください。" |> html
            | None ->
                let (Password password) = form.Password
                let email = form.Email.Address
                let user = Db.newUser (form.Username, passHash password, email) ctx
                authenticateUser user
        )
    ]
```

POSTの部分に対する補足です：

- リクエストを検証するために`Form.register`へバインドしています
- 入力された`username`が既に存在するかどうかチェックします
- ユーザーが登録済みの場合、エラーメッセージとともに再度`View.register`のフォームを表示します
- そうでなければフォームのフィールドの値を読み取り、ユーザーを新規作成した後、`authenticateUser`関数を呼びます

これで登録機能は完成です。新しいユーザーが買い物をする準備が整いました。
ちょっと待ってください。カートにアルバムが追加出来るようになったものの、どうやって購入すればいいのでしょう？
ああそうでした。まだ実装出来ていませんね。
