ビュー
======

これまではSuaveアプリケーションの基本的なルーティングを説明しました。
この節では、見た目がいい感じのHTMLマークアップをHTTPレスポンスに入れて
返す方法を紹介します。
HTMLビューをテンプレート化する方法はそれだけでも1つのトピックなので、
ここでは詳細には触れません。
様々な方法で実装することが出来、ここで紹介する方法がHTMLビューを描画する
唯一の正しい方法ではないということだけ念頭に置いておいてください。
とはいえ、以下の実装方法が皆さんにとっても
簡潔かつ分かりやすいものであって欲しいと思っています。
今回のアプリケーションでは、Suaveの別パッケージである `Suave.Experimental` を使用して
サーバーサイドHTMLテンプレートを実装します。

> 注釈：本書の執筆時点で `Suave.Experimental` は
> 別パッケージとして公開されていますが、
> 次のリリースでは破壊的な変更が導入される見込みです。
> また、本書では別パッケージ内のモジュールとして使用している機能が
> Suaveのコアパッケージに組み込まれる可能性もあります。

このパッケージを使用するには、以下のNuGetコマンドを実行して
依存パッケージをインストールする必要があります：

````
install-package Suave.Experimental
````

ビューを定義する前に、 `App.fs` の1行目に以下を追加してコードを整頓しましょう：

````fsharp
module SuaveMusicStore.App

open Suave
````

このコードはファイル内に書かれたすべてのコードが `SuaveMusicStore.App` モジュールに
属するということを表します。
F#コードの構造ならびに構成方法については
[こちら][organizingandstructuringfsharpcode]
を参照してください。
では `App.fs` のすぐ上に `View.fs` というファイルを追加して、
以下のモジュール定義をファイルの先頭部分に記述します：

````fsharp
module SuaveMusicStore.View

open Suave.HTML
````

プロジェクトの構成が分かりやすくなるよう、
チュートリアルを通してこのような名前付け規則を採用することにします。

> 注釈：`View.fs` ファイルが `App.fs` ファイルよりも上にあるということが
> 非常に重要です。F#コンパイラは使用する項目がそれよりも前の位置で
> 定義されていることを必須としています。一見すると重大な欠点のように見えますが、
> 慣れてしまえばこのほうが依存関係を制御しやすいということに気がつくでしょう。
> F#プロジェクトにおいて、循環参照がサポートされていないことの利点については
> [こちら][lackofcyclicdependencies] を参照してください。

[organizingandstructuringfsharpcode]: http://fsharpforfunandprofit.com/posts/recipe-part3/
[lackofcyclicdependencies]: http://fsharpforfunandprofit.com/posts/cyclic-dependencies/
