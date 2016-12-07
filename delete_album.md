## Delete album

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
            GET >=> warbler (fun _ -> 
                html (View.deleteAlbum album.Title))
            POST >=> warbler (fun _ -> 
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
let manage (albums : Db.AlbumDetails list) = [ 
    h2 "Index"
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
]
```

- there's a new empty column header in the first row
- in the last column of each album row comes a cell with link to delete the album
- note, we had to use the `yield` keyword again

