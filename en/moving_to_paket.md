# Moving to Paket

The application is in quite a good shape, however the dependency on Microsoft SQL Server makes it run on Windows only (however recently there has appeared an announcement of [SQL Server for Linux](https://blogs.microsoft.com/blog/2016/03/07/announcing-sql-server-on-linux/)).
Because the F# community pushes the language towards **cross-platform** direction, and indeed most of F# OSS projects are built with "X-Plat" in mind, we'll give it a try to port the Suave Music Store to run on [Mono](http://www.mono-project.com/).

In this and next sections I'll be using Ubuntu.

First step will be moving away from NuGet client, which has a [number of issues](https://github.com/NuGet/Home/issues?utf8=%E2%9C%93&q=is%3Aissue+is%3Aopen+mono) running on Mono to a much more cross-platform friendly [Paket](fsprojects.github.io/Paket/).

## Installing Paket

For the sake of this tutorial, we'll follow the [Installation per repository](http://fsprojects.github.io/Paket/installation.html#Installation-per-repository) wizard, but it's also possible to configure Paket on system-wide basis.

After completing steps described in the above wizard, we can use built-in [`convert-from-nuget` command](http://fsprojects.github.io/Paket/paket-convert-from-nuget.html) to convert from NuGet to Paket:

```bash
$ mono .paket/paket.exe convert-from-nuget --no-install --no-auto-restore
```

Next, we can ask Paket to resolve dependencies and [install](http://fsprojects.github.io/Paket/paket-install.html) them into our `SuaveMusicStore.fsproj` project:

```bash
$ mono .paket/paket.exe install --hard --force
```

That's it. If everything went fine, we should be now able to use Paket with the Suave Music Store.
Next step is replacing the database vendor to one that could be run on Unix OS.
