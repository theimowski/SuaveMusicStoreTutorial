# Using Postgres

Now that we've moved to using Paket, let's replace the dependency on MS SQL Server with [Postgres](http://www.postgresql.org/).
I'd like say thanks here to Chris Holt ([@lefthandedgoat](https://twitter.com/lefthandedgoat)) for proposing the postgres substitute and implementing the Db module for Postgres.

To start with, let's replace the `SQLProvider` dependency with `Npgsql` - .NET provider for PostgreSQL (`paket.dependencies`):

```bash
source https://www.nuget.org/api/v2

nuget Suave 1.0.0
nuget Suave.Experimental 1.0.0
nuget Npgsql 2.2.5
```

and `paket.references`:

```bash
Suave
Suave.Experimental
Npgsql
```

next, install the new dependency with Paket:

```bash
$ mono .paket/paket.exe install --hard
```