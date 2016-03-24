# Dockerizing on mono

Now that we can finally run the Suave Music Store on mono, it's time to do what all cool kids do - run it on [docker](https://www.docker.com/)!

> If you're not familiar with Docker, and have [pluralsight](https://www.pluralsight.com/) subscription, I highly recommend watching [this](https://www.pluralsight.com/courses/docker-deep-dive) docker deep-dive course by [Nigel Poulton](https://twitter.com/nigelpoulton). This is a really great course and it gives a quick ramp up on docker, just enough to get started. If you don't have access to pluralsight, I'm pretty sure you can still find plenty of sources on learning docker.

Let's just briefly go through the overall architecture of Suave Music Store running on docker:

* We'll use two separate docker images:
  * First image for the database,
  * Second image for the actual F# app,
* The db image will be build on top of the [official postgres image](https://hub.docker.com/_/postgres/),
* The db image, upon build initialization, will create our `suavemusicstore` database from script,
* The app image will extend the [official mono image](https://hub.docker.com/_/mono/),
* The app image will copy **compiled** binaries to the image (more on that in later course),
* The app image will depend on the db container, that's why we'll provide a link between those.

That's the big picture, let's now go straight to essential configuration:

## Database connection string

To run on docker, we'll have to provide a proper connection string.
Up until now, function `getContext` in Db module worked as expected:

```fsharp
let getContext() = Sql.GetDataContext()
``` 

That's because we accessed it via `localhost` (127.0.0.1) - the same host where we compiled our Sql type provider:

```fsharp
type Sql = 
    SqlDataProvider< 
        ConnectionString      = "Server=127.0.0.1;Database=suavemusicstore;User Id=suave;Password=1234;",
        ...
```

But as we'll run the app container in isolation, we have to specify a proper connection string:

```fsharp
let getContext() = Sql.GetDataContext("Server=suavemusicstore_db;Database=suavemusicstore;User Id=suave;Password=1234;")
```

Server `suavemusicstore_db` is the docker link name that we'll apply when firing up containers.
Docker networking infrastructure takes care of matching the link name with destination host, which will be run in a separate container.

**In real world** (yeah, I knew I'd use this phrase one day) we'd probably move the connection strings to some kind of configuration file.

## Server http binding

At the moment last line of App module looks like following:

```fsharp
startWebServer defaultConfig webPart
```

This uses the `defaultConfig`, which in turn uses binding to `127.0.0.1:8083` by default.
From [this](http://stackoverflow.com/a/27818259) Stack Overflow answer we can read that *"binding inside container to localhost usually prevent from accepting connections"*.
Solution here is to accept requests from all IPs instead.
This can be done by providing `0.0.0.0` address:

```fsharp
let cfg =
  { defaultConfig with
      bindings = [ HttpBinding.mk HTTP (System.Net.IPAddress.Parse "0.0.0.0") 8083us  ] }

startWebServer cfg webPart
```

The snippet above copies all fields from the `defaultConfig` and overrides the binding to `0.0.0.0:8083`.

## Database image

As stated above, the db image will be based on the official postgres image.
On top of that, we'll run our postgres_create.sql script.
This can be declared in the lines of following Dockerfile (create `Dockerfile` under `postgres` directory):

```bash
FROM postgres

COPY postgres_create.sql /docker-entrypoint-initdb.d/postgres_create.sql
```

The `COPY` instruction will place the script in a special directory inside the container, from which all scripts are run when the image is being built.

## Server image

```bash
FROM mono:4.2.3.4

COPY ./bin/Debug /app

EXPOSE 8083

WORKDIR /app

CMD ["mono", "SuaveMusicStore.exe"]
```

http://stackoverflow.com/a/25558040

State of the application to be run on docker: [Tag - docker](https://github.com/theimowski/SuaveMusicStore/tree/docker)