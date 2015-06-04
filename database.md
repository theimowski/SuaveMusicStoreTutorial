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

```fsharp
module SuaveMusicStore.Db

open FSharp.Data.Sql
```

Next, comes the most interesting part:

```fsharp
type Sql = 
    SqlDataProvider< 
        "Server=(LocalDb)\\v11.0;Database=SuaveMusicStore;Trusted_Connection=True;MultipleActiveResultSets=true", 
        DatabaseVendor=Common.DatabaseProviderTypes.MSSQLSERVER >
```

You'll need to adjust the above connection string, so that it can access the `SuaveMusicStore` database.
After the SQLProvider can access the database, it will generate a set of types in background - each for single database table, as well as each for single database view.
This might be similar to how Entity Framework generates models for your tables, except there's no explicit code generation involved - all of the types reside under the `Sql` type defined.

The generated types have a bit cumbersome names, but we can define type aliases to keep things simpler:

```fsharp
type DbContext = Sql.dataContext
type Album = DbContext.``[dbo].[Albums]Entity``
type Genre = DbContext.``[dbo].[Genres]Entity``
type AlbumDetails = DbContext.``[dbo].[AlbumDetails]Entity``
```

`DbContext` is our data context.
`Album` and `Genre` reflect database tables.
`AlbumDetails` reflects database view - it will prove useful when we'll need to display names for the album's genre and artist.






It's time to read album's details from the database. 
Start by adjusting the `details` in `View` module:

```fsharp
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

```fsharp
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

```fsharp
let details id =
    match Db.getAlbumDetails id (Db.getContext()) with
    | Some album ->
        html (View.details album)
    | None ->
        never
```

```fsharp
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

```fsharp
let webPart = 
    choose [
        ...

        html View.notFound
    ]
```

the `View.notFound` can then look like:

```fsharp
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