---
title: "Androidのアイコンを揺らしてベロベロさせる方法\U0001F92A"
tags:
  - Android
  - icon
private: false
updated_at: '2026-10-05T23:43:06+09:00'
id: ddc977400c7082d4fafb
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

:::note warn
この記事は2026年10月時点の情報です。
Adaptive Iconの視覚効果はLauncherによって挙動が異なるため、同じように動かない環境もあります。
:::

# やりたいこと

ホーム画面でアイコンを上下に揺らすとベロを出す、というアイコンを作りました。

![bero.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/3f44d42d-4b06-4fb2-9e7a-4bdb370aec41.gif)

# 発想

Adaptive Iconはforegroundとbackgroundの2枚重ねです。しかもLauncherによっては、アイコンを動かしたときにこの2枚が少しずれます。

ということは、foregroundに穴をあけておけば、ずれた瞬間にbackgroundの「本来見えないはずの部分」が見えるはずです。

- foreground = 顔の皮(目と口のところが透明な穴)
- background = 皮に隠れている目玉とベロ

にしておけば、アイコンが動いた瞬間に皮がずれてベロが出てくる……のでは?と思いました。

# Adaptive Iconについて

Adaptive Iconは、**foreground**と**background**の2枚(各108×108dp)に分かれています。そのためLauncherによっては、この2枚をずらすような視覚効果(parallaxなど)が入ります。

表示するときの形の切り抜き(マスク)はLauncher側の仕事なので、同じアイコンでも端末によって形が違います(円・squircleなど)。

あと、絵を作るときに関係する決まりはこのへんです。

- 中央の**66×66dpはsafe zone**です。端末/OEMごとのマスクで切れて困る絵柄はこの中に収めます
- 外周**18dpずつ**は、マスクやparallax/pulsingなどで使われる余白です
- 効果を出すか、どう出すかもLauncher依存です。アプリ側からは指定できません

詳しくは[公式ドキュメント](https://developer.android.com/develop/ui/views/launch/icon_design_adaptive)参照。

# アイコンの設計

あとは、2枚がずれたときに面白く見えるように絵を作ります。

## 2枚のレイヤー

| foreground(顔の皮)<br>`drawable/ic_bero_foreground` | background(隠すもの)<br>`drawable/ic_bero_background` |
| --- | --- |
| ![foregroundレイヤー(顔の皮。水色=透明)](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/940b8bad-7804-4529-9a67-12fb9fe92fe9.png) | ![backgroundレイヤー(目玉とベロ)](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/aefac974-8fec-4ba8-a75a-2fe3db87163b.png) |

- **foreground**: 顔の皮。目や口など、backgroundを見せたい部分を透明にしておきます(画像の水色のところが全部穴)
- **background**: 白い下地に、普段は隠れる目玉とベロを仕込んでおきます

作り方は何でもよくて、穴さえ開いていればOKです。今回はVectorDrawableで作りました。

## 重ねたときの見え方

| 静止時 | foregroundがずれたとき |
| --- | --- |
| ![重ねた状態(静止時)](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/961f890d-9ef8-4fc2-9faa-b4617d9df153.png) | ![ずれたときの見え方](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/0b03870b-c7d2-460e-823f-41cc9f88562f.png) |

静止時は穴の真下が白いだけなので、輪郭だけの普通の顔です。foregroundがずれると穴が目玉・ベロの場所に来るので、赤いのが出てきます。

目やベロ自体を動かしているわけではなく、皮がずれて隠れていた部分が見えただけ、というのが今回の仕掛けです。

# 実機で試す

2枚のdrawableができたら、`res/mipmap-anydpi-v26/`にadaptive-iconのXMLを置いて、foreground/backgroundにそれぞれ指定します。

```xml:res/mipmap-anydpi-v26/ic_launcher.xml
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_bero_background" />
    <foreground android:drawable="@drawable/ic_bero_foreground" />
</adaptive-icon>
```

あとはインストールして、ホーム画面でアイコンをドラッグしたり動かしたりします。冒頭のGIFがその録画です。

動作確認環境: Pixel 7 / Pixel Launcher / Android 17

注意点として、

- 視覚効果を出さないLauncherもあります。その場合はずっと静止画です
- どの方向にどのくらい動くかはLauncher次第なので、「ここを動かすとこう見える」は完全には制御できません
- Adaptive Icon自体がAPI 26以降の機能なので、それ未満のOSではこの仕掛けは動きません(古いOS向けには別途普通のアイコンが必要です: [公式ドキュメント](https://developer.android.com/studio/write/create-app-icons))
- テーマアイコン表示時は`monochrome`レイヤーを元に描画されるため、foreground/backgroundの相対移動を利用した今回の仕掛けはそのままでは使えません。なおAndroid 16 QPR 2以降では、`monochrome`を用意していないアプリにもテーマアイコンが自動生成される場合があります

# まとめ

- Adaptive Iconのforeground/background分離を、ちょっと変な方向に使いました
- やっていることは単なるレイヤー移動ですが、絵の作り方次第で「表情が変わった」ように見せられます
- 動くかどうかはLauncher次第です。ベロが出るかはLauncherの気分です
