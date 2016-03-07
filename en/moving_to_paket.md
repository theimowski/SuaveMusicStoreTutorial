# Moving to Paket

The application is in quite a good shape, however the dependency on Microsoft SQL Server makes it run on Windows only.
Because the F# community pushes the language towards **cross-platform** direction, and indeed most of F# OSS projects are built with "X-Plat" in mind, we'll give it a try to port the Suave Music Store to run on [Mono](http://www.mono-project.com/).

In this and next sections I'll be using Ubuntu 15.10.

First step will be moving away from NuGet client, which has a [number of issues](https://github.com/NuGet/Home/issues?utf8=%E2%9C%93&q=is%3Aissue+is%3Aopen+mono) running on Mono to a much more cross-platform friendly [Paket](fsprojects.github.io/Paket/).

## Installing Paket

For the sake of this tutorial, we'll follow the [Installation per repository](http://fsprojects.github.io/Paket/installation.html#Installation-per-repository) wizard, but it's also possible to configure Paket on system-wide basis.

After completing steps described in the above wizard, we can use built-in [`convert-from-nuget` command](http://fsprojects.github.io/Paket/paket-convert-from-nuget.html) to convert from NuGet to Paket:

```bash
$ mono .paket/paket.exe convert-from-nuget --no-install --no-auto-restore
```
