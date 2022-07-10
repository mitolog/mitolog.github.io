---
title: "Passcodeview made with SwiftUI"
date: 2021-03-15T00:00:03+09:00
# weight: 1
# aliases: ["/first"]
tags: ["SwiftUI"]
categories: ["Blog", "Tech"]
author: "mito"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
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
    URL: "https://github.com/mitolog/mitolog.github.io/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

## Source Code

https://github.com/mitolog/SwiftUI-Passcode

## Motivation

On one of my client's project, I need to prepare passcode view that works independent of touch ID or face ID.

Moreover, the project is totally craeted by SwiftUI, so I was looking for some libraries or samples that is written in SwiftUI and meets the functionality needed. But I couldn't find appropriate one. So, I needed to implement by myself.

## The Functionality

- When an user returned to the app after locking the window or closing the app with pushing the home button, the app always needs to show the passcode view.
- An user can unlock the passcode view by choosing numbers from 0 to 9 with pre-defined digits.
- When a user fails to unlock, the shake animation is executed and resets the numbers a user has input.
- You can set fonts and colors when init

You can check how it works by the gif below:

![](https://raw.githubusercontent.com/wiki/mitolog/SwiftUI-Passcode/images/SwiftUI-Passcode.gif)

## How it works

---

### The UI aspects

The core UI consists of these files:

- PasscodeField.swift
- LegacyTextField.swift
- ShakeAnimation.swift

located under the [Passcode folder](https://github.com/mitolog/SwiftUI-Passcode/tree/main/PasscodeSample/Passcode).

`PasscodeField` is the main view and its made with SwiftUI and Combine.

The key points that I'd like to mention are:

#### 1. The cusor and hideen letter imitation

Because the passcode view itself is not using TextInputView, I need to imitate the cusor blinking and need to make the letter hidden some milliseconds after the user's input. I ended up learning that where to put the `.animation` is the key not to animate the properties that you don't want to.

#### 2. Toggle the keyboard

LegacyTextField is used behind the SwiftUI because I need to toggle keyboard on show.

I basically referred articles via [stackoverflow](https://stackoverflow.com/questions/56507839/swiftui-how-to-make-textfield-become-first-responder) and [hackingwithswift](https://www.hackingwithswift.com/example-code/uikit/how-to-limit-the-number-of-characters-in-a-uitextfield-or-uitextview).

I think you should better to use `@FocusState` if the target OS is over iOS 15.

#### 3. Shake animation

I have no idea to achieve shake animation by myself, so I reffered [stack overflow](https://stackoverflow.com/questions/61619013/is-there-a-better-way-to-implement-a-shake-animation-in-swiftui).

Then, to achieve reset function after shakeanimation, I utilised `onAnimationCompleted` callback as written on [ANTOINE VAN DER LEE's article](https://www.avanderlee.com/swiftui/withanimation-completion-callback/).

---

### The Window handling

I tried...

- `.fullScreenCover`
- Custom View(Struct) where you can use passcode view like `GeometryReader{ proxy in ... }`

But, I found that controlling `Window` has less affection towards existing code base and works smoother than the owther methods. So, I adopt layered Window methods.

Thanks to the article: [How to layer multiple windows in SwiftUI](https://www.fivestars.blog/articles/swiftui-windows/) by [@zntfdr](https://twitter.com/zntfdr), I could implement layered window. I really appreciated.

Anyways, you can find how window works on these files:

- SceneDelegate.swift
- PassThroughWindow.swift

located here https://github.com/mitolog/SwiftUI-Passcode/tree/main/PasscodeSample .

The key concept is like this:

Basically we put `PassThroughWindow` in front of the main window.
If you want to hide the Passcode, set the background color `clear` so that you can see the view behind it which is the main window's view.

So, how we can pass the touch event to the main window? We can achieve it like this:

```swift
class PassThroughWindow: UIWindow {
  override func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    guard let hitView = super.hitTest(point, with: event) else { return nil }
    return rootViewController?.view == hitView ? nil : hitView
  }
}
```

PassThroughWindow is the window that PasscodeView is on.

If we returns `nil` here, The PassThroughWindow declares this touch event doesn't relevalent to the view underneath.
So, the event will be passed to the next window which is the main window.

To say it more concretely, we can attain hitView as follows...

- When Passcode is shown, hitView would be certain view under the PasscodeView
- When Passcode is hidden, hitView would be a rootView, where you set it as clear background, of the PasscodeView

So, it would work as you expected as a result.

---

### The State handling

Within the sample, I don't sync the state among windows. But, You can sync by using `let appState = Store<AppState>(AppState())` defined on SceneDelegate.swift where you can deal the appState as a single source of truth.

You can refer actual state handling via https://github.com/nalexn/clean-architecture-swiftui that I use it on the project ongoing.

## Next step...

If there are some comments or needs, I'd like to

- write some tests
- register SwiftPackages

If you have any comments or think it useful, please add a star on https://github.com/mitolog/SwiftUI-Passcode !
