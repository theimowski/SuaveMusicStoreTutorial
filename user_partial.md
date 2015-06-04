# User partial

It would be good if a visitor to our site could authenticate himself.
To help him with that, we'll put a user partial view next to the navigation menu.
Just as in every other e-commerce website, if a user is logged in, he'll be shown his name and a "Log off" link.
Otherwise, we'll just display a "Log on" link.
First, open up the `Path` module and define routes for `logon` and `logoff` in `Account` submodule:

```fsharp
module Account =
    let logon = "/account/logon"
    let logoff = "/account/logoff"
```

Next, define `partUser` in the `View` module:

```fsharp
let partUser (user : string option) = 
    divId "part-user" [
        match user with
        | Some user -> 
            yield text (sprintf "Logged on as %s, " user)
            yield aHref Path.Account.logoff (text "Log off")
        | None ->
            yield aHref Path.Account.logon (text "Log on")
    ]
```

> Note: Because we're inside pattern matching, the `yield` keyword is mandatory here.

and include it in "header" `div` as well"

```fsharp
divId "header" [
    h1 (aHref Path.home (text "F# Suave Music Store"))
    partNav
    partUser (None)
]
```

The only argument to `partUser` is an optional username - if it exists, then the user is authenticated.
For now, we assume no user is logged on, thus we hardcode the `None` in call to `partUser`.