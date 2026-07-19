---
title: "swift-argument-parserでCLIツールを作ってみよう！"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["swift", "cli", "apple"]
published: true
---

## はじめに

みなさんはSwiftでCLIコマンドツールを作りたいと思ったとき、どうしていますか？  

私の場合は、[jakeheis/SwiftCLI](https://github.com/jakeheis/SwiftCLI)を組み込んで[テンプレリポジトリ](https://github.com/mui-z/SwiftCLIToolTemplate)を作成して利用していました。

しかしAppleが公開している[swift-argument-parser](https://github.com/apple/swift-argument-parser)を見つけ、Apple純正のライブラリのみで手軽にCLIツールが作成できることを知ったため、ここで共有しようと思います！

https://github.com/apple/swift-argument-parser

## swift-argument-parserとは

`swift-argument-paser`は前述したようにAppleが開発、公開しているSwiftでCLIツールを作成できるライブラリです。

名前からして引数のパースだけできると思っていたのですが、そういうわけではなくCLIツールを作るための機能が一通り揃っており、これ一つで作成することできます！

非常にシンプルな例ですが引数として渡された文字列を返すだけであれば、以下のコードだけでできてしまいます！ (ヘルプコマンドもバリデーションも自動で実装される...！！)

**ソース**
```swift
import ArgumentParser

@main
struct Echo: ParsableCommand {
    @Argument(help: "Echo text.")
    var text: String

    mutating func run() throws {
        print(text)
    }
}
```

**実行**
```bash
$ swift run echo "Hello, World!"
Hello, World! 
```

**自動生成されるヘルプコマンド**
```bash
$ swift run echo -h

USAGE: echo <text>

ARGUMENTS:
  <text>                  Echo text.

OPTIONS:                  
  -h, --help              Show help information.
```

READMEにあるように、`Swift Package Manager`や`Swift Format`なども`swift-argument-parser`が利用されているようです！


## 準備

まず最初にSwift Packageの作成をします。
`swift-argument-paser`でCLIツールを作成する場合は、以下のコマンドで自動的に`Package.swift`に`swift-arguemnt-parser`が追加された状態で作成することができます。

```bash
swift package init --type tool
```

生成された`Package.swift`
```swift
// swift-tools-version: 6.4
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "example",
    dependencies: [
        .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.2.0"),
    ],
    targets: [
        // Targets are the basic building blocks of a package, defining a module or a test suite.
        // Targets can depend on other targets in this package and products from dependencies.
        .executableTarget(
            name: "example",
            dependencies: [
                .product(name: "ArgumentParser", package: "swift-argument-parser"),
            ],
            swiftSettings: [
                .enableUpcomingFeature("ApproachableConcurrency"),
            ],
        ),
        .testTarget(
            name: "exampleTests",
            dependencies: ["example"],
            swiftSettings: [
                .enableUpcomingFeature("ApproachableConcurrency"),
            ],
        ),
    ]
)
```

## Hello World!
この時点でコマンドが実行できる状態になっているので、以下で実行してみましょう！

```bash
$ swift run example
Hello, world!
```

ここからは、CLIコマンドを作成するためのオプションなどを解説していきます。

## 引数を受け取る

作成するツールにおいて引数を受け取るには、主に三つのproperty wrapperを利用します。

```swift
// ※ apple/swift-argument-parser READMEを引用、加筆
 
// Booleanで引数を受け取る
@Flag(help: "Include a counter with each repetition.")
var includeCounter = false

// 名前付きで引数を受け取る
@Option(name: .shortAndLong, help: "How many times to repeat 'phrase'.")
var count: Int? = nil

// 名前なしで引数を受け取る
@Argument(help: "The phrase to repeat.")
var phrase: String
```

上記の定義より、実際に叩くと以下のようになります
```
$ repeat hello --count 3
```

初期値を渡したりOptionalにしたりすることで、引数の振る舞いを変えることができます。

### 配列で受け取る
`@Argument`, `@Option`においては、配列で引数を受け取ることもできます。
```swift
@Argument()
var foo: [String]

@Option(name: .customLong("char"))
var chars: [Character]
```


### 補完をサポートする
先ほど述べたproperty wrapperには補完をサポートする機能があり、たとえばファイル名の指定やディレクトリ名、特定のオプションを自動で補完できるようになります。


```swift
// 他にも`.custom(), .file(), .list(), .shellCommand()`などが定義されている
// 詳細：https://swiftpackageindex.com/apple/swift-argument-parser/main/documentation/argumentparser/completionkind
@Argument(.directory) 
var target: String
```

実際にこれらを使う場合はコマンドの引数に `--generate-completion-script [zsh|bash|fish]`をつけてスクリプトを登録する必要があります。
zsh, bashにおいては、事前準備等が必要な場合があります。詳しくは以下をご覧ください。

https://apple.github.io/swift-argument-parser/documentation/argumentparser/installingcompletionscripts/


## エラー、終了コードで終了する

`swift-argument-parser`では `ValidationError`という`Error型`の拡張が定義されています。

エラーでコマンドを終了させたい場合に、以下のようにthrowすることで、終了させることができます。

```swift
throw ValidationError("Invalid input.")
```

```bash
Error: Invalid input.
```

### ExitCode
また`ExitCode`に定義された静的プロパティをthrowすることで、その時点でコマンドを終了させることもできます。

中でも`ExitCode.validationFailure`は、プラットフォームごとに適した終了コードを投げるように実装されています。

```swift
throw ExitCode.success

throw ExitCode.failure

throw ExitCode.validationFailure


// 任意のコードで終わらせたいとき
throw ExitCode(5)
```

加えて`CleanExit`というものもあり、エラー扱いではなくかつメッセージを表示して正常終了させたい。という場面で扱えます。
ヘルプを表示して抜けたい、バージョンを表示したいなどの際に活用できます。
```swift
// 任意のParsableCommandのヘルプを表示して正常終了
throw CleanExit.helpRequest(Echo.self)

// 任意の文字列を表示して正常終了
throw CleanExit.message("Everything looks good!")
```


## Asyncを扱う
ファイルの読み書きやAPI通信などを伴う場合は、`async/await`を用いて非同期処理をしたくなることもあると思います。

その場合は`mutating func run() async throws`と定義することで対応できます。

```swift

mutating func run() async throws {
    let data = await hogeAsyncFunc()
    print(data.rawValue)
}
```


## スクリプトライクに書く

`ParsableArguments.parseOrExit()`を使うことで、スクリプトライクに書くこともできます。

ペライチで済むような小さな処理だけさせる場合は、こちらで書いてしまった方が楽そうです。

```swift
// Echo.swift

import ArgumentParser

// @mainを外すのを忘れずに...！
struct Echo: ParsableCommand {
    @Argument(help: "Echo text.")
    var text: String
}

let echo = Echo.parseOrExit()
print(echo.text)
```


## 終わりに
Swiftで手軽にCLIコマンドツールを作れることが伝わりましたでしょうか？

今回の例ではシンプルなものばかりでしたが、ゴリゴリのCLIコマンドを作るためのオプションも備えています...！
このdockerの引数の取り方を真似たいとか、排他的にフラグを処理したいとか、よくある挙動は実装できるようになっているので、[公式Document](https://swiftpackageindex.com/apple/swift-argument-parser/main/documentation/argumentparser)を見てみると良さそうです！

また最後に宣伝ですが、`swift-argument-parser`を利用して、コマンドからQRコードをサクッと生成できるツールを作成してみました！　よければ使ってみてください！

https://github.com/mui-z/rr


## 参考文献
- [swift-argument-parser - github/apple](https://github.com/apple/swift-argument-parser)
- [Swift Argument Parser を使うツールの作成時は `swift package init --type tool` を使う - taji-taji’s blog](https://taji-taji.hatenablog.com/entry/2025/02/02/220837)
