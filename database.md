# Database

In this section we'll see how to add data access to our application.
We'll use SQL Server for the database - you can use the Express version bundled with Visual Studio.
Download the [`create.sql` script](https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/create.sql) to create `SuaveMusicStore` database.

There are many ways to talk with a database from .NET code including ADO.NET, light-weight libraries like Dapper, ORMs like Entity Framework or NHibernate.
To have more fun, we'll do something completely different, namely try an awesome F# feature called Type Providers.
In short, Type Providers allows to automatically generate a set of types based on some type of schema.
To learn more about Type Providers, check out [this resource](https://msdn.microsoft.com/en-us/library/hh156509.aspx).

SQLProvider is example of a Type Provider library, which gives ability to cooperate with a relational database.
We can install SQLProvider from NuGet:
```install-package SQLProvider -includeprerelease```

> Note: SQLProvider is marked on NuGet as a "prerelease". While it could be risky for more sophisticated queries, we are perfectly fine to use it in our case, as it fulfills all of our data access requirements.

If you're using Visual Studio, a dialog window can pop asking to confirm enabling the Type Provider. 
This is just to notify about capability of the Type Provider to execute its custom code in design time.
To be sure the SQLProvider is referenced correctly, select "enable".

Let's also add reference to `System.Data` assembly.

Having installed the SQLProvider, let's add `Db.fs` file to the beginning of our project - before any other `*.fs` file.

In the newly created file, open `FSharp.Data.Sql` module:

```
module SuaveMusicStore.Db

open FSharp.Data.Sql
```

Next, comes the most interesting part:

```
type Sql = 
    SqlDataProvider< 
        "Server=(LocalDb)\\v11.0;Database=SuaveMusicStore;Trusted_Connection=True;MultipleActiveResultSets=true", 
        DatabaseVendor=Common.DatabaseProviderTypes.MSSQLSERVER >
```

You'll need to adjust the above connection string, so that it can access the `SuaveMusicStore` database.
After the SQLProvider can access the database, it will generate a set of types in background - each for single database table, as well as each for single database view.
This might be similar to how Entity Framework generates models for your tables, except there's no explicit code generation involved - all of the types reside under the `Sql` type defined.

The generated types have a bit cumbersome names, but we can define type aliases to keep things simpler:

```
type DbContext = Sql.dataContext
type Album = DbContext.``[dbo].[Albums]Entity``
type Genre = DbContext.``[dbo].[Genres]Entity``
type AlbumDetails = DbContext.``[dbo].[AlbumDetails]Entity``
```

`DbContext` is our data context.
`Album` and `Genre` reflect database tables.
`AlbumDetails` reflects database view - it will prove useful when we'll need to display names for the album's genre and artist.

With the type aliases set up, we can move forward to creating our first queries:

```
let firstOrNone s = s |> Seq.tryFind (fun _ -> true)

let getGenres (ctx : DbContext) : Genre list = 
    ctx.``[dbo].[Genres]`` |> Seq.toList

let getAlbumsForGenre genreName (ctx : DbContext) : Album list = 
    query { 
        for album in ctx.``[dbo].[Albums]`` do
            join genre in ctx.``[dbo].[Genres]`` on (album.GenreId = genre.GenreId)
            where (genre.Name = genreName)
            select album
    }
    |> Seq.toList

let getAlbumDetails id (ctx : DbContext) : AlbumDetails option = 
    query { 
        for album in ctx.``[dbo].[AlbumDetails]`` do
            where (album.AlbumId = id)
            select album
    } |> firstOrNone
```

`getGenres` is a function for finding all genres. 
The function, as well as all functions we'll define in `Db` module, takes the `DbContext` as a parameter.
The `: Genre list` part is a type annotation, which makes sure the function returns a list of `Genre`s.
Implementation is straight forward:  ```ctx.``[dbo].[Genres]`` ``` queries all genres, so we just need to pipe it to the `Seq.toList`.

`getAlbumsForGenre` takes `genreName` as argument (inferred to be of type string) and returns a list of `Album`s.
It makes use of "query expression" (`query { }`) which is very similar to C# Linq query.
Read [here](https://msdn.microsoft.com/en-us/library/hh225374.aspx) for more info about query expressions.
Inside the query expression, we're performing an inner join of `Albums` and `Genres` with the `GenreId` foreign key, and then we apply a predicate on `genre.Name` to match the input `genreName`.
The result of the query is piped to `Seq.toList`.

`getAlbumDetails` takes `id` as argument (inferred to be of type int) and returns `AlbumDetails option` because there might be no Album with the given id.
Here, the result of the query is piped to the `firstOrNone` function, which takes care to transform the result to `option` type.
`firstOrNone` verifies if a query returned any result.
In case of any result, `firstOrNone` will return `Some x`, otherwise `None`.

For more convenient instantiation of `DbContext`, let's introduce a small helper function in `Db` module:

```
let getContext() = Sql.GetDataContext()
```

Now we're ready to finally read real data in the `App` module:

```
let overview =
    Db.getContext() 
    |> Db.getGenres 
    |> List.map (fun g -> g.Name) 
    |> View.store 
    |> html

...

    path Path.Store.overview >>= overview
```

`overview` is a WebPart that... 
Hold on, do I really need to explain it?
The usage of pipe operator here makes the flow rather obvious - each line defines each step.
The return value is passed from one function to another, starting with DbContext and ending with the WebPart.
This is just a single example of how composition in functional programming makes functions look like building blocks "glued" together.

We also need to wrap the `overview` WebPart in a `warbler`:

```
let overview = warbler (fun _ ->
    Db.getContext() 
    |> Db.getGenres 
    |> List.map (fun g -> g.Name) 
    |> View.store 
    |> html)
```

That's because our `overview` WebPart is in some sense static - there is no parameter for it that could influence the outcome.
`warbler` ensures that genres will be fetched from the database whenever a new request comes.
Otherwise, without the `warbler` in place, the genres would be fetched only at the start of the application - resulting in stale genres in case the list changes.
How about the rest of WebParts?

- `browse` is parametrized with the genre name - each request will result in a database query.
- `details` is parametrized with the id - the same as above applies.
- `home` is just fine - for the moment it's completely static and doesn't need to touch the database.

Moving to our next WebPart "browse", let's first adjust it in `View` module:

```
let browse genre (albums : Db.Album list) = [
    h2 (sprintf "Genre: %s" genre)
    ul [
        for a in albums ->
            li (aHref (sprintf Path.Store.details a.AlbumId) (text a.Title))
    ]
]
```

so that it takes two arguments: name of the genre (string) and a list of albums for that genre.
For each album we'll display a list item with a direct link to album's details.

> Note: Here we used the `Path.Store.details` of type `IntPath` in conjunction with `sprintf` function to format the direct link. Again this gives us safety in regards to static typing.

Now, we can modify the `browse` WebPart itself:

```
let browse =
    request (fun r ->
        match r.queryParam Path.Store.browseKey with
        | Choice1Of2 genre -> 
            Db.getContext()
            |> Db.getAlbumsForGenre genre
            |> View.browse genre
            |> html
        | Choice2Of2 msg -> BAD_REQUEST msg)
```

Again, usage of pipe operator makes it clear what happens in case the `genre` is resolved from the query parameter.

> Note: in the example above we adopted "partial application", both for `Db.getAlbumsForGenre` and `View.browse`. 
> This could be achieved because the return type between the pipes is the last argument for these functions.

If you navigate to "/store/browse?genre=Latin", you may notice there are some characters displayed incorrectly.
Let's fix this by setting the "Content-Type" header with correct charset for each HTTP response:

```
let html container =
    OK (View.index container)
    >>= Writers.setMimeType "text/html; charset=utf-8"
```

It's time to read album's details from the database. 
Start by adjusting the `details` in `View` module:

```
let details (album : Db.AlbumDetails) = [
    h2 album.Title
    p [ imgSrc album.AlbumArtUrl ]
    divId "album-details" [
        for (caption,t) in ["Genre:",album.Genre;"Artist:",album.Artist;"Price:",formatDec album.Price] ->
            p [
                em caption
                text t
            ]
    ]
]
```

Above snippet requires defining a few more helper functions in `View`:

```
let imgSrc src = imgAttr [ "src", src ]
let em s = tag "em" [] (text s)

let formatDec (d : Decimal) = d.ToString(Globalization.CultureInfo.InvariantCulture)
```

as well as opening the `System` namespace at the top of the file.

> Note: It's a good habit to open the `System` namespace every single time - in practice it usually turns out to be helpful.

In the `details` function we used list comprehension syntax with an inline list of tuples (`["Genre:",album.Genre;...`).
This is just to save us some time from typing the `p` element three times for all those properties.
You're welcome to change the implementation so that it doesn't use this shortcut if you like.

The `AlbumDetails` database view turns out to be handy now, because we can use all the attributes we need in a single step (no explicit joins required).

To read the album's details in `App` module we can do following:

```
let details id =
    match Db.getAlbumDetails id (Db.getContext()) with
    | Some album ->
        html (View.details album)
    | None ->
        never

...

pathScan Path.Store.details details
```

A few remarks regarding above snippet:

- `details` takes `id` as parameter and returns WebPart
- `Path.Store.details` of type IntPath guarantees type safety
- `Db.getAlbumDetails` can return `None` if no album with given id is found
- If an album is found, html WebPart with the `View.details` container is returned
- If no album is found, `None` WebPart is returned with help of `never`.

No pipe operator was used this time, but as an exercise you can think of how you could apply it to the `details` WebPart.

Before testing the app, add the "placeholder.gif" image asset. 
You can download it from [here](https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/placeholder.gif).
Don't forget to set "Copy To Output Directory", as well as add new file extension to the `pathRegex` in `App` module.

You might have noticed, that when you try to access a missing resource (for example entering album details url with arbitrary album id) then no response is sent.
In order to fix that, let's add a "Page Not Found" handler to our main `choose` WebPart as a last resort:

```
let webPart = 
    choose [
        ...

        html View.notFound
    ]
```

the `View.notFound` can then look like:

```
let notFound = [
    h2 "Page not found"
    p [
        text "Could not find the requested resource"
    ]
    p [
        text "Back to "
        aHref Path.home (text "Home")
    ]
]
```

Results of the section can be seen here: [Tag - Database](https://github.com/theimowski/SuaveMusicStore/tree/database)