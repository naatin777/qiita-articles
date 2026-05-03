---
title: VSCodeの拡張機能を配布するときはバージョンを見直したほうがいいかもしれない
tags:
  - VSCode-Extension
private: false
updated_at: '2026-05-03T16:01:52+09:00'
id: eb58397d8f7c7913cb00
organization_url_name: null
slide: false
ignorePublish: false
---
:::note warn
この記事は2026年1月時点の情報です。
ツールのアップデートにより状況が変わる可能性がありますのでご了承ください🙇
:::

# 先に結論

`npx yo code`でプロジェクトを作成した後は、`package.json`の以下の2箇所を修正することをおすすめします。

```diff_jsonc:package.json
  "engines": {
-    "vscode": "^1.108.0"
+    "vscode": "^1.104.0" // ターゲットとするエディタのバージョンに合わせて調整
  },
  "devDependencies": {
-   "@types/vscode": "^1.108.0",
+   "@types/vscode": "^1.104.0", // enginesと同じバージョンを指定
    ...
  }
```

WindsurfやCursor、Google Antigravityでは少し古いバージョンのVSCodeをフォークしています。`npx yo code`のデフォルト設定ではその時点の最新のVSCodeを要求するため、互換性エラーになります。

# 発生するエラー

`npx yo code`で生成されたデフォルト設定(最新版)のままpackage化するとVSCodeの場合、以下のようにインストールすることができます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/9701f29d-90c2-46ff-a61e-6a6849a22a48.png)

Google Antigravity(CursorやWindsurfなども同様)の場合、同じようにインストールするとエラーが発生します(エディタのバージョンが1.104.0なので拡張機能が要求するバージョン^1.108.1よりも古いことが原因)。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/c96b04ac-cc95-4bec-8b48-8552714e895c.png)

先ほどのコードのようにバージョンをある程度(ターゲットとなるエディタのバージョンまたはそれ以下)まで下げると無事にインストールすることができました!

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/59238149-9fb0-47dc-997d-deb4329e3d3b.png)

# まとめ

いかがでしたか？
半年ほど前に自作のVSCodeの拡張機能をVisual Studio MarketplaceやOpen VSXにアップしたのですが、Windsurfでインストールしてみたところバージョンの食い違いでエラーが起きたので急いで修正したという経験からこの記事を書きました。
VSCodeで開発していると気づきにくい落とし穴ですが、派生エディタを利用する人が増えているので見直した方がいいかもしれません。
