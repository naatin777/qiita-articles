---
title: SwiftUI×Core Dataの初期プロジェクトを解析してSwiftUIを入門する for 私
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
この記事を書いた理由が勉強記録的なものなのでご了承ください。
:::

:::note info
筆者はSwiftを始めて3日分ぐらいの知識しかありません。
また、AIに頼っている部分が多々ありますのでご了承ください。
:::

:::note info
この記事は筆者が理解できなかった部分をまとめているのでこの記事を理解する際はJetpack Composeなどの知識が必要かもしれません。
:::

## はじめに

最初、私は「KotlinとSwiftは、ほんのちょっと方言が違うだけでほぼ同じ文法だから余裕なのでは？」と思っていました。  
しかし、いざSwiftUIでプロジェクトを立ち上げてみると、Kotlinの常識が通用しない見慣れない概念や文法の壁にぶつかりました。  

この記事では、Xcode が生成した初期プロジェクトを題材にして、**私が個人的に詰まったポイント**を順番に整理します。

### Xcodeでプロジェクトをセットアップするときの設定

- Interface: SwiftUI
- Language: Swift
- Storage: Core Data

### Xcodeが生成してくれたサンプルコード（コメントなどは省略）

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

## ExampleApp.swift

まずぱっと見で以下のことは理解できました。

- `@main` は SwiftUI のエントリーポイント
- `struct`は値型（主にスタック領域）、`class`は参照型（ヒープ領域）という使い分け
- `let persistenceController = PersistenceController.shared`はDBにアクセスするシングルトン

ただ、その先の`var body: some Scene { ... }` でかなり詰まりました。

### 壁1: {}ってなんや

調べると計算プロパティの省略形として読むと少し理解しやすかったです。
イメージはこんな感じです。

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

Kotlin で書くとだいたいこんな感じ。

```kotlin
var a = 1
val b: Int
    get() = a * 2
```

わかりづらいからKotlinみたいに`get() =`って書いて欲しいよー！と思いました。

### 壁2: someってなんや

ここが一番詰まりました。  
ちょっと調べてみた感じリバースジェネリクス？？？意味わからん...
そこで `some` って何？を理解するために、次の3パターンを比較しました。

- **型推論の場合**
    ```swift
    var x = Circle()
    ```
    これだと `x` は `Circle` 型になります。  
    そのため `Circle` 以外の `Shape`（例: `Rectangle`）を再代入できません。

- **anyの場合**
    ```swift
    var x: any Shape = Circle()
    // var x: Shape = Circle() と書ける時代もあったっぽいですが最新のSwiftだとanyは書かないとコンパイルエラー
    ```
    この場合、`x` には `Circle` 以外の `Shape` も代入できます。  
    一方で、実行時に実体を解決する都合で、`some` に比べると最適化しづらい（動的ディスパッチ寄り）という理解です。

- **someの場合**
    ```swift
    var x: some Shape = Circle()
    ```
    この場合、`x` は `Shape` として隠蔽されつつ、型は `Circle` に固定されます。  
    そのため `Rectangle` など別型を入れるとコンパイルエラーになります。  
    コンパイル時に型情報を保ちやすいので、`any` より最適化されやすい（静的ディスパッチ寄り）という理解です。

#### ここで疑問

`some` を使わず、型推論&ジェネリクスだけでよくない？と思いました。  
Kotlin の感覚だと、必要な箇所だけ抽象化すれば十分に見えるからです（Flutterでも勝手にWidgetとして扱ってくれるし）。

#### 疑問への整理（ここで少し納得したかな？）

- SwiftUIの `body` の型は `@ViewBuilder` によって合成されていってネストされていくので型が爆発的に膨大になる（おそらくこんな感じ`VStack<TupleView<(Text, Button<Text>)>>`）
- 具体的な型を公開すると実装変更のたびに修正が必要となってしまう
- 型推論に任せるとコンパイル時間が長くなる原因となる

#### このあたりで、「リバースジェネリクス」と呼ばれる理由が少しわかった気がする

- 普通のジェネリクス: 関数の**引数側**を抽象化
    ```swift
    func execute<T: Shape>(arg: T) {
        // なんか処理
    }
    ```

- リバースジェネリクス: 関数の**返り値側**を抽象化
    ```swift
    func execute() -> some Shape {
        return Circle()
    }
    ```

#### protocolはジェネリクスではなくassociatedtypeを使うらしい?

Viewの構造（Appの構造）はこのような感じになっていると考えられる
```swift
protocol View {
    associatedtype Body: View
    var body: Body { get }
}
```
この場合、型推論を用いて省略したりすることはできないっぽいです。
```swift
var body: ??? {  // ???は省略できない
    Text("hello")
}
```

#### その他

Kotlin にも用途は違うようですが `inline reified` と呼ばれる似たようなものがあるのですね...初めて知りました。

### 壁3: View{...}の中身はどうなってる

まず、`WindowGroup` も `get() =` みたいなもの？と思ったのですが、  
これは普通に実体のあるコンテナで、Jetpack Compose と同じくクロージャで子Viewを渡す形でした。
同じ中括弧だから紛らわしい...

#### クロージャなのにどうしてそんなに複雑な型になる

SwiftUI では `@ViewBuilder` によってクロージャー内の複数 View が1つに合成されるらしいです。
例えば、

- `VStack` などで複数のViewを並べるときタプルで合体
- `if` などで条件分岐する場合は `_ConditionalContent` を用いて合体

Listなどを並べる際には `ForEach` が使われるらしいです。

#### environmentってなんや？

調べた感じ、`environment` は Jetpack Composeの CompositionLocal や DI に近い役割をしている感じだと思います。
`\.managedObjectContext` が環境変数のキーで、次の引数に実際に注入する値を入れるらしいです。

## `Persistence.swift`

まず、 `static` を使っているのでシングルトン的な構成です。  
`shared` は実機/通常実行用、`preview` は Xcode Preview 用の初期データ作成という役割分担に見えます。
`{ ... }()` はその場で実行されるクロージャです。

### @MainActorって何？

iOS でも Android と同じく、UI 更新は main スレッドで扱うのが基本らしいです。
`@MainActor` は Kotlin で言う `withContext(Dispatchers.Main)` に近い感覚で捉えると理解しやすかったです。  
~~`preview` 側ではインメモリストアを使っていて軽量なので、MainActor 上で完結させる意図があるのかな、と解釈しました。~~
`viewContext`がメインスレッドに紐づいてるからっぽいです。

`init` の中身はだいたい次の理解です。

- `NSPersistentContainer(name:)` で Core Data を組み立てる
- `inMemory` のときは `/dev/null` に向けて永続化を無効化する
- `automaticallyMergesChangesFromParent` で、バックグラウンドで中身を変更してもメインスレッドのviewContextでその変更を検知するやつ

ここで理解を終えようと思ったのですが、AI がやたら「マージ」と言うので整理してみました。
DBなんてKotlinのやつと一緒やろと思いましたが、Core Data の場合は差分が適用されるらしいです。
つまり、今あるオブジェクトを保ったまま（ポインタが同じ）そのまま最新の状態と合体している。
TypeScriptで表すと下のような感じのイメージです。

```typescript:Core Dataの場合
let value = [new Data(1), new Data(2), new Data(3)]
// DBが更新されると（今あるプロパティを直接更新する）
value.push(new Data(4))
```
```typescript:Room&Flowの場合
let value = [new Data(1), new Data(2), new Data(3)]
// DBが更新されると（新しいインスタンスが生成）
value = [new Data(1), new Data(2), new Data(3), new Data(4)]
```

## `ContentView.swift`

`@Environment(\.managedObjectContext)` は、先ほど注入した environment から値を取り出す場所、だと思われます。
`@FetchRequest` は Core Data 側の変更を監視して UI を更新してくれる仕組みで、ソートやアニメーションもここで指定できます（React Queryにちょっと似てるな...）。
`body` の中身は、雰囲気的に Jetpack Compose と大きくは変わらない印象でした。

最初わかりませんでしたが、引数にクロージャを2つ渡すときは以下のように書けるらしいです。

```swift
NavigationLink {
    Text("Item at \(item.timestamp!, formatter: itemFormatter)")
} label: {
    Text(item.timestamp!, formatter: itemFormatter)
}
```
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

ナビゲーションの書き方が随分あっさりしてるな...この辺りは今後も勉強します。

### そういえば `Item` はどこから来たのか？

これは `.xcdatamodeld` で定義したエンティティから自動生成されるクラスのようです。
デフォルトでは `timestamp` を持つ `Item` が最初から用意されています。

### その他関数群

`itemFormatter` は日付表示用の `DateFormatter`で、デフォルト表示だと見づらいので、クロージャをその場で実行して内容を上書きしてそうです。
`withAnimation` は、状態変更のbefore afterの差分を検知してアニメーションしてくれるらしいです（デフォルトでは `.easeInOut` 系のアニメーション）めっちゃ便利。

## おわりに

パッとみた感じSwiftUIはそこら辺のFlutterとかJetpack Composeとほぼ同じような書き方でいけるものだと思っていて完全に舐めてました。
個人的な感覚としてSwiftは案外かなり低水準寄りなものなのかなと思いました（GCを使ってないと言うのも驚き）。
今後も少しずつSwiftUIに慣れていこうと思います！

## 多分参考文献

- https://qiita.com/rizumita/items/913b05d799b3712260f6
- https://qiita.com/nozomi2025/items/de08ef58a48c7d252d87
- https://zenn.dev/snoop/articles/3f67c185993263
- https://qiita.com/kaneko77/items/e51f736447526342bc34
- https://qiita.com/koher/items/b21879a31210f7408502
