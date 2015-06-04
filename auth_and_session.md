# Auth and Session

In the previous section we succeeded in setting up Create, Update and Delete functionality for albums in the Music Store.
All of these actions are likely to be performed by some kind of shop manager, or administrator.
In fact, `Path` module defines that all the operations are available under "/admin" route.
It would be nice if we could authorize only chosen users to mess with albums in our Store.
That's exactly what we'll do right now.










Up to this point, we should be able to authenticate with "admin" -> "admin" credentials to our application.
This is however not very useful, as there are no handlers that would demand user to be authenticated yet.

To change that, let's define custom types to represent user state:

```fsharp
type UserLoggedOnSession = {
    Username : string
    Role : string
}

type Session = 
    | NoSession
    | UserLoggedOn of UserLoggedOnSession
```

`Session` type is so-called "Discriminated Union" in F#.
It basically means that an instance of `Session` type is either `NoSession` or `UserLoggedOn`, and no other than that.
`of UserLoggedOnSession` is an indicator that there is some data of type `UserLoggedOnSession` related.
Read [here](http://fsharpforfunandprofit.com/posts/discriminated-unions/) for more info on Discriminated Unions. 

On the other hand, `UserLoggedOnSession` is a "Record type".
We can think of Record as a Plain-Old-Class Object, or DTO, or whatever you like.
It has however a number of language built-in features that make it really awesome, including:

- immutability by default
- structural equality by default
- pattern matching
- copy-and-update expression

Again, if you want to learn more about Records, make sure you visit [this](http://fsharpforfunandprofit.com/posts/records/) post.

With these two types, we'll be able to distinguish from when a user is logged on to our application and when he is not.

As stated before, we'll now add a parameter to the `session` function:

```fsharp
let session f = 
    statefulForSession
    >>= context (fun x -> 
        match x |> HttpContext.state with
        | None -> f NoSession
        | Some state ->
            match state.get "username", state.get "role" with
            | Some username, Some role -> f (UserLoggedOn {Username = username; Role = role})
            | _ -> f NoSession)
```

Type of `f` parameter is `Session -> WebPart`.
You guessed it, it means we will be able to do different things including returning different responses, depending on the user session state.
In order to confirm that a user is logged on, session state store must contain both "username" and "role" values.

> Note: We have used a pattern matching on a tuple in the above snippet - we could pass two values separated with comma to the `match` construct, and then pattern match on both values: `Some username, Some role` means both values are present. The latter (`_`) covers all other instances.

The only usage of `session` for now is in the `logon` POST handler - let's adjust it to new version:

```fsharp
...
Auth.authenticated Cookie.CookieLife.Session false 
>>= session (fun _ -> succeed)
>>= sessionStore (fun store ->
...
```

Yes I know, I promised we'll pass something funky to the `session` function, but bear with with me - we will later.
For the moment usage of `session` in `logon` doesn't require any custom action, but we still need to invoke it to "initalize" the user state.

There are a few more helper functions needed before we can set up proper authorization for "/admin" handlers.
Add following to `App` module:

```fsharp
open Suave.Cookie

...

let reset =
    unsetPair Auth.SessionAuthCookie
    >>= unsetPair StateCookie
    >>= Redirection.FOUND Path.home

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
        | UserLoggedOn _ -> FORBIDDEN "Only for admin"
        | _ -> UNAUTHORIZED "Not logged in"
    ))
```

Remarks:

- `reset` is a WebPart to clean up auth and state cookie values, and redirect to home page afterwards. We'll use it for logging user off.
- `redirectWithReturnPath` aims to point user to some url, with the "returnPath" query parameter baked into the url. We'll use it for redirecting user to logon page if specific action requires authentication.
- `loggedOn` takes `f_success` WebPart as argument, which will be applied if user is authenticated. Here we use the library function `Auth.authenticate`, to which `f_success` comes as last parameter. The rest of parameters, starting with first are are respectively:
    - `CookieLife` - `Session` in our case
    - `httpsOnly` - we pass false as we won't cover HTTPS bindings in the tutorial (however Suave does support it).
    - 3rd parameter is a function applied if auth cookie is missing - that's where we want to redirect user to the logon page with a "returnPath"
    - 4th parameter is a function applied if there occurred a decryption error. In real world this could mean a malformed request, however at current stage we'll stick to reseting the state. This way we can re-run the server multiple times during development, without worrying about the browser to pass a cookie value encrypted with stale server key (Suave regenerates a new server key each time it is run).
- `admin` also takes `f_success` WebPart as argument. Here, we invoke `loggedOn` with an inline function using `session`. The interesting part is inside the `session` function:
    - syntax `function | ... -> ` is just a shorter version of `match x with | ... -> ` but the `x` param is implicit here, and `x` is of type `Session`
    - first pattern shows the real power of the pattern matching technique - `f_success` will be applied only if user is logged on, and his Role is "admin" (we'll distinguish between "admin" and "user" roles)
    - second pattern holds if user is logged on but with different role, thus we return 403 Forbidden status code
    - the last "otherwise" (`_`) pattern should never hold, because we're already inside `loggedOn`. We use it anyway, just as a safety net.

That was quite long, but worth it. Finally we're able to guard the "/admin" actions:

```fsharp
path Path.Admin.manage >>= admin manage
path Path.Admin.createAlbum >>= admin createAlbum
pathScan Path.Admin.editAlbum (fun id -> admin (editAlbum id))
pathScan Path.Admin.deleteAlbum (fun id -> admin (deleteAlbum id))
```

Go and have a look what happens when you try to navigate to "/admin/manage" route.

We still need to update the `partUser`, when a user is logged on (remember we hardcoded `None` for username).
To do this, we can pass `partUser` as parameter to `View.index`:

```fsharp
let index partUser container = 

...

            divId "header" [
                h1 (aHref Path.home (text "F# Suave Music Store"))
                partNav
                partUser
            ]
```

and determine whether a user is logged on in the `html` WebPart in `App` module:

```fsharp
let html container =
    let result user =
        OK (View.index (View.partUser user) container)
        >>= Writers.setMimeType "text/html; charset=utf-8"

    session (function
    | UserLoggedOn { Username = username } -> result (Some username)
    | _ -> result None)
```

We declared a sub function `result` which takes the `user` parameter.
`session` can be used to determine user state.
Effectively, we invoke the `result` function always but with `user` argument based on the user state.

The last thing we need to support is `logoff`. In the main `choose` WebPart add:

```fsharp
path Path.Account.logoff >>= reset
```

`logoff` doesn't require separate WebPart, `reset` can be reused instead.

That concludes our journey to Auth and Session features in `Suave` library. 
We'll revisit the concepts in next section, but much of the implementation can be reused.
Code up to this point can be browsed here: [Tag - auth_and_session](https://github.com/theimowski/SuaveMusicStore/tree/auth_and_session)
