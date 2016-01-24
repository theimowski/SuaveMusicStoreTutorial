# Register handler

We're now left with proper `Path.Account` entry:

```fsharp
let register = "/account/register"
```

GET handler for registration in `App`:

```fsharp
let register =
    choose [
        GET >=> (View.register "" |> html)
    ]
```

```fsharp
path Path.Account.register >=> register
```

and a direct link from the `View.logon` :

```fsharp
let logon msg = [
    h2 "Log On"
    p [
        text "Please enter your user name and password."
        aHref Path.Account.register (text " Register")
        text " if you don't have an account yet."
    ]

    ...
```

This allows us to navigate to the registration form.
Moving on to implementing the actual POST handler, let's first create necessary functions in `Db` module:

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

`getUser` will be crucial to check if a user with given `username` already exists in the database - we don't want two users with the same `username`.
`newUser` is a simple function that creates and returns the new user.
Note that we hardcode "user" for each new user's role. 
This way, they can be distinguished from admin's role.

After a successful registration, we'd like to authenticate user at once - in other words apply the same logic which happens after successful logon.
In a real application, you'd probably use a confirmation mail mechanism, but for the sake of simplicity we'll skip that.
In order to reuse the logic from logon POST handler, extract a separate function:

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

after extraction, `logon` POST handler looks like this:

```fsharp
match Db.validateUser(form.Username, passHash password) ctx with
| Some user ->
    authenticateUser user
| _ ->
    View.logon "Username or password is invalid." |> html
```

Finally, the full register handler can be implemented following:

```fsharp
let register =
    choose [
        GET >=> (View.register "" |> html)
        POST >=> bindToForm Form.register (fun form ->
            let ctx = Db.getContext()
            match Db.getUser form.Username ctx with
            | Some existing -> 
                View.register "Sorry this username is already taken. Try another one." |> html
            | None ->
                let (Password password) = form.Password
                let email = form.Email.Address
                let user = Db.newUser (form.Username, passHash password, email) ctx
                authenticateUser user
        )
    ]
```

Comments for POST part:

- bind to `Form.register` to validate the request
- check if a user with given `username` already exists
- if that's the case then show the `View.register` form again with a proper error message
- otherwise read the form fields' values, create new user and invoke the `authenticateUser` function

This concludes register feature - we're now set for new customers to do the shopping.
Wait a second, they can put albums to the cart, but how do they checkout?
Ah yes, we haven't implemented that yet.
