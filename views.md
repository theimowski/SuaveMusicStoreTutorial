# Views

We've seen how to define basic routing in a Suave application. 
In this section we'll see how we can deal with returning good looking HTML markup in a HTTP response.
Templating HTML views is quite a big topic itself, that we don't want to go into much details about.
Keep in mind that the concept can be approached in many different ways, and the way presented here is not the only proper way of rendering HTML views.
Having said that, I hope you'll still find the following implementation concise and easy to understand.
In this application we'll use server-side HTML templating with the help of a separate Suave package called `Suave.Experimental`.

> Note: As of the time of writing, `Suave.Experimental` is a separate package. It's likely that next releases of the package will include breaking changes. It's also possible that the modules we're going to use from within the package will be extracted to the core Suave package.

To use the package, we need to take a dependency on the following NuGet:
```install-package Suave.Experimental -version 0.28.1```

Before we start defining views, let's organize our `App.fs` source file by adding following line at the beginning of the file:

```fsharp
module SuaveMusicStore.App

open Suave
```

The line means that whatever we define in the file will be placed in `SuaveMusicStore.App` module.
Read [here](http://fsharpforfunandprofit.com/posts/recipe-part3/) for more info about organizing and structuring F# code.
Now let's add a new file `View.fs` to the project just before the `App.fs` file and place the following module definition at the very top:

```fsharp
module SuaveMusicStore.View
```

We'll follow this convention throughout the tutorial to have a clear understanding of the project structure.

> Note: It's very important that the `View.fs` file comes before `App.fs`. F# compiler requires the referenced items to be defined before their usage. At first glance, that might seem like a big drawback, however after a while you start realizing that you can have much better control of your dependencies. Read the [following](http://fsharpforfunandprofit.com/posts/cyclic-dependencies/) for further benefits of lack of cyclic dependencies in F# project.








It's time to replace plain text placeholders in containers with meaningful content.
First, define `h2` in `View` module to output HTML header of level 2:

```fsharp
let h2 s = tag "h2" [] (text s)
```

and replace `text` with new `h2` in each of 4 containers.

We'd like the "/store" route to output hyperlinks to all genres in our Music Store.
Let's add a helper function in `Path` module, that will be responsible for formatting HTTP url with a key-value parameter:

```fsharp
let withParam (key,value) path = sprintf "%s?%s=%s" path key value
```

The `withParam` function takes a tuple `(key,value)` as its first argument, `path` as the second and returns properly formatted url.
A tuple (or a pair) is a widely used structure in F#. It allows us to group two values into one in an easy manner. 
Syntax for creating a tuple is following: `(item1, item2)` - this might look like a standard parameter passing in many other languages including C#.
Follow [this link](http://fsharpforfunandprofit.com/posts/tuples/) to learn more about tuples.

Add also a string key for the url parameter "/store/browse" in `Path.Store` module:

```fsharp
    let browseKey = "genre"
```

We'll use it in `App` module:

```fsharp
    match r.queryParam Path.Store.browseKey with
    ...
```

Now, add the following for working with unordered list (`ul`) and list item (`li`) elements in HTML:

```fsharp
let ul xml = tag "ul" [] (flatten xml)
let li = tag "li" []
```

`flatten` takes a list of `Xml` and "flattens" it into a single `Xml` object model.
The actual container for `store` can now look like following:

```fsharp
let store genres = [
    h2 "Browse Genres"
    p [
        text (sprintf "Select from %d genres:" (List.length genres))
    ]
    ul [
        for g in genres -> 
            li (aHref (Path.Store.browse |> Path.withParam (Path.Store.browseKey, g)) (text g))
    ]
]
```

Things worth commenting in above snippet:

- `store` now takes a list of genres (again the type is inferred by the compiler)
- the `[ for g in genres -> ... ]` syntax is known as "list comprehension". Here we map every genre string from `genres` to a list item
- `aHref` inside list item points to the `Path.Store.browse` url with "genre" parameter - we use the `Path.withParam` function defined earlier

To use `View.store` from `App` module, let's simply pass a hardcoded list for `genres` like following:

```fsharp
    path Path.Store.overview >>= html (View.store ["Rock"; "Disco"; "Pop"])
```

Here is what the solution looks like up to this point: [Tag - View](https://github.com/theimowski/SuaveMusicStore/tree/view)