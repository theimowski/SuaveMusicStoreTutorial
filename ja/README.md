はじめに
========

本書は [F#][fsharp] と [Suave.IO][suaveio] フレームワークを使用して
アプリケーションを作成する方法についてのチュートリアルになっています。
内容としては、ASP.NETチームが作成した [こちら][aspnetmusicstore] の
Music Storeチュートリアルに強く影響を受けています。
具体的にどういったアプリケーションになるのか知りたい方は
リンク先をチェックしてみてください。

本チュートリアルの対象読者はASP.NET MVCに慣れ親しんだC#プログラマーや、
F#の実用的なアプリケーションを開発する方法を知りたい方です。
C#や.NETの経験が無い方でも意義のあるチュートリアルではありますが、
その場合には部分的に分かりづらいところが有るかもしれません。
ASP.NET MVCとC#で同じ機能を実装する場合でも、その書き方は時間とともに変化していきます。
なお、事前にF#を習得しておく必要はありません。
F#言語の基本的な概念についてはチュートリアル内で説明しています。
チュートリアルでは、Scott Wlaschin氏の素晴らしいWebサイト
 [fsharpforfunandprofit.com][fsharpforfunandprofitcom]
をあちこちで参照しています。
このサイトにはF#に関する素晴らしい記事が多数掲載されています。

以降のほとんどの節では、その時点までに実装されたアプリケーションの
ソースコードに対応するコミットへのリンクが直接記載されています。
これにより、アプリケーションの作成を追体験することができ、
曖昧な部分を振り返ることが出来るようになっています。

チュートリアルはVisual Studio 2013を前提としていますが、
もちろんお好みのIDEを使用していただいて構いません。

このチュートリアルに関するIssue(問題点)あるいはPull Request(プルリクエスト)があれば
いつでも歓迎していますので、是非作成してください。
アプリケーションのコードは [こちら][sourceofsuavemusicstore] 、
本書のコンテンツは [こちら][bookofsuavemusicstore] にあります。

[fsharp]: http://fsharp.org/
[suaveio]: http://suave.io/
[aspnetmusicstore]: http://www.asp.net/mvc/overview/older-versions/mvc-music-store/mvc-music-store-part-1
[fsharpforfunandprofitcom]: http://fsharpforfunandprofit.com/
[sourceofsuavemusicstore]: https://github.com/theimowski/SuaveMusicStore/tree/v1.0
[bookofsuavemusicstore]: https://github.com/theimowski/SuaveMusicStoreTutorial/blob/master/ja/SUMMARY.md
