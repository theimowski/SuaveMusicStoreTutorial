SuaveからのHello World
======================

Suaveアプリケーションはスタンドアロンのコンソールプロジェクトでホスト出来ます。
まず`SuaveMusicStore`という名前のF# コンソールプログラムを
作成するところから始めましょう。
(すべてのファイルを1つのフォルダに置くようにするために、
ソリューションのディレクトリを作成のチェックは外しておきます。)
次にSuaveのNuGetパッケージを参照に追加します。
パッケージ マネージャー コンソール内で
 `install-package Suave -version 1.0`
と入力します。
あるいはNuGetのGUIを使用してSuaveのパッケージをインストールしても構いません。
ファイル `Program.fs` を `App.fs` に変更して、
用途に適した名前にします。
また、ファイルの内容をすべて書き換えて、以下の通りになるようにします：

````fsharp
open Suave                 // suaveは常にオープンしておきます
open Suave.Http.Successful // OKの結果用
open Suave.Web             // 設定用

startWebServer defaultConfig (OK "Hello world!")
````

ご想像の通り、F5キーを押してプロジェクトを実行すると
アプリケーションが起動します！
標準では `http://localhost:8083` でアクセスできます。
このアドレスをブラウザーで開くと、昔ながらの `Hello World!` が表示されるはずです。
ファイルの先頭にある `open` はC#の `using` ステートメントと同じです。
ただし `App.fs` には `Main` メソッドが定義されていない点に注意してください。
プログラムが実行されると、即座に`startWebServer`関数が呼び出されて、
プロセスが終了するまでリクエストを受け付けられるよう、Suaveがポートを監視し始めます。

[タグ - hello_world](https://github.com/theimowski/SuaveMusicStore/tree/hello_world)

