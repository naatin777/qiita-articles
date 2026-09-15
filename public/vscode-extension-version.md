---
title: VSCodeの拡張機能を配布するときはバージョンを見直したほうがいいかもしれない
tags:
  - VSCode-Extension
private: false
updated_at: '2026-05-03T18:40:24+09:00'
id: eb58397d8f7c7913cb00
organization_url_name: null
slide: false
ignorePublish: false
---
:::note warn
この記事は2026年6月時点の情報です。
ツールのアップデートにより状況が変わる可能性がありますのでご了承ください🙇
:::

# 何をすれば良いか

`npx yo code`でプロジェクトを作成した後は、`package.json`の以下の2箇所を修正することをおすすめします。

```diff_jsonc:package.json
  "engines": {
-    "vscode": "^1.126.0"
+    "vscode": "^1.105.0"
  },
  "devDependencies": {
-   "@types/vscode": "^1.126.0",
+   "@types/vscode": "^1.105.0", 
    ...
  }
```

DevinやCursor、Antigravityでは少し古いバージョンのVSCodeをフォークしています。
`npx yo code`のデフォルト設定ではその時点の最新のVSCodeを要求するため、互換性エラーになります。

# 発生するエラー

`npx yo code`で生成されたデフォルト設定(最新版)のままパッケージ化するとVSCodeの場合、以下のようにインストールすることができます。
しかし、Cursor(AntigraviityやWindsurfなども同様)の場合、同じようにインストールするとこのようにエラーが発生します。
エディタのバージョンが1.105.1なのでそれよりも新しいエンジンはインストール出来ないと怒られています。

|VS Code|Cursor|
|---|---|
|![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/ae5a4318-9e85-48f9-8460-c339f973ff29.png)|![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/0f444a1c-5959-4889-b2c1-eaa4fcfb7b50.png)|

# 注意点(追記)

ターゲットとするエディタのバージョンを下げすぎると今度はVSCodeの新しいAPIなどが使えなくなってしまうのである程度までに抑えましょう。

# バージョン確認(追記)

Macの場合、メニューバーのXXX→About XXXから以下のように元となっているVSCodeのバージョンを見ることができます。

|VS Code|Cursor|Devin|Antigravity|
|---|---|---|---|
|![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/8a41743c-78c7-459a-9a48-b3b048696b8d.png)|![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/64aaa3b7-2fa6-4e54-b71f-ca35c24f7139.png)|![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/8a0ffe27-4d62-4e01-9884-fa0f9c8948aa.png)|![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/507962dc-e31c-42e2-905f-f29afadf61d6.png)|
|1.260.0|1.105.1|1.110.1|1.107.0|

# まとめ

いかがでしたか？
前に自作のVSCodeの拡張機能をVisual Studio MarketplaceやOpen VSXにアップしたのですが、Devin(旧Windsurf)でインストールしてみたところバージョンの食い違いでエラーが起きたので急いで修正したという経験からこの記事を書きました。
VSCodeで開発していると気づきにくい落とし穴ですが、派生エディタを利用する人が増えているので見直した方がいいかもしれません。
