# Hello World from Suave

Suave application can be hosted as a standalone Console Application.
Let's start by creating a Console Application Project named `SuaveMusicStore` (to keep all the files in single folder, uncheck the option to create folder for solution).
Now we can add NuGet reference to Suave. To do that, in Package Manager Console type:
```install-package Suave -version 1.0```.
Alternatively, you can use the NuGet GUI to find and install the Suave package.
Rename the `Program.fs` file to `App.fs` to better reflect the purpose of the file, and replace its contents completely with the following code:

<pre class="fssnip highlighted"><div lang="fsharp"><span class="k">open</span> <span class="i">Suave</span><span class="o">.</span><span class="i">Successful</span>      <span class="c">// for OK-result</span>&#10;&#10;<span class="i">startWebServer</span> <span class="i">defaultConfig</span> (<span class="i">OK</span> <span class="s">&quot;Hello World!&quot;</span>)&#10;</div></pre>&#10;&#10;

Guess what, if you press F5 to run the project, your application is now up and running!
By default it should be available under `http://localhost:8083`.
If you browse that url, you should be greeted with the classic `Hello World!`.
The `open` statements at the top of the file are the same as `using` statements in C#.
Note there is no `Main` method defined in `App.fs` - what happens here is that the `startWebServer` function is invoked immediately after the program is run and Suave starts listening for incoming request till the process is killed.


---

GitHub commit: [c5d42dd6e7a4391f718d752f78567b573b32f281](https://github.com/theimowski/SuaveMusicStoreTutorial/commit/c5d42dd6e7a4391f718d752f78567b573b32f281)

Files changed:

* .gitignore (added)
* App.config (added)
* App.fs (added)
* AssemblyInfo.fs (added)
* SuaveMusicStore.fsproj (added)
* SuaveMusicStore.sln (added)
* packages.config (added)
