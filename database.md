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

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">open</span> <span onmouseout="hideTip(event, 'Db.fs:1-3_fs1', 1)" onmouseover="showTip(event, 'Db.fs:1-3_fs1', 1)" class="i">FSharp</span><span class="o">.</span><span onmouseout="hideTip(event, 'Db.fs:1-3_fs2', 2)" onmouseover="showTip(event, 'Db.fs:1-3_fs2', 2)" class="i">Data</span><span class="o">.</span><span class="i">Sql</span>&#10;</div></pre>&#10;<div class="tip" id="Db.fs:1-3_fs1">namespace Microsoft.FSharp</div>&#10;<div class="tip" id="Db.fs:1-3_fs2">namespace Microsoft.FSharp.Data</div>&#10;&#10;

Next, comes the most interesting part:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">type</span> <span onmouseout="hideTip(event, 'Db.fs:5-8_fs1', 1)" onmouseover="showTip(event, 'Db.fs:5-8_fs1', 1)" class="t">Sql</span> <span class="o">=</span> &#10;    <span class="i">SqlDataProvider</span><span class="o">&lt;</span> &#10;        <span class="s">&quot;Server=(LocalDb)</span><span class="e">\\</span><span class="s">v11.0;Database=SuaveMusicStore;Trusted_Connection=True;MultipleActiveResultSets=true&quot;</span>, &#10;        <span class="i">DatabaseVendor</span><span class="o">=</span><span class="i">Common</span><span class="o">.</span><span class="i">DatabaseProviderTypes</span><span class="o">.</span><span class="i">MSSQLSERVER</span> <span class="o">&gt;</span>&#10;</div></pre>&#10;<div class="tip" id="Db.fs:5-8_fs1">type Sql = obj<br /><br />Full name: CDocument.Sql</div>&#10;&#10;

You'll need to adjust the above connection string, so that it can access the `SuaveMusicStore` database. At least you need to make sure that the server instance part is correct. If you're not sure how to configure it, [here](https://www.connectionstrings.com/sql-server/) is a great resource on dealing with connection strings in SQL Server.
After the SQLProvider can access the database, it will generate a set of types in background - each for single database table, as well as each for single database view.
This might be similar to how Entity Framework generates models for your tables, except there's no explicit code generation involved - all of the types reside under the `Sql` type defined.

The generated types have a bit cumbersome names, but we can define type aliases to keep things simpler:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">type</span> <span onmouseout="hideTip(event, 'Db.fs:10-13_fs1', 1)" onmouseover="showTip(event, 'Db.fs:10-13_fs1', 1)" class="t">DbContext</span> <span class="o">=</span> <span class="i">Sql</span><span class="o">.</span><span class="i">dataContext</span>&#10;<span class="k">type</span> <span onmouseout="hideTip(event, 'Db.fs:10-13_fs2', 2)" onmouseover="showTip(event, 'Db.fs:10-13_fs2', 2)" class="t">Album</span> <span class="o">=</span> <span onmouseout="hideTip(event, 'Db.fs:10-13_fs1', 3)" onmouseover="showTip(event, 'Db.fs:10-13_fs1', 3)" class="i">DbContext</span><span class="o">.</span><span class="i">``[dbo].[Albums]Entity``</span>&#10;<span class="k">type</span> <span onmouseout="hideTip(event, 'Db.fs:10-13_fs3', 4)" onmouseover="showTip(event, 'Db.fs:10-13_fs3', 4)" class="t">Genre</span> <span class="o">=</span> <span onmouseout="hideTip(event, 'Db.fs:10-13_fs1', 5)" onmouseover="showTip(event, 'Db.fs:10-13_fs1', 5)" class="i">DbContext</span><span class="o">.</span><span class="i">``[dbo].[Genres]Entity``</span>&#10;<span class="k">type</span> <span onmouseout="hideTip(event, 'Db.fs:10-13_fs4', 6)" onmouseover="showTip(event, 'Db.fs:10-13_fs4', 6)" class="t">AlbumDetails</span> <span class="o">=</span> <span onmouseout="hideTip(event, 'Db.fs:10-13_fs1', 7)" onmouseover="showTip(event, 'Db.fs:10-13_fs1', 7)" class="i">DbContext</span><span class="o">.</span><span class="i">``[dbo].[AlbumDetails]Entity``</span>&#10;</div></pre>&#10;<div class="tip" id="Db.fs:10-13_fs1">type DbContext = obj<br /><br />Full name: CDocument.DbContext</div>&#10;<div class="tip" id="Db.fs:10-13_fs2">type Album = obj<br /><br />Full name: CDocument.Album</div>&#10;<div class="tip" id="Db.fs:10-13_fs3">type Genre = obj<br /><br />Full name: CDocument.Genre</div>&#10;<div class="tip" id="Db.fs:10-13_fs4">type AlbumDetails = obj<br /><br />Full name: CDocument.AlbumDetails</div>&#10;&#10;

`DbContext` is our data context.
`Album` and `Genre` reflect database tables.
`AlbumDetails` reflects database view - it will prove useful when we'll need to display names for the album's genre and artist.


---

GitHub commit: [88f4e18a1e4f2a590605ebad3284dc9a102cc83d](https://github.com/theimowski/SuaveMusicStoreTutorial/commit/88f4e18a1e4f2a590605ebad3284dc9a102cc83d)

Files changed:

* Db.fs (added)
* SuaveMusicStore.fsproj (modified)
* create.sql (added)
* packages.config (modified)
