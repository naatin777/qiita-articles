---
title: SwiftUI×Core Dataの初期プロジェクトを解析してSwiftUIを入門する
tags:
  - Swift
  - SwiftUI
private: false
updated_at: '2026-05-06T22:34:09+09:00'
id: 08650db87c7a0b606244
organization_url_name: null
slide: false
ignorePublish: false
---

:::note info
この記事では、Xcode が生成する SwiftUI + Core Data の初期プロジェクトを題材に、SwiftUI の基本的な仕組みを整理します。
Kotlin や Jetpack Compose などの知識があると、比較しながら読みやすいかもしれません。
:::

# はじめに

Kotlin と Swift は文法が少し違うだけで、基本的には似たようなものに見えます。

しかし、SwiftUI で Core Data を使う初期プロジェクトには、Kotlin や Jetpack Compose の感覚だけでは理解しにくい概念もいくつか登場します。

この記事では、Xcode が生成した SwiftUI + Core Data の初期プロジェクトを題材に、理解しづらいポイントを順番に整理します。

## Xcode でプロジェクトをセットアップするときの設定

- Interface: SwiftUI
- Language: Swift
- Storage: Core Data

## Xcode が生成してくれたサンプルコード（コメントなどは省略）

```swift:ExampleApp.swift
import SwiftUI
import CoreData

@main
struct ExampleApp: App {
    let persistenceController = PersistenceController.shared

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.managedObjectContext, persistenceController.container.viewContext)
        }
    }
}
```

```swift:Persistence.swift
import CoreData

struct PersistenceController {
    static let shared = PersistenceController()

    @MainActor
    static let preview: PersistenceController = {
        let result = PersistenceController(inMemory: true)
        let viewContext = result.container.viewContext
        for _ in 0..<10 {
            let newItem = Item(context: viewContext)
            newItem.timestamp = Date()
        }
        do {
            try viewContext.save()
        } catch {
            let nsError = error as NSError
            fatalError("Unresolved error \(nsError), \(nsError.userInfo)")
        }
        return result
    }()

    let container: NSPersistentContainer

    init(inMemory: Bool = false) {
        container = NSPersistentContainer(name: "swiftuitest")
        if inMemory {
            container.persistentStoreDescriptions.first!.url = URL(fileURLWithPath: "/dev/null")
        }
        container.loadPersistentStores(completionHandler: { (storeDescription, error) in
            if let error = error as NSError? {
                fatalError("Unresolved error \(error), \(error.userInfo)")
            }
        })
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
}
```

```swift:ContentView.swift
import SwiftUI
import CoreData

struct ContentView: View {
    @Environment(\.managedObjectContext) private var viewContext

    @FetchRequest(
        sortDescriptors: [NSSortDescriptor(keyPath: \Item.timestamp, ascending: true)],
        animation: .default)
    private var items: FetchedResults<Item>

    var body: some View {
        NavigationView {
            List {
                ForEach(items) { item in
                    NavigationLink {
                        Text("Item at \(item.timestamp!, formatter: itemFormatter)")
                    } label: {
                        Text(item.timestamp!, formatter: itemFormatter)
                    }
                }
                .onDelete(perform: deleteItems)
            }
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    EditButton()
                }
                ToolbarItem {
                    Button(action: addItem) {
                        Label("Add Item", systemImage: "plus")
                    }
                }
            }
            Text("Select an item")
        }
    }

    private func addItem() {
        withAnimation {
            let newItem = Item(context: viewContext)
            newItem.timestamp = Date()

            do {
                try viewContext.save()
            } catch {
                let nsError = error as NSError
                fatalError("Unresolved error \(nsError), \(nsError.userInfo)")
            }
        }
    }

    private func deleteItems(offsets: IndexSet) {
        withAnimation {
            offsets.map { items[$0] }.forEach(viewContext.delete)

            do {
                try viewContext.save()
            } catch {
                let nsError = error as NSError
                fatalError("Unresolved error \(nsError), \(nsError.userInfo)")
            }
        }
    }
}

private let itemFormatter: DateFormatter = {
    let formatter = DateFormatter()
    formatter.dateStyle = .short
    formatter.timeStyle = .medium
    return formatter
}()

#Preview {
    ContentView().environment(\.managedObjectContext, PersistenceController.preview.container.viewContext)
}
```

# `ExampleApp.swift`

ぱっと見で理解できたことは以下の通りです。

- `@main` は、その型をアプリケーションのエントリーポイントとして扱うための属性
- `struct` は値型、`class` は参照型
- `let persistenceController = PersistenceController.shared` は、Core Data へアクセスするためのシングルトン的なインスタンスを取得している部分

ただ、その先の `var body: some Scene { ... }` には、Swift 特有の概念がいくつか登場します。

## 壁1: `{}` の意味

ここは計算プロパティの省略形として読むと理解しやすくなります。

例えば、次のようなコードがあるとします。

```swift
// 元の形
var a = 1
var b: Int {
    get {
        return a * 2
    }
}

// 省略形
var a = 1
var b: Int {
    a * 2
}
```

Kotlin で書くと、だいたい次のようなイメージです。

```kotlin
var a = 1
val b: Int
    get() = a * 2
```

Kotlin の `get() =` に慣れている場合、Swift の計算プロパティの書き方は少し読みにくく感じるかもしれません。

## 壁2: `some` の意味

`some` は SwiftUI を理解するうえで重要な概念です。

Swift の `some` は opaque type、つまり「具体的な型を外からは隠すが、実装側では具体型が1つに決まっている型」として扱われます。

例えば、次のように書くと、呼び出し側からは `Shape` として見えますが、実際に返している型は `Circle` に固定されます。

```swift
func makeShape() -> some Shape {
    Circle()
}
```

ここでは、次の3パターンを比較します。

### 型推論の場合

```swift
var x = Circle()
```

この場合、`x` は `Circle` 型になります。

そのため、`Circle` 以外の `Shape`、例えば `Rectangle` を再代入することはできません。

### `any` の場合

```swift
var x: any Shape = Circle()
x = Rectangle()
```

この場合、`x` には `Circle` 以外の `Shape` も代入できます。

`any Shape` は存在型なので、`Shape` に準拠した値を抽象的に扱えます。一方で、具体的な型をコンパイル時に固定する `some` とは性質が異なります。

### `some` の場合

```swift
var x: some Shape = Circle()
```

この場合、`x` は `Shape` として隠蔽されつつ、実際の型は `Circle` に固定されます。

そのため、次のように別の型を入れようとするとコンパイルエラーになります。

```swift
var x: some Shape = Circle()
x = Rectangle() // エラー
```

また、関数の戻り値で `some` を使う場合も、実装側で返す具体型は1つに決まっている必要があります。

```swift
func makeShape(_ flag: Bool) -> some Shape {
    if flag {
        Circle()
    } else {
        Rectangle() // エラー
    }
}
```

型推論やジェネリクスだけで十分なのではないか、と考えるかもしれません。

Kotlin では必要な箇所だけを抽象化することが多く、Flutter でも View を Widget として自然に扱えます。そのため、SwiftUI で `some View` が必要になる理由は、直感的にはわかりにくい部分です。

### `some View` が必要な理由

SwiftUI の `body` では、具体的な View の型が非常に複雑になります。

SwiftUI では、`@ViewBuilder` や modifier によって View の型が合成されていきます。例えば、実際の型は次のように深くネストしたものになる可能性があります。

```swift
VStack<TupleView<(Text, Button<Text>)>>
```

実際には modifier なども加わるため、さらに複雑になります。

その具体型を毎回書くのは現実的ではありません。また、具体型を API として公開してしまうと、内部実装を少し変えただけで外側に見える型も変わってしまいます。

そこで `var body: some View` と書くことで、具体型を外からは隠しつつ、コンパイラ内部では型情報を保てるようにしています。

### 「リバースジェネリクス」と呼ばれる理由

普通のジェネリクスは、関数の引数側を抽象化するイメージです。

```swift
func execute<T: Shape>(arg: T) {
    // 何らかの処理
}
```

一方で、`some` は関数の戻り値側を抽象化するように見えます。

```swift
func execute() -> some Shape {
    Circle()
}
```

呼び出し側が型を選ぶのが通常のジェネリクスだとすると、`some` は実装側が具体型を選び、呼び出し側には抽象化して見せる仕組みだと考えると理解しやすかったです。

### `protocol` はジェネリクスではなく `associatedtype` を使うらしい

`View` の構造は、かなり単純化すると次のようなイメージだと考えています。

```swift
protocol View {
    associatedtype Body: View
    var body: Body { get }
}
```

`View` は `associatedtype Body` を持つプロトコルです。

`body` の具体的な戻り値の型は Swift が推論してくれますが、プロパティの型注釈としては `var body: some View` のように書く必要があります。

```swift
var body: some View {
    Text("hello")
}
```

このあたりは、Kotlin の interface や generics の感覚だけで読むと少し混乱しやすい部分です。

### その他

Kotlin にも用途は異なりますが、`inline reified` と呼ばれる仕組みがあります。

## 壁3: `View { ... }` の中身はどうなってる

`WindowGroup` は計算プロパティの `get` ではなく、Jetpack Compose と同じくクロージャで子 View を受け取るコンテナです。

計算プロパティの `{}` とクロージャの `{}` は役割が異なるため、文脈から読み分けます。

### クロージャなのに、どうしてそんなに複雑な型になるのか

SwiftUI では、`@ViewBuilder` によってクロージャ内の複数の View が1つの View として合成されます。

例えば、次のようなことが起きます。

- `VStack` などで複数の View を並べると、タプルのような形で合成される
- `if` などで条件分岐すると、条件分岐用の型として合成される
- リスト表示では、複数要素を扱うために `ForEach` などを使う

このように、SwiftUI の View は見た目よりも複雑な型として組み立てられています。

### `environment` の役割

`environment` は、Jetpack Compose の `CompositionLocal` や DI に近い役割を持ちます。

```swift
.environment(\.managedObjectContext, persistenceController.container.viewContext)
```

このコードでは、`\.managedObjectContext` が environment のキーで、第2引数に実際に注入する値を渡します。

つまり、ここでは Core Data の `viewContext` を SwiftUI の環境に注入しています。

# `Persistence.swift`

`static` を使っているため、シングルトン的な構成になっています。

`shared` は実機や通常実行用、`preview` は Xcode Preview 用の初期データ作成という役割で使い分けられています。

また、次のような `{ ... }()` は、その場で定義したクロージャをすぐに実行している形です。

```swift
static let preview: PersistenceController = {
    let result = PersistenceController(inMemory: true)
    // 初期データ作成
    return result
}()
```

## `@MainActor` って何？

iOS でも Android と同じく、UI 更新はメインスレッドで扱うのが基本です。

`@MainActor` は Kotlin の `withContext(Dispatchers.Main)` と比較すると理解しやすいですが、厳密には単にその場でメインスレッドに切り替えるものではありません。「この処理やプロパティは MainActor 上で扱う」という隔離指定に近いものです。

今回のコードでは、`preview` の中で `viewContext` を扱っています。`viewContext` はメインスレッド側で使う context なので、`@MainActor` が付いているのだと考えました。

## `init` の中身

`init` の中身は、だいたい次の理解です。

- `NSPersistentContainer(name:)` で Core Data のコンテナを作成する
- `inMemory` のときは `/dev/null` に向けて、永続化されないストアとして扱う
- `loadPersistentStores` で永続ストアを読み込む
- `automaticallyMergesChangesFromParent` で、別の context などで保存された変更を `viewContext` に自動で反映する

```swift
container.viewContext.automaticallyMergesChangesFromParent = true
```

この `automaticallyMergesChangesFromParent` が少しわかりづらかったです。

Core Data の場合は、すでに存在している managed object に対して変更がマージされる、というイメージの方が近いです。

TypeScript で雑に表すと、次のようなイメージです。

```typescript:Core Dataの場合
let value = [new Data(1), new Data(2), new Data(3)]

// DB が更新されると、既存の context 側に変更がマージされる
value.push(new Data(4))
```

一方で、Room + Flow のようなリアクティブな仕組みでは、新しいリストが再度流れてくるイメージがあります。

```typescript:Room + Flowの場合
let value = [new Data(1), new Data(2), new Data(3)]

// DB が更新されると、新しいインスタンスとして再取得されるイメージ
value = [new Data(1), new Data(2), new Data(3), new Data(4)]
```

実際の挙動はより複雑ですが、Core Data の「マージ」は、既存の object/context に対して変更を反映するものと考えると理解しやすくなります。

# `ContentView.swift`

`@Environment(\.managedObjectContext)` は、先ほど注入した environment から値を取り出す場所です。

```swift
@Environment(\.managedObjectContext) private var viewContext
```

`@FetchRequest` は、Core Data 側のデータを取得し、その変更に応じて UI を更新する仕組みです。

```swift
@FetchRequest(
    sortDescriptors: [NSSortDescriptor(keyPath: \Item.timestamp, ascending: true)],
    animation: .default)
private var items: FetchedResults<Item>
```

ソート条件やアニメーションもここで指定できます。

Core Data の変更に応じて UI が更新されるという点では、React Query や Room + Flow に少し近い印象を受けました。

`body` の中身は、Jetpack Compose と共通する部分が多くあります。

## 複数のクロージャを渡す書き方

Swift では、引数にクロージャを複数渡すとき、次のような書き方ができます。

```swift
NavigationLink {
    Text("Item at \(item.timestamp!, formatter: itemFormatter)")
} label: {
    Text(item.timestamp!, formatter: itemFormatter)
}
```

これは、次のように書いているのと近い意味です。

```swift
NavigationLink(
    destination: {
        Text("Item at \(item.timestamp!, formatter: itemFormatter)")
    },
    label: {
        Text(item.timestamp!, formatter: itemFormatter)
    }
)
```

SwiftUI ではこのような trailing closure の書き方が多用されます。ナビゲーションも簡潔に記述できます。

## そういえば `Item` はどこから来たのか？

`Item` は、`.xcdatamodeld` で定義した Entity から自動生成されるクラスです。

デフォルトの Core Data テンプレートでは、`timestamp` を持つ `Item` が最初から用意されています。

そのため、コード上で `Item` の定義が直接見えなくても、Core Data のモデル定義をもとに利用できます。

## その他の関数群

`itemFormatter` は、日付表示用の `DateFormatter` です。

```swift
private let itemFormatter: DateFormatter = {
    let formatter = DateFormatter()
    formatter.dateStyle = .short
    formatter.timeStyle = .medium
    return formatter
}()
```

ここでも `{ ... }()` の形が使われています。

クロージャの中で `DateFormatter` を生成し、設定を加えたうえで、その結果を `itemFormatter` に代入している形です。

`withAnimation` は、状態変更の前後の差分を検知して、UI 更新にアニメーションを付ける仕組みです。

```swift
withAnimation {
    let newItem = Item(context: viewContext)
    newItem.timestamp = Date()

    do {
        try viewContext.save()
    } catch {
        let nsError = error as NSError
        fatalError("Unresolved error \(nsError), \(nsError.userInfo)")
    }
}
```

追加や削除の処理を `withAnimation` で囲むことで、リストの変更にアニメーションを付けられます。

# おわりに

SwiftUI は Flutter や Jetpack Compose と似た宣言的な記法を持ちますが、`some View`、`@ViewBuilder`、`associatedtype`、Core Data の context など、SwiftUI と Swift ならではの概念も多く登場します。

Swift の型システムやメモリ管理を理解することは、SwiftUI のコードを読み解くうえでも重要です。

# 参考文献

- https://qiita.com/rizumita/items/913b05d799b3712260f6
- https://qiita.com/nozomi2025/items/de08ef58a48c7d252d87
- https://zenn.dev/snoop/articles/3f67c185993263
- https://qiita.com/kaneko77/items/e51f736447526342bc34
- https://qiita.com/koher/items/b21879a31210f7408502
