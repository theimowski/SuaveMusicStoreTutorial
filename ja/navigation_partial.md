# ナビゲーション用パーシャルビュー

手始めに、ビューの一番上へナビゲーションメニューを追加しましょう。
`partNav`という名前の関数として実装します：

```fsharp
let partNav = 
    ulAttr ["id", "navlist"] [ 
        li (aHref Path.home (text "ホーム"))
        li (aHref Path.Store.overview (text "ストア"))
        li (aHref Path.Admin.manage (text "管理"))
    ]
```

`partNav`には「ホーム」「ストア」「管理」という3つのメインタブがあります。`ulAttr`は以下の通り定義します：

```fsharp
let ulAttr attr xml = tag "ul" attr (flatten xml)
```

ここではCSSを使用してメニューの見た目をいい感じに出来るよう、`id`属性を指定出来るようにしています。
`partNav`をメインビューの「header」`div`内に追加します：

```fsharp
divId "header" [
    h1 (aHref Path.home (text "F# Suave Music Store"))
    partNav
]
```

これでMisic Storeのメイン機能に移動出来るようになりました。
