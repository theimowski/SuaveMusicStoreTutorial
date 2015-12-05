データベース
============

この節ではアプリケーションにデータアクセス機能を実装する方法を紹介します。
データベースとしてはSQL Serverを使用します。
エディションとしてはVisual Studio付属のExpressで構いません。
`SuaveMusicStore`データベースを作成するためのスクリプトファイル
[`create.sql`][createsql] をダウンロードしておいてください。

.NETコードからデータベースにアクセスする方法としては、
ADO.NETをはじめ、Dapperのような軽量なライブラリ、
Entity FrameworkやNHibernateのようなORマッパーなど、
様々な方法があります。
しかし今回はこれらとは全く異なる、F#の型プロバイダーというものを使用して
データにアクセスしてみることにします。
簡単に説明すると、型プロバイダーとは特定の形式のスキーマを元にして
一連の型を自動生成することができるという機能です。
型プロバイダを詳しく学びたい方は
[こちら][typeprovider] を参照してください。

SQLProviderはリレーショナルデータベースとやりとりする機能を持った
型プロバイダライブラリの1つです。
SQLProviderは以下のNuGetコマンドでインストールできます：

````
install-package SQLProvider -includeprerelease
````

> 注釈：SQLProviderは「プレリリース」版のNuGetパッケージとして公開されています。
> 複雑なクエリを実行するにはまだリスクが大きいのですが、
> 今回のケースにおいてはデータアクセスに必要な機能がすべてそろっているため、
> プレリリース版でも全く問題ありません。

Visual Studioを使用している場合、型プロバイダーを有効化するかどうかを確認する
ダイアログが表示されます。これはデザイン時に型プロバイダー独自のコードを
実行していいかどうかを確認するためのものです。
SQLProviderが正しく参照されていることが確認出来れば「有効化」を選択します。

また、`System.Data`も参照に追加しておきましょう。

SQLProviderをインストールした後、
プロジェクトの先頭、つまりその他の`*.fs`ファイルよりも
上の位置に`Db.fs`ファイルを追加します。

この新しいファイルの先頭で、まず`FSharp.Data.Sql`モジュールをオープンします：

````fsharp
module SuaveMusicStore.Db

open FSharp.Data.Sql
````

次に最も興味深いコードを追加します：

````fsharp
type Sql =
    SqlDataProvider<
        "Server=(LocalDb)\\v11.0;Database=SuaveMusicStore;Trusted_Connection=True;MultipleActiveResultSets=true",
        DatabaseVendor=Common.DatabaseProviderTypes.MSSQLSERVER >
````

上の接続文字列は環境により適宜修正して、
`SuaveMusicStore`に接続できるようにしてください。
少なくともサーバーを指定する部分が適切かどうかはチェックする必要があります。
設定方法がわからない場合には、
[こちら][sqlserver_connectionstring] にある、SQL Serverの接続文字列を扱う方法を
参照してください。
SQLProviderがデータベースにアクセスできるようになると、
バックグラウンドで一連の型が生成されます。
型はそれぞれ単一のデータベーステーブル、あるいはデータベースビューを表します。
これはEntity Frameworkがテーブル毎にモジュールを生成するという動作と
似ているかもしれませんが、それとは異なり、
あらかじめ何かコマンドを実行するとコードが生成されるというわけではありません。
`Sql`という型を定義するだけで自動的に定義されるのです。

生成された型それぞれには少し扱いづらい名前が付けられていますが、
型のエイリアス(省略型)を定義して扱いやすくしておきます：

````fsharp
type DbContext = Sql.dataContext
type Album = DbContext.``[dbo].[Albums]Entity``
type Genre = DbContext.``[dbo].[Genres]Entity``
type AlbumDetails = DbContext.``[dbo].[AlbumDetails]Entity``
````

`DbContext`はデータコンテキストを表します。
`Album`と`Genre`はデータベースのテーブルを表す型です。
`AlbumDetails`はデータベースビューを表す型です。
この型はアルバムのジャンル名とアーティスト名を表示する際に役立ちます。

[createsql]: https://raw.githubusercontent.com/theimowski/SuaveMusicStore/master/create.sql
[typeprovider]: https://msdn.microsoft.com/en-us/library/hh156509.aspx
[sqlserver_connectionstring]: https://www.connectionstrings.com/sql-server/
