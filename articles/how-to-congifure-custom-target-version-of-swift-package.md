---
title: "Swift Packageのターゲットバージョンを細かく設定する"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Swift", "iOS"]
published: true
---
iOS向けSwift PackageのPackage.swiftでターゲットバージョンを設定するとき、以下の書き方だとメジャーバージョン単位での設定しかできませんが、

```swift
    platforms: [
        .iOS(.v14)
    ],
```

[文字列でバージョン指定できるイニシャライザ](https://developer.apple.com/documentation/swift_packages/supportedplatform/3112758-ios)を使うと、こんな感じでマイナーバージョン単位やパッチバージョン単位でも設定できるようになります。

```swift
    platforms: [
        .iOS("14.5")
    ],
```
