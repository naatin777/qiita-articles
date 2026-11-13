---
title: AndroidのタイトルバーにChrome風のタブを実装してみた
tags:
  - Android
  - タブ
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

:::note warn
この記事は2026年11月時点の情報です。
Android 16.1のエミュレータで動作確認しています。
:::

# やりたいこと

Androidのデスクトップウィンドウで、Chromeのようにタイトルバーにタブを配置し、タブをウィンドウの外へドラッグすると新しいウィンドウとして切り離せるようにしました。

![dragout_0_10.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/150f1fa5-d639-4870-b054-4bee377ccdf6.gif)

今回はJetpack Composeを使用しています。

# Androidのウィンドウモードについて

Androidには、アプリを自由に移動・リサイズできる**Freeform windowing**と、それを基盤にしたデスクトップ向けの**Desktop windowing**があります。

ただし、Freeform windowingが利用できても、Desktop windowingが有効になっているとは限りません。

今回はDesktop windowingのキャプションバーと、Androidのドラッグ&ドロップ機能を利用します。

詳しくは[AOSPのドキュメント](https://source.android.com/docs/core/display/multi-window)を参照してください。

# タイトルバーにタブを配置する

Desktop windowingでは、ウィンドウ上部に最小化・最大化・閉じるボタンなどが配置される**キャプションバー**があります。

この領域には、アプリ側から独自のUIを描画することができます。

## キャプションバーにComposeを描画する

まず、システムが描画するキャプションバーの背景を透明にします。

```kotlin
window.insetsController?.setSystemBarsAppearance(
    WindowInsetsController.APPEARANCE_TRANSPARENT_CAPTION_BAR_BACKGROUND,
    WindowInsetsController.APPEARANCE_TRANSPARENT_CAPTION_BAR_BACKGROUND,
)
```

これでアプリ側のUIをキャプションバーの背景として表示できるようになります。なお、システムのウィンドウ操作ボタンはそのまま残ります。

次に、Composeの`WindowInsets.captionBar`を使用して、キャプションバーと同じ高さのUIを配置します。

```kotlin
Row(
    modifier = Modifier
        .fillMaxWidth()
        .background(MaterialTheme.colorScheme.surface)
        .then(
            if (WindowInsets.isCaptionBarVisible) {
                // キャプションバーの高さに合わせる
                Modifier.windowInsetsTopHeight(WindowInsets.captionBar)
            } else {
                // キャプションバーがない場合
                Modifier.statusBarsPadding().height(48.dp)
            }
        )
        .absolutePadding(left = leftReserved, right = rightReserved),
) {
    // タブを配置
}
```

ただし、このままではシステムのウィンドウ操作ボタンとタブが重なってしまいます。

そこで、`WindowInsetsCompat.Type.captionBar()`の`getBoundingRects()`を使って、システム側が占有する領域を取得します。

```kotlin
var left = 0
var right = 0

for (rect in insets.getBoundingRects(WindowInsetsCompat.Type.captionBar())) {
    // 中央より左右どちらにあるかを判定
    if (rect.centerX() > windowWidthPx / 2f) {
        right = maxOf(right, windowWidthPx - rect.left)
    } else {
        left = maxOf(left, rect.right)
    }
}
```

取得した矩形から左右の余白をpxで計算し、dpに変換したものを`leftReserved`と`rightReserved`として、先ほどの`Row`に適用しています。

## タブのドラッグ操作を優先する

キャプションバーは通常、ドラッグするとウィンドウ自体が移動します。しかし、今回はキャプションバーに配置したタブをドラッグできるようにしたいです。

そこで、`View.setSystemGestureExclusionRects()`を使用してタブの領域を指定し、システムによるウィンドウ移動よりもアプリ側のドラッグ操作を優先させています。

これにより、タブはドラッグでき、キャプションバーの空いている部分では通常どおりウィンドウを移動できます。

# タブをウィンドウの外へ移動する

次に、タブをウィンドウの外へドラッグした際、新しいウィンドウとして切り離せるようにします。

Androidには、**どのウィンドウもドロップを処理しなかった場合に、指定した`IntentSender`をOS側で起動できる仕組み**が用意されています。

今回はこれを利用します。

## PendingIntentを作成する

通常、Activityを起動するときには`startActivity()`を使用しますが、今回はOS側にActivityを起動してもらいたいです。

そこで使用するのが`PendingIntent`です。

`PendingIntent`は、自分のアプリに代わって、OSなどの別プロセスに後から処理を実行してもらうための仕組みです。

まず、新しいウィンドウで起動するActivityの`Intent`を作成します。

```kotlin
val intent = Intent.makeMainActivity(activity.componentName).apply {
    // 新しいウィンドウでタブを復元するための情報
    putExtra(MainActivity.EXTRA_TAB_ID, tab.id)
    putExtra(MainActivity.EXTRA_TAB_TITLE, tab.title)
    putExtra(MainActivity.EXTRA_TAB_HUE, tab.hue)

    addFlags(
        Intent.FLAG_ACTIVITY_NEW_TASK or
            Intent.FLAG_ACTIVITY_MULTIPLE_TASK or
            Intent.FLAG_ACTIVITY_LAUNCH_ADJACENT
    )
}
```

`FLAG_ACTIVITY_NEW_TASK`と`FLAG_ACTIVITY_MULTIPLE_TASK`を組み合わせて、新しいタスクとして起動するようにしています。

次に、この`Intent`を使って`PendingIntent`を作成します。

```kotlin
// 新しいウィンドウの初期位置とサイズを指定
val options = ActivityOptions.makeBasic().apply {
    setLaunchBounds(newTabWindowBounds(activity.resources.displayMetrics))
}

val pendingIntent = PendingIntent.getActivity(
    activity,
    tab.id.hashCode(),
    intent,
    PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT,
    options.toBundle(),
)
```

`PendingIntent.getActivity()`を使うことで、後からActivityを起動するための処理を用意できます。

## IntentSenderをドラッグデータに含める

続いて、`PendingIntent`から取得した`IntentSender`を`ClipData`に含めます。

```kotlin
val item = ClipData.Item.Builder()
    .setText(tab.id)
    // OS側に実行してもらうIntentSender
    .setIntentSender(pendingIntent.intentSender)
    .build()

val clipData = ClipData(
    "tab:${tab.id}",
    arrayOf(
        ClipDescription.MIMETYPE_TEXT_INTENT,
        ClipDescription.MIMETYPE_TEXT_PLAIN,
    ),
    item,
)
```

ここで作成した`PendingIntent`をアプリが直接実行するわけではありません。

ドロップが処理されなかった場合に、OS側が`IntentSender`を使用して新しいActivityを起動します。

## ドラッグを開始する

今回はComposeの`detectDragGestures`でドラッグ開始を検知し、`View.startDragAndDrop()`でシステムのドラッグ処理を開始しています。

```kotlin
val dragSource = view.rootView

dragSource.startDragAndDrop(
    clipData,
    TabDragShadowBuilder(
        tab, widthPx, heightPx, density, darkTheme, showTabTitles
    ),
    tab, // localState
    View.DRAG_FLAG_GLOBAL_SAME_APPLICATION or
        View.DRAG_FLAG_START_INTENT_SENDER_ON_UNHANDLED_DRAG,
)
```

ここで重要なのが、以下の2つのフラグです。

- `DRAG_FLAG_GLOBAL_SAME_APPLICATION`：同じアプリの別ウィンドウにもドラッグできるようにする
- `DRAG_FLAG_START_INTENT_SENDER_ON_UNHANDLED_DRAG`：どのウィンドウもドロップを処理しなかった場合に、OS側で`IntentSender`を起動できるようにする

これらはAndroid 15（API 35）から利用できます。

Desktop windowingが有効な環境では、ドロップが処理されなかった場合に、OS側が`IntentSender`を使用して新しいウィンドウを起動できます。

そのため、OS標準の仕組みを利用する場合は、アプリ側でウィンドウ外へのドラッグを直接判定する必要がありません。

詳しくは[Android公式のDesktop windowingガイド](https://developer.android.com/develop/adaptive-apps/guides/support-desktop-windowing)を参照してください。

## 元のウィンドウからタブを削除する

ただし、ここまでの実装では、新しいウィンドウを開いても元のウィンドウにタブが残ってしまいます。

そこで今回は、ドラッグごとにUUIDのトークンを生成して新しいActivityに渡し、ドラッグ終了後に`ActivityManager.appTasks`から一致するタスクが起動しているかを確認するようにしました。

新しいタスクが確認できた場合に、元のウィンドウからタブを削除しています。
