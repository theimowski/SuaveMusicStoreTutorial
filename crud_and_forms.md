# CRUD and Forms

With the database in place, we can now move to implementing a management module.
This will be a simple Create, Update, Delete functionality with a grid to display all albums in the store.






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