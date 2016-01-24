# ログイン用ハンドラー

POSTハンドラーはもっと複雑です。
まず簡単なところから始めるとして、受け取った認証情報をデータベースに問い合わせて(`Db`モジュールの機能として)検証するロジックから実装しましょう：

```fsharp
let validateUser (username, password) (ctx : DbContext) : User option =
    query {
        for user in ctx.``[dbo].[Users]`` do
            where (user.UserName = username && user.Password = password)
            select user
    } |> firstOrNone
```

このコードでは省略型`User`を使用しています：

```fsharp
type User = DbContext.``[dbo].[Users]Entity``
```

そして`App`モジュールに2つの`open`ステートメントを追加します：

```fsharp
open System
...
open Suave.State.CookieStateStore
```

いくつかのヘルパー関数も追加します：

```fsharp
let passHash (pass: string) =
    use sha = Security.Cryptography.SHA256.Create()
    Text.Encoding.UTF8.GetBytes(pass)
    |> sha.ComputeHash
    |> Array.map (fun b -> b.ToString("x2"))
    |> String.concat ""

let session = statefulForSession

let sessionStore setF = context (fun x ->
    match HttpContext.state x with
    | Some state -> setF state
    | None -> never)

let returnPathOrHome = 
    request (fun x -> 
        let path = 
            match (x.queryParam "returnPath") with
            | Choice1Of2 path -> path
            | _ -> Path.home
        Redirection.FOUND path)
```

以下補足です：

- `passHash` は `string -> string` 型の関数で、受け取った文字列を元にSHA256ハッシュを生成し、16進数に変換します。この値がユーザーのパスワードとしてデータベースに登録されることになります。
- `session` は今のところ単にSuaveの`statefulForSession`の省略型で、ブラウジングセッションの状態を初期化します。しかしすぐ後で`session`に引数を追加する予定なので、あえてこのような定義になっているというわけです。
- `sessionStore` は高階関数で、引数に`setF`を取ります。これはセッションストアの値を読み書きするための関数です。
- `returnPathOrHome` はURLからクエリ引数「returnPath」を取り出せるかどうか試す関数で、この引数が見つかった場合にはそのパスへとリダイレクトします。「returnPath」が見つからない場合はホームへ戻る事になります。

ではようやく`logon`のPOSTハンドラーという難敵に立ち向かうことにしましょう：

```fsharp
let logon =
    choose [
        GET >=> (View.logon |> html)
        POST >=> bindToForm Form.logon (fun form ->
            let ctx = Db.getContext()
            let (Password password) = form.Password
            match Db.validateUser(form.Username, passHash password) ctx with
            | Some user ->
                    Auth.authenticated Cookie.CookieLife.Session false 
                    >=> session
                    >=> sessionStore (fun store ->
                        store.set "username" user.UserName
                        >=> store.set "role" user.Role)
                    >=> returnPathOrHome
            | _ ->
                never
        )
    ]
```

そんなにはひどくないですよね。
まず`Form.logon`をバインドします。
つまり不正なリクエストを受け取った場合には、不正なリクエストを示すエラーコード400が`bindToForm`によって返されます。
一方、誰かがおとなしくログインフォームに正しい値を入力した場合には、データベースに問い合わせて、特定のパスワードを持った特定のユーザーが居るかどうかをチェックします。
なおフォームの結果からパスワード文字列を取得する際に、パターンマッチ(`let (Password password) = form.Password`)を使用している点に注意してください。
`Db.validateUser`が`Some user`を返した場合、ユーザーの状態を反映しつつ、適切な場所へ遷移させるために4つのWebPartを組み合わせています。
まず`Auth.authenticated`はセッションが終わるまで有効な特定のクッキーを設定します。
2番目の引数(`false`)はCookieが「HTTPS専用(HttpsOnly)」ではないことを指定しています。
そしてその結果を`session`にバインドしています。
これは既に説明した通り、ユーザーのセッション状態をセットアップするためのものです。
続いて、セッションストアに「username」と「role」という2つの値を書き込んでいます。
最後に、`returnPathOrHome`へとバインドしています。
この関数がどのように役立つのかという説明は後ほどします。

気づいた方もいると思いますが、`Db.validateUser`がNoneを返した場合、上のコードでは「Not found」が返されるようになっています。
これはNoneにマッチした場合に、一時的に`never`を返すようにしているためです。
理想的には、フォームの隣にバリデーション用のメッセージなどを表示しておくとよいでしょう。
そこで、`View.logon`に`msg`引数を追加します：

```fsharp
let logon msg = [
    h2 "ログイン"
    p [
        text "名前とパスワードを入力してください。"
    ]

    divId "logon-message" [
        text msg
    ]
...
```

そうすると以下のように、2通りの方法で呼び出せるようになります：

```fsharp
GET >=> (View.logon "" |> html)

...

View.logon "ユーザー名またはパスワードが間違っています。" |> html
```

1つめのコードは`logon`のGETハンドラー用で、2つめのコードは入力された認証情報が間違っていた場合に使用されるものです。
