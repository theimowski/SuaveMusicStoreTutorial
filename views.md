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

```
module SuaveMusicStore.App

open Suave
...
```

The line means that whatever we define in the file will be placed in `SuaveMusicStore.App` module.
Read [here](http://fsharpforfunandprofit.com/posts/recipe-part3/) for more info about organizing and structuring F# code.
Now let's add a new file `View.fs` to the project just before the `App.fs` file and place the following module definition at the very top:

```
module SuaveMusicStore.View
```

We'll follow this convention throughout the tutorial to have a clear understanding of the project structure.

> Note: It's very important that the `View.fs` file comes before `App.fs`. F# compiler requires the referenced items to be defined before their usage. At first glance, that might seem like a big drawback, however after a while you start realizing that you can have much better control of your dependencies. Read the [following](http://fsharpforfunandprofit.com/posts/cyclic-dependencies/) for further benefits of lack of cyclic dependencies in F# project.

With the `View.fs` file in place, let's add our first view:

```
module SuaveMusicStore.View

open Suave.Html

let divId id = divAttr ["id", id]
let h1 xml = tag "h1" [] xml
let aHref href = tag "a" ["href", href]

let index = 
    html [
        head [
            title "Suave Music Store"
        ]

        body [
            divId "header" [
                h1 (aHref "/" (text "F# Suave Music Store"))
            ]

            divId "footer" [
                text "built with "
                aHref "http://fsharp.org" (text "F#")
                text " and "
                aHref "http://suave.io" (text "Suave.IO")
            ]
        ]
    ]
    |> xmlToString
```

This will serve as a common layout in our application.
A few remarks about the above snippet:

- open `Suave.Html` module, for functions to generate HTML markup.
- 3 helper functions come next:
    - `divId` which appends "div" element with a string attribute `id`
    - `h1` which takes inner markup to generate HTML header level 1.
    - `aHref` which takes string attribute `href` and inner HTML markup to output "a" element.
- `tag` function comes from Suave. It's of type `string -> Attribute list -> Xml -> Xml`. First arg is name of the HTML element, second - a list of attributes, and third - inner markup
- `Xml` is an internal Suave type holding object model for the HTML markup
- `index` is our representation of HTML markup. 
- `html` is a function that takes a list of other tags as its argument. So do `head` and `body`.
- `text` serves outputting plain text into an HTML element.
- `xmlToString` transforms the object model into the resulting raw HTML string.

> Note: `tag` function from Suave takes 3 arguments ().
> We've defined the `aHref` function by invoking `tag` with only 2 arguments, and the compiler is perfectly happy with that - Why?
> This concept is called "partial application", and allows us to invoke a function by passing only a subset of arguments.
> When we invoke a function with only a subset of arguments, the function will return another function that will expect the rest of arguments.
> In our case this means `aHref` is of type `string -> Xml -> Xml`, so the second "hidden" argument to `aHref` is of type `Xml`.
> Read [here](http://fsharpforfunandprofit.com/posts/partial-application/) for more info about partial application.

We can see usage of the "pipe" operator `|>` in the above code. 
The operator might look familiar if you have some UNIX background.
In F#, the `|>` operator basically means: take the value on the left side and apply it to the function on the right side of the operator.
In this very case it simply means: invoke the `xmlToString` function on the HTML object model.

Let's test the `index` view in our `App.fs`:
```
    path "/" >>= (OK View.index)
```

If you navigate to the root url of the application, you should see that proper HTML has been returned.

Before we move on to defining views for the rest of the application, let's introduce one more file - `Path.fs` and insert it **before** `View.fs`:

```
module SuaveMusicStore.Path

type IntPath = PrintfFormat<(int -> string),unit,string,string,int>

let home = "/"

module Store =
    let overview = "/store"
    let browse = "/store/browse"
    let details : IntPath = "/store/details/%d"
```

The module will contain all valid routes in our application.
We'll keep them here in one place, in order to be able to reuse both in `App` and `View` modules.
Thanks to that, we will minimize the risk of a typo in our `View` module.
We defined a submodule called `Store` in order to group routes related to one functionality - later in the tutorial we'll have a few more submodules, each of them reflecting a specific set of functionality of the application.

The `IntPath` type alias that we declared will let use our routes in conjunction with static-typed Suave routes (`pathScan` in `App` module). 
We don't need to fully understand the signature of this type, for now we can think of it as a route parametrized with integer value.
And indeed, we annotated the `details` route with this type, so that the compiler treats this value *specially*. 
We'll see in a moment how we can use `details` in `App` and `View` modules, with the advantage of static typing.

Let's use the routes from `Path` module in our `App`:

```
let webPart = 
    choose [
        path Path.home >>= (OK View.index)
        path Path.Store.overview >>= (OK "Store")
        path Path.Store.browse >>= browse
        pathScan Path.Store.details (fun id -> OK (sprintf "Details %d" id))
    ]
```

as well as in our `View` for `aHref` to `home`:

```
    divId "header" [
        h1 (aHref Path.home (text "F# Suave Music Store"))
    ]
```

Note, that in `App` module we still benefit from the static typed routes feature that Suave gives us - the `id` parameter is inferred by the compiler to be of integer type.
If you're not familiar with type inference mechanism, you can follow up [this link](http://fsharpforfunandprofit.com/posts/type-inference/).

It's high time we added some CSS styles to our HTML markup.
We'll not deep-dive into the details about the styles itself, as this is not a tutorial on Web Design.
The stylesheet can be downloaded [from here](https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/Site.css) in its final shape.
Add the `Site.css` stylesheet to the project, and don't forget to set the `Copy To Output Directory` property to `Copy If Newer`.

In order to include the stylesheet in our HTML markup, let's add the following to our `View`:

```
let cssLink href = linkAttr [ "href", href; " rel", "stylesheet"; " type", "text/css" ]

let index = 
    html [
        head [
            title "Suave Music Store"
            cssLink "/Site.css"
        ]
...
```

This enables us to output the link HTML element with `href` attribute pointing to the CSS stylesheet.

There's two more things before we can see the styles applied on our site.

A browser, when asked to include a CSS file, sends back a request to the server with the given url.
If we have a look at our main `WebPart` we'll notice that there's really no handler capable of serving this file.
That's why we need to add another alternative to our `choose` `WebPart`:

```
    pathRegex "(.*)\.css" >>= Files.browseHome
```

The `pathRegex` `WebPart` returns `Some` if an incoming request concerns path that matches the regular expression pattern. 
If that's the case, the `Files.browseHome` WebPart will be applied.
`(.*)\.css` pattern matches every file with `.css` extension.
`Files.browseHome` is a `WebPart` from Suave that serves static files from the root application directory.

The CSS depends on `logo.png` asset, which can be downloaded from [here](https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/logo.png).

Add `logo.png` to the project, and again don't forget to select `Copy If Newer` for `Copy To Output Directory` property for the asset.
Again, when browser wants to render an image asset, it needs to GET it from the server, so we need to extend our regular expression to allow browsing of `.png` files as well:

```
    pathRegex "(.*)\.(css|png)" >>= Files.browseHome
```

Now you should be able to see the styles applied to our HTML markup.

With styles in place, let's get our hands on extracting a shared layout for all future views to come.
Start by adding `container` parameter to `index` in `View`:

```
let index container = 
    html [
...
```

and div with id "container" just after the div "header":

```
    divId "header" [
        h1 (aHref Path.home (text "F# Suave Music Store"))
    ]

    divId "container" container
```

`index` previously was a constant value, but now became a function taking `container` as parameter.

We can now define actual container for the "home" page:

```
let home = [
    text "Home"
]
```

For now it will only contain plain "Home" text.
Let's also extract a common function for creating WebPart, parametrized with the container itself.
Add to `App` module, just before the `browse` WebPart the following:

```

let html container =
    OK (View.index container)

```

Usage for the home page looks like this:

```
    path Path.home >>= html View.home
```

Next, containers for each valid route in our application can be defined in `View` module:

```
let home = [
    text "Home"
]

let store = [
    text "Store"
]

let browse genre = [
    text (sprintf "Genre: %s" genre)
]

let details id = [
    text (sprintf "Details %d" id)
]
```

Note that both `home` and `store` are constant values, while `browse` and `details` are parametrized with `genre` and `id` respectively.

`html` can be now reused for all 4 views:

```
let browse =
    request (fun r ->
        match r.queryParam "genre" with
        | Choice1Of2 genre -> html (View.browse genre)
        | Choice2Of2 msg -> BAD_REQUEST msg)

let webPart = 
    choose [
        path Path.home >>= html View.home
        path Path.Store.overview >>= html View.store
        path Path.Store.browse >>= browse
        pathScan Path.Store.details (fun id -> html (View.details id))

        pathRegex "(.*)\.(css|png)" >>= Files.browseHome
    ]
```

It's time to replace plain text placeholders in containers with meaningful content.
First, define `h2` in `View` module to output HTML header of level 2:

```
let h2 s = tag "h2" [] (text s)
```

and replace `text` with new `h2` in each of 4 containers.

We'd like the "/store" route to output hyperlinks to all genres in our Music Store.
Let's add a helper function in `Path` module, that will be responsible for formatting HTTP url with a key-value parameter:

```
let withParam (key,value) path = sprintf "%s?%s=%s" path key value
```

The `withParam` function takes a tuple `(key,value)` as its first argument, `path` as the second and returns properly formatted url.
A tuple (or a pair) is a widely used structure in F#. It allows us to group two values into one in an easy manner. 
Syntax for creating a tuple is following: `(item1, item2)` - this might look like a standard parameter passing in many other languages including C#.
Follow [this link](http://fsharpforfunandprofit.com/posts/tuples/) to learn more about tuples.

Add also a string key for the url parameter "/store/browse" in `Path.Store` module:

```
    let browseKey = "genre"
```

We'll use it in `App` module:

```
    match r.queryParam Path.Store.browseKey with
    ...
```

Now, add the following for working with unordered list (`ul`) and list item (`li`) elements in HTML:

```
let ul xml = tag "ul" [] (flatten xml)
let li = tag "li" []
```

`flatten` takes a list of `Xml` and "flattens" it into a single `Xml` object model.
The actual container for `store` can now look like following:

```
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

```
    path Path.Store.overview >>= html (View.store ["Rock"; "Disco"; "Pop"])
```

Here is what the solution looks like up to this point: [Tag - View](https://github.com/theimowski/SuaveMusicStore/tree/view)