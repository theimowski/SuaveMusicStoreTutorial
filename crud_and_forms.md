# CRUD and Forms

With the database in place, we can now move to implementing a management module.
This will be a simple Create, Update, Delete functionality with a grid to display all albums in the store.

Let's start by adding `manage` to our `View` module:

```fsharp
let manage (albums : Db.AlbumDetails list) = [ 
    h2 "Index"
    table [
        yield tr [
            for t in ["Artist";"Title";"Genre";"Price"] -> th [ text t ]
        ]

        for album in albums -> 
        tr [
            for t in [ truncate 25 album.Artist; truncate 25 album.Title; album.Genre; formatDec album.Price ] ->
                td [ text t ]
        ]
    ]
]
```

The view requires a few of new helper functions for table HTML markup:

```fsharp
let table x = tag "table" [] (flatten x)
let th x = tag "th" [] (flatten x)
let tr x = tag "tr" [] (flatten x)
let td x = tag "td" [] (flatten x)
```

as well as a `truncate` function that will ensure our cell content doesn't span over a maximum number of characters:

```fsharp
let truncate k (s : string) =
    if s.Length > k then
        s.Substring(0, k - 3) + "..."
    else s
```

Remarks:

- our HTML table consists of first row (`tr`) containing column headers (`th`) and a set of rows for each album with cells (`td`) to display specific values.
- we used the `yield` keyword for the first time. It is required here because we're using it in conjunction with the `for album in albums ->` list comprehension syntax inside the same list. The rule of thumb is that whenever you use the list comprehension syntax, then you need the `yield` keyword for any other item not contained in the comprehension syntax. This might seem hard to remember, but don't worry - the compiler is helpful here and will issue a warning if you forget the `yield` keyword.
- for the sake of saving a few keystrokes we used a nested list comprehension syntax to output `th`s and `td`s. Again, it's just a matter of taste, and could be also solved by enumerating each element separately

We are going to need to fetch the list of all `AlbumDetail`s from the database. 
For this reason, let's create following query in `Db` module:

```fsharp
let getAlbumsDetails (ctx : DbContext) : AlbumDetails list = 
    ctx.``[dbo].[AlbumDetails]`` |> Seq.toList
```

Now we're ready to define an actual handler to display the list of albums.
Let's add a new sub-module to `Path`:

```fsharp
module Admin =
    let manage = "/admin/manage"
```

The `Admin` sub-module will contain all album management paths or routes if you will.

`manage` WebPart in `App` module can be implemented in following way:

```fsharp
let manage = warbler (fun _ ->
    Db.getContext()
    |> Db.getAlbumsDetails
    |> View.manage
    |> html)
```

and used in the main `choose` WebPart:

```fsharp
    path Path.Admin.manage >>= manage
```

Don't forget about the `warbler` for `manage` WebPart - we don't use an parameters for this WebPart, so we need to prevent it's eager evaluation.

If you navigate to the "/admin/manage" url in the application now, you should be presented the grid with every album in the store.
We can't make any operation on an album yet.
To fix this, let's first add the delete functionality:

```fsharp
let deleteAlbum albumTitle = [
    h2 "Delete Confirmation"
    p [ 
        text "Are you sure you want to delete the album titled"
        br
        strong albumTitle
        text "?"
    ]
    
    form [
        submitInput "Delete"
    ]

    div [
        aHref Path.Admin.manage (text "Back to list")
    ]
]
```

`deleteAlbum` is to be placed in the `View` module. It requires new markup functions:

```fsharp
let strong s = tag "strong" [] (text s)

let form x = tag "form" ["method", "POST"] (flatten x)
let submitInput value = inputAttr ["type", "submit"; "value", value]
```

- `strong` is just an emphasis
- `form` is HTML element for a form with it's "method" attribute set to "POST"
- `submitInput` is button to submit a form

A couple of snippets to handle `deleteAlbum` are still needed, starting with `Db`:

```fsharp
let getAlbum id (ctx : DbContext) : Album option = 
    query { 
        for album in ctx.``[dbo].[Albums]`` do
            where (album.AlbumId = id)
            select album
    } |> firstOrNone
```

for getting `Album option` (not `AlbumDetails`). 
New route in `Path`:

```fsharp
module Admin =
    let manage = "/admin/manage"
    let deleteAlbum : IntPath = "/admin/delete/%d"
```

Finally we can put following in the `App` module:

```fsharp
let deleteAlbum id =
    match Db.getAlbum id (Db.getContext()) with
    | Some album ->
        html (View.deleteAlbum album.Title)
    | None ->
        never
```

```fsharp
    pathScan Path.Admin.deleteAlbum deleteAlbum
```

Note that the code above allows us to navigate to to "/admin/delete/%d", but we still are not able to actually delete an album.
That's because there's no handler in our app to delete the album from database.
For the moment both GET and POST requests will do the same, which is return HTML page asking whether to delete the album.

In order to implement the deletion, add `deleteAlbum` to `Db` module:

```fsharp
let deleteAlbum (album : Album) (ctx : DbContext) = 
    album.Delete()
    ctx.SubmitUpdates()
```

The snippet takes an `Album` as a parameter - instance of this type comes from database, and we can invoke `Delete()` member on it - SQLProvider keeps track of such changes, and upon `ctx.SubmitUpdates()` executes necessary SQL commands. This is somewhat similar to the "Active Record" concept.

Now, in `App` module we can distinguish between GET and POST requests:

```fsharp
let deleteAlbum id =
    let ctx = Db.getContext()
    match Db.getAlbum id ctx with
    | Some album ->
        choose [ 
            GET >>= warbler (fun _ -> 
                html (View.deleteAlbum album.Title))
            POST >>= warbler (fun _ -> 
                Db.deleteAlbum album ctx; 
                Redirection.FOUND Path.Admin.manage)
        ]
    | None ->
        never
```

- `deleteAlbum` WebPart gets passed the `choose` with two possibilities. 
- `GET` and `POST` are WebParts that succeed (return `Some x`) only if the incoming HTTP request is of GET or POST verb respectively.
- after successful deletion of album, the `POST` case redirects us to the `Path.Admin.manage` page

> Important: We have to wrap both GET and POST handlers with a `warbler` - otherwise they would be evaluated just after `Some album` match, resulting in invoking `Db.deleteAlbum` even if POST does not apply.

The grid can now contain a column with link to delete the album in question:

```fsharp
table [
        yield tr [
            for t in ["Artist";"Title";"Genre";"Price";""] -> th [ text t ]
        ]

        for album in albums -> 
        tr [
            for t in [ truncate 25 album.Artist; truncate 25 album.Title; album.Genre; formatDec album.Price ] ->
                td [ text t ]

            yield td [
                aHref (sprintf Path.Admin.deleteAlbum album.AlbumId) (text "Delete")
            ]
        ]
    ]
```

- there's a new empty column header in the first row
- in the last column of each album row comes a cell with link to delete the album
- note, we had to use the `yield` keyword again

We can delete an album, so why don't we proceed to add album functionality now.
It will require a bit more effort, because we actually need some kind of a form with fields to create a new album.
Fortunately, there's a helper module in Suave library exactly for this purpose.

> Note: `Suave.Form` module at the time of writing is still in `Experimental` package - just as `Suave.Html` which we're already using.

First, let's create a separate module `Form` to keep all of our forms in there (yes there will be more soon).
Add the `Form.fs` file just before `View.fs` - both `View` and `App` module will depend on `Form`.
As with the rest of modules, don't forget to follow our modules naming convention.

Now declare the first `Album` form:

```fsharp
module SuaveMusicStore.Form

open Suave.Form

type Album = {
    ArtistId : decimal
    GenreId : decimal
    Title : string
    Price : decimal
    ArtUrl : string
}

let album : Form<Album> = 
    Form ([ TextProp ((fun f -> <@ f.Title @>), [ maxLength 100 ])
            TextProp ((fun f -> <@ f.ArtUrl @>), [ maxLength 100 ])
            DecimalProp ((fun f -> <@ f.Price @>), [ min 0.01M; max 100.0M; step 0.01M ])
            ],
          [])
```

`Album` type contains all fields needed for the form.
For the moment, `Suave.Form` supports following types of fields:

- decimal
- string
- System.Net.Mail.MailAddress
- Suave.Form.Password

> Note: the int type is not supported yet, however we can easily convert from decimal to int and vice versa

Afterwards comes a declaration of the album form (let album : `Form<Album>` =).
It consists of list of "Props" (Properties), of which we can think as of validations:

- First, we declared that the `Title` must be of max length 100
- Second, we declared the same for `ArtUrl`
- Third, we declared that the `Price` must be between 0.01 and 100.0 with a step of 0.01 (this means that for example 1.001 is invalid)

Those properties can be now used as both client and server side.
For client side we will the `album` declaration in our `View` module in order to output HTML5 input validation attributes.
For server side we will use an utility WebPart that will parse the form field values from a request.

> Note: the above snippet uses F# Quotations - a feature that you can read more about [here](https://msdn.microsoft.com/en-us/library/dd233212.aspx).
> For the sake of tutorial, you only need to know that they allow Suave to lookup the name of a Field from a property getter.

To see how we can use the form in `View` module, add `open Suave.Form` to the beginning:

```fsharp
module SuaveMusicStore.View

open System

open Suave.Html
open Suave.Form
```

Next, add a couple of helper functions:

```fsharp
let divClass c = divAttr ["class", c]

...

let fieldset x = tag "fieldset" [] (flatten x)
let legend txt = tag "legend" [] (text txt)
```

And finally this block of code:

```fsharp
type Field<'a> = {
    Label : string
    Xml : Form<'a> -> Suave.Html.Xml
}

type Fieldset<'a> = {
    Legend : string
    Fields : Field<'a> list
}

type FormLayout<'a> = {
    Fieldsets : Fieldset<'a> list
    SubmitText : string
    Form : Form<'a>
}

let renderForm (layout : FormLayout<_>) =    
    
    form [
        for set in layout.Fieldsets -> 
            fieldset [
                yield legend set.Legend

                for field in set.Fields do
                    yield divClass "editor-label" [
                        text field.Label
                    ]
                    yield divClass "editor-field" [
                        field.Xml layout.Form
                    ]
            ]

        yield submitInput layout.SubmitText
    ]
```

Above snippet is quite long but, as we'll soon see, we'll be able to reuse it a few times.
The `FormLayout` types defines a layout for a form and consists of:

- `SubmitText` that will be used for the string value of submit button
- `Fieldsets` - a list of `Fieldset` values
- `Form` - instance of the form to render

The `Fieldset` type defines a layout for a fieldset:

- `Legend` is a string value for a set of fields
- `Fields` is a list of `Field` values

The `Field` type has:

- a `Label` string
- `Xml` - function which takes `Form` and returns `Xml` (object model for HTML markup). It might seem cumbersome, but the signature is deliberate in order to make use of partial application

> Note: all of above types are generic, meaning they can accept any type of form, but the form's type must be consistent in the `FormLayout` hierarchy.

`renderForm` is a reusable function that takes an instance of `FormLayout` and returns HTML object model:

- it creates a form element
- the form contains a list of fieldsets, each of which:
    - outputs its legend first
    - iterates over its `Fields` and
        - outputs div element with label element for the field
        - outputs div element with target input element for the field
- the form ends with a submit button

`renderForm` ca be invoked like this:

```fsharp
let createAlbum genres artists = [ 
    h2 "Create"
        
    renderForm
        { Form = Form.album
          Fieldsets = 
              [ { Legend = "Album"
                  Fields = 
                      [ { Label = "Genre"
                          Xml = selectInput (fun f -> <@ f.GenreId @>) genres None }
                        { Label = "Artist"
                          Xml = selectInput (fun f -> <@ f.ArtistId @>) artists None }
                        { Label = "Title"
                          Xml = input (fun f -> <@ f.Title @>) [] }
                        { Label = "Price"
                          Xml = input (fun f -> <@ f.Price @>) [] }
                        { Label = "Album Art Url"
                          Xml = input (fun f -> <@ f.ArtUrl @>) ["value", "/placeholder.gif"] } ] } ]
          SubmitText = "Create" }

    div [
        aHref Path.Admin.manage (text "Back to list")
    ]
]
```

We can see that for the `Xml` values we can invoke `selectInput` or `input` functions.
Both of them take as first argument function which directs to field for which the input should be generated.
`input` takes as second argument a list of optional attributes (of type `string * string` - key and value).
`selectInput` takes as second argument list of options (of type `decimal * string` - value and display name).
As third argument, `selectInput` takes an optional selected value - in case of `None`, the first one will be selected initially.

> Note: We are hardcoding the album's `ArtUrl` property with "/placeholder.gif" - we won't implement uploading images, so we'll have to stick with a placeholder image.

Now that we have the `createAlbum` view, we can write the appropriate WebPart handler.
Start by adding `getArtists` to `Db`:

```fsharp
type Artist = DbContext.``[dbo].[Artists]Entity``

...

let getArtists (ctx : DbContext) : Artist list = 
    ctx.``[dbo].[Artists]`` |> Seq.toList
```

Then proper entry in `Path` module:

```fsharp
    let createAlbum = "/admin/create"
```

and WebPart in `App` module:

```fsharp
let createAlbum =
    let ctx = Db.getContext()
    choose [
        GET >>= warbler (fun _ -> 
            let genres = 
                Db.getGenres ctx 
                |> List.map (fun g -> decimal g.GenreId, g.Name)
            let artists = 
                Db.getArtists ctx
                |> List.map (fun g -> decimal g.ArtistId, g.Name)
            html (View.createAlbum genres artists))
    ]

...

    path Path.Admin.createAlbum >>= createAlbum
```

Once again, `warbler` will prevent from eager evaluation of the WebPart - it's vital here.
To our `View.manage` we can add a link to `createAlbum`:

```fsharp
let manage (albums : Db.AlbumDetails list) = [ 
    h2 "Index"
    p [
        aHref Path.Admin.createAlbum (text "Create New")
    ]
...
```

This allows us to navigate to "/admin/create", however we're still lacking the actual POST handler.

Before we define the handler, let's add another helper function to `App` module:

```fsharp
let bindToForm form handler =
    bindReq (bindForm form) handler BAD_REQUEST
```

It requires a few modules to be open, namely:

- `Suave.Form`
- `Suave.Http.RequestErrors`
- `Suave.Model.Binding`

What `bindToForm` does is:

- it takes as first argument a form of type `Form<'a>`
- it takes as second argument a handler of type `'a -> WebPart`
- if the incoming request contains form fields filled correctly, meaning they can be parsed to corresponding types, and hold all `Prop`s defined in `Form` module, then the `handler` argument is applied with the values of `'a` filled in
- otherwise the 400 HTTP Status Code "Bad Request" is returned with information about what was malformed.

There are just 2 more things before we're good to go with creating album functionality.

We need `createAlbum` for the `Db` module (the created album is piped to `ignore` function, because we don't need it afterwards):

```fsharp
let createAlbum (artistId, genreId, price, title) (ctx : DbContext) =
    ctx.``[dbo].[Albums]``.Create(artistId, genreId, price, title) |> ignore
    ctx.SubmitUpdates()
```

as well as POST handler inside the `createAlbum` WebPart:

```fsharp
choose [
        GET >>= ...

        POST >>= bindToForm Form.album (fun form ->
            Db.createAlbum (int form.ArtistId, int form.GenreId, form.Price, form.Title) ctx
            Redirection.FOUND Path.Admin.manage)
    ]
```

We have delete, we have create, so we're left with the update part only.
This one will be fairly easy, as it's gonna be very similar to create (we can reuse the album form we declared in `Form` module).

`editAlbum` in `View`:

```fsharp
let editAlbum (album : Db.Album) genres artists = [ 
    h2 "Edit"
        
    renderForm
        { Form = Form.album
          Fieldsets = 
              [ { Legend = "Album"
                  Fields = 
                      [ { Label = "Genre"
                          Xml = selectInput (fun f -> <@ f.GenreId @>) genres (Some (decimal album.GenreId)) }
                        { Label = "Artist"
                          Xml = selectInput (fun f -> <@ f.ArtistId @>) artists (Some (decimal album.ArtistId))}
                        { Label = "Title"
                          Xml = input (fun f -> <@ f.Title @>) ["value", album.Title] }
                        { Label = "Price"
                          Xml = input (fun f -> <@ f.Price @>) ["value", formatDec album.Price] }
                        { Label = "Album Art Url"
                          Xml = input (fun f -> <@ f.ArtUrl @>) ["value", "/placeholder.gif"] } ] } ]
          SubmitText = "Save Changes" }

    div [
        aHref Path.Admin.manage (text "Back to list")
    ]
]
```

Path:

```fsharp
    let editAlbum : IntPath = "/admin/edit/%d"    
```

Link in `manage` in `View`:

```fsharp
for album in albums -> 
        tr [
            ...

            yield td [
                aHref (sprintf Path.Admin.editAlbum album.AlbumId) (text "Edit")
                text " | "
                aHref (sprintf Path.Admin.deleteAlbum album.AlbumId) (text "Delete")
            ]
        ]
```

`updateAlbum` in `Db` module:

```fsharp
let updateAlbum (album : Album) (artistId, genreId, price, title) (ctx : DbContext) =
    album.ArtistId <- artistId
    album.GenreId <- genreId
    album.Price <- price
    album.Title <- title
    ctx.SubmitUpdates()
```

`editAlbum` WebPart in `App` module:

```fsharp
let editAlbum id =
    let ctx = Db.getContext()
    match Db.getAlbum id ctx with
    | Some album ->
        choose [
            GET >>= warbler (fun _ ->
                let genres = 
                    Db.getGenres ctx 
                    |> List.map (fun g -> decimal g.GenreId, g.Name)
                let artists = 
                    Db.getArtists ctx
                    |> List.map (fun g -> decimal g.ArtistId, g.Name)
                html (View.editAlbum album genres artists))
            POST >>= bindToForm Form.album (fun form ->
                Db.updateAlbum album (int form.ArtistId, int form.GenreId, form.Price, form.Title) ctx
                Redirection.FOUND Path.Admin.manage)
        ]
    | None -> 
        never
```

and finally `pathScan` in main `choose` WebPart:

```fsharp
    pathScan Path.Admin.editAlbum editAlbum
```

Comments to above snippets:

- `editAlbum` View looks very much the same as the `createAlbum`. The only significant difference is that it has all the filed values pre-filled. 
- in `Db.updateAlbum` we can see examples of property setters. This is the way SQLProvider mutates our `Album` value, while keeping track on what has changed before `SubmitUpdates()`
- `warbler` is needed in `editAlbum` GET handler to prevent eager evaluation
- but it's not necessary for POST, because POST needs to parse the incoming request, thus the evaluation is postponed upon the successful parsing.
- after the album is updated, a redirection to `manage` is applied

> Note: SQLProvider allows to change `Album` properties after the object has been instantiated - that's generally against the immutability concept that's propagated in the functional programming paradigm. We need to remember however, that F# is not pure functional programming language, but rather "functional first". This means that while it encourages to write in functional style, it still allows to use Object Oriented constructs. This often turns out to be useful, for example when we need to improve performance.

As the icing on the cake, let's also add link to details for each of the albums in `View.manage`:

```fsharp
aHref (sprintf Path.Admin.editAlbum album.AlbumId) (text "Edit")
text " | "
aHref (sprintf Path.Store.details album.AlbumId) (text "Details")
text " | "
aHref (sprintf Path.Admin.deleteAlbum album.AlbumId) (text "Delete")
```

Pheeew, this section was long, but also very productive. Looks like we can already do some serious interaction with the application!
Results can be seen here: [Tag - crud_and_forms](https://github.com/theimowski/SuaveMusicStore/tree/crud_and_forms)