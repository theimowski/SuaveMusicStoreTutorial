# Using Postgres

Now that we've moved to using Paket, let's replace the dependency on MS SQL Server with [Postgres](http://www.postgresql.org/).
I'd like here to say big thank you to Chris Holt ([@lefthandedgoat](https://twitter.com/lefthandedgoat)) for proposing the postgres substitute and implementing the Db module for Postgres.

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

Next, install the new dependency with Paket:

```bash
$ mono .paket/paket.exe install --hard
```

The SQL script used for creating the Database Schema in Postgres can be downloaded [from here](https://github.com/theimowski/SuaveMusicStore/blob/postgres/postgres/postgres_create.sql).

Now we can move to actually replacing the whole Db module with a new one designed to work with `Npgsql`.
I won't go into details of implementing the Db module for Postgres, as it should be fairly similar to its MS SQL Server counterpart.
The implementation can be found [here](https://github.com/theimowski/SuaveMusicStore/blob/postgres/postgres/Db.fs).

Last step before we can run the Suave Music Store on a *nix box is to install and configure PostreSQL.
In case you haven't done that before (just as me prior to writing this article), [this](https://help.ubuntu.com/community/PostgreSQL) is a handy guide on how to do that.
When you have PostgreSQL configured, run the `postgres_create.sql` from within the `psql` interactive terminal:

```bash
$ \i .../postgres/postgres_create.sql
```

If you didn't encounter any issues, you should finally be able to compile and run the application.
We can prepare a `build.sh` script to include steps necessary for compiling the app:

```bash
#!/usr/bin/env bash

mono .paket/paket.bootstrapper.exe
mono .paket/paket.exe restore
xbuild
```

Run the `build.sh` script and then fire up the application:

```bash
$ mono bin/Debug/SuaveMusicStore.exe
```

Navigate to 8083 port of your localhost to verify that the app is up and running.