---
title: "Passcodeview made with SwiftUI"
date: 2021-03-15T00:00:03+09:00
# weight: 1
# aliases: ["/first"]
tags: ["Blog"]
author: "mito"
# author: ["Me", "You"] # multiple authors
showToc: false
TocOpen: false
draft: true
hidemeta: false
comments: false
description: ""
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: false
ShowBreadCrumbs: true
ShowPostNavLinks: false
ShowWordCount: false
ShowRssButtonInSectionTermList: false
UseHugoToc: false
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://marginals.jp"
    Text: "edit" # edit text
    appendFilePath: false # to append file path to Edit link
---

# SwiftUI で Passcode のサンプルを作ってみた

## ソースコード

https://github.com/mitolog/SwiftUI-Passcode

## なぜ作ったか

iOS の touchID や faceID などの生体認証と連携すれば簡単かもしれないけど、今回は独立した機能としてのパスコード画面が必要でした。

また、現在進行中の SwiftUI + Clean Architecture + Combine なプロジェクトだったのですが、SwiftUI 製のいい感じのサンプルやライブラリがなかったので、作成してみました。といっても UI の部分だけがゼロベースであとは既存の組み合わせでうまく機能してくれただけなんですが！

## どんな機能？

- 画面をロックしたり、ホームボタンで閉じて、再度アプリをひらいたときにパスコード画面がでてくる
- 0~9 の数値を任意の桁数で入力してアンロックする
- 間違えた場合は、シェイクアニメーションが走り、入力された数値がリセットされる
- フォントや色などのパラメータを init 時に指定できる

ざっくりいうとこんな感じです。動きは以下の gif を確認ください

![](https://raw.githubusercontent.com/wiki/mitolog/SwiftUI-Passcode/images/SwiftUI-Passcode.gif)

## 仕組み(というかもはや謝辞)

### UI

コアの部分は [Passcode フォルダ配下](https://github.com/mitolog/SwiftUI-Passcode/tree/main/PasscodeSample/Passcode)にある

- PasscodeField.swift
- LegacyTextField.swift
- ShakeAnimation.swift

この 3 つのファイルで構成されています。

PasscodeField が本体で、SwiftUI と Combine で構成されています。細かい UI の説明はソースコードをみた方がはやいと思うので割愛しますが、こだわりポイントとして 2 点だけ。カーソルの点滅具合と、入力された数値が時間差で ● に変わるところはアニメーションの仕組みを理解しながら頑張りました！

LegacyTextField は、画面表示時にキーボードを出すために UIResponder を利用したかったので、[stackoverflow の記事](https://stackoverflow.com/questions/56507839/swiftui-how-to-make-textfield-become-first-responder)と、[hackingwithswift の記事](https://www.hackingwithswift.com/example-code/uikit/how-to-limit-the-number-of-characters-in-a-uitextfield-or-uitextview)をほぼそのまま引っ張ってきています。

※ 今回は iOS14 もサポートしたかったので、そのようにしましたが、iOS15 以降であれば、@FocusState を使うと良さそうです。

ShakeAnimation は、皆さんおなじみ[stack overflow](https://stackoverflow.com/questions/61619013/is-there-a-better-way-to-implement-a-shake-animation-in-swiftui)より。そして、シェイクアニメーションの後、入力したパスコードをリセットしたかったので、[ANTOINE VAN DER LEE さんの記事](https://www.avanderlee.com/swiftui/withanimation-completion-callback/)より抜粋。

### window の取り回し

.fullScreenCover 試してみたり、専用の View(Struct)を作って、GeometryReader 的な感じで必要な View で呼び出してみたりもしたんですが、結局 window を使うと影響範囲が小さくすむし、挙動がスムーズだったので、window を重ねて表示する方法を採用しました。

方法は[@zntfdr](https://twitter.com/zntfdr)さんの、[How to layer multiple windows in SwiftUI](https://www.fivestars.blog/articles/swiftui-windows/)をほぼそのまま採用しました。

window の取り回しが垣間見えるのは、

- SceneDelegate.swift
- PassThroughWindow.swift

です。

考え方としては、PassThroughWindow をメインの Window の上に常に置いておき、Passcode を非表示としたいときは、背景色を Clear とします。それによって、下にあるメインの Window 配下の view が見えるようになっています。

ではどうやってタッチイベントをメインの Window におくっているかといと、PassThroughWindow がオーバーライドしているの hitTest です。

```swift
class PassThroughWindow: UIWindow {
  override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    guard let hitView = super.hitTest(point, with: event) else { return nil }
    return rootViewController?.view == hitView ? nil : hitView
  }
}
```

nil を返すと、PassThroughWindow はこのタッチイベントに関与しないことを宣言し、もう一方のメインの Window に処理が渡るようです。

hitView には、

- パスコード表示時は PasscodeView 配下の何かしらの view
- パスコード非表示時は、PasscodeView の rootView(clear 指定した view)

が入ってくるので、結果として、意図した通りに動作するようになっています。

### ステートの共有

サンプルでは、特にステートを window 間で共有してないですが、SceneDelegate.swift で `let appState = Store<AppState>(AppState())` とすることで single source of truth として扱い、各 view に振り分けられるようにしています。

ステートの共有は、実プロジェクトでは https://github.com/nalexn/clean-architecture-swiftui をベースにしています。今回はあくまでサンプルなので、考え方として必要な部分だけ引っ張ってきました。

## 今後

需要があれば、

- テスト書く
- SwiftPackage に登録する

などしたいです。なので、いい感じに使いたければ[github](https://github.com/mitolog/SwiftUI-Passcode)で star つけてね！
