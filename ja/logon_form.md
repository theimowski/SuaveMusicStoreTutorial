# ログインフォーム

`logon`のルート用のハンドラーがまだ無いので、追加しなければいけません。
ログイン用のビューは比較的簡単です。
単にユーザー名とパスワードを入力するフォームになっていればいいです。

```fsharp
type Logon = {
    Username : string
    Password : Password
}

let logon : Form<Logon> = Form ([],[])
```

`logon`フォームは`Form`モジュール内で上の通りに定義できます。
`Password`はSuaveライブラリ由来の型で、表示されるHTMLマークアップに影響します(入力した秘密のパスワードを他の誰かに見られたくありませんからね)。

```fsharp
let logon = [
    h2 "ログイン"
    p [
        text "名前とパスワードを入力してください。"
    ]

    renderForm
        { Form = Form.logon
          Fieldsets = 
              [ { Legend = "アカウント情報"
                  Fields = 
                      [ { Label = "ユーザー名"
                          Xml = input (fun f -> <@ f.Username @>) [] }
                        { Label = "パスワード"
                          Xml = input (fun f -> <@ f.Password @>) [] } ] } ]
          SubmitText = "ログイン" }
]
```

約束したとおり、特別な事は何もありません。
`renderForm`の動作については既に説明しましたので、上のコードは先頭で追加情報を表示しているだけの、単なるHTMLフォームであることがわかるはずです。

`logon`のGETハンドラーも非常にシンプルです：

```fsharp
let logon =
    View.logon
    |> html
```

```fsharp
path Path.Account.logon >>= logon
```
