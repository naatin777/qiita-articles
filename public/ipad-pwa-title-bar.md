---
title: iPadでPWAするときに左上の信号機を回避する
tags:
  - iPad
  - PWA
private: false
updated_at: '2026-09-16T01:28:10+09:00'
id: 0f83cfa40493b68647a4
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

:::note warn
この記事はiPadOS 26およびiPadOS 27など、左上に信号機が表示されるバージョンを対象にしています。
:::

:::note warn
この記事は2026年9月に書かれた記事になります。
アップデートなどによって挙動が変わることもありますので注意してください。
:::

# やりたいこと

やりたいことは以下のように左上の信号機とステータスバーを回避しつつ、隙間ができないように信号機やステータスバーが重ならない場合は敷き詰めると言った感じです。
これを行わないと信号機とheaderのコンテンツが重なってしまい、ボタンなどを配置したときに押すことができなくなってしまいます。

![やりたいこと](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4148597/b9454174-95b2-40c7-88c8-9faa9e86646a.gif)

動画では、fullscreen以外の場合で左側を5rem空けるようにしています。

# iPadかつstandaloneの判定

まずiPadかつstandaloneで起動しているかを以下のように判定します。
iPadOSのSafariがiPadでも`MacIntel`を返すこともあるため注意です。

```js
// iPadの判定
const ipadLike = navigator.platform === "iPad" || (
 navigator.platform === "MacIntel" &&
 navigator.maxTouchPoints > 1
); 
// standaloneで起動しているかの判定
const standalone = navigator.standalone === true || window.matchMedia("(display-mode: standalone)").matches;
// iPadかつstandaloneか
const ipadStandalone = ipadLike && standalone;
```

# PWAの設定

PWAは以下のように設定しています

```html
<!-- viewport-fit=coverでsafe areaの内側だけでなく、画面全体まで広げる。 -->
<!-- ただし、ステータスバーと重ならないようにenv(safe-area-inset-*)を使って余白を取る必要がある -->
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" /> 
<!-- Webアプリとして起動できるようにする(SafariのUIを表示しない) -->
<meta name="apple-mobile-web-app-capable" content="yes" /> 
<!--　ステータスバーを透過させる -->
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" /> 
<!-- PWAの名前や起動方法などを設定するファイル -->
<link rel="manifest" href="/manifest.webmanifest" />
```

```json:manifest.webmanifest
{ 
  "name": "iPad PWA Demo", 
  "short_name": "PWA Demo", 
  "start_url": "/", 
  "display": "standalone", 
  "background_color": "#ffffff", 
  "theme_color": "#ffffff" 
}
```

# safeAreaTopを取得する

viewport-fit=coverを指定すると、コンテンツを画面端まで広げられます。
そのため、ステータスバーを回避するための余白と、fullscreen判定の補正に使うために`safe-area-inset-top`を取得します。

```js
const probe = document.createElement("div");

probe.style.position = "fixed";
probe.style.inset = "0 auto auto 0";
probe.style.width = "0";
probe.style.height = "0";
// ここで取得している
probe.style.paddingTop = "env(safe-area-inset-top, 0px)";
probe.style.visibility = "hidden";
probe.style.pointerEvents = "none";

document.body.append(probe);

function getSafeAreaTop() {
  return Number.parseFloat(getComputedStyle(probe).paddingTop) || 0;
}
```

# fullscreen判定の罠

この記事でいうfullscreenはFullscreen APIではなく、iPadOSのアプリウインドウをフルスクリーン表示した状態を指します。
画面がfullscreenかどうかを判定する場合に罠があります。

何かしらresizeした後にfullscreenにした場合は

`innerHeight ≈ availHeight`

によってfullscreenかどうかを判定することができます。
`innerHeight`は現在のlayout viewportの高さです。
`screen.availHeight`はWebコンテンツに利用可能な画面領域の高さです。
そのため、両者がほぼ一致していればfullscreenと判定できます。
しかし、起動直後がfullscreenである場合は

`innerHeight + safeAreaTop ≈ availHeight`

によってfullscreenかどうかを判定する必要があります。
そのため、同じfullscreenでも起動直後とresize後の2通りの値を考慮して判定する必要があります。

このようになってしまう理由はわかりませんが、実機で計測したところ以下のような感じです。

```
PWA起動直後の fullscreen

screen / app window
┌─────────────────────┐
│    safe-area-top    │
├─────────────────────┤ ← innerHeight の上端
│                     │
│     innerHeight     │
│                     │
└─────────────────────┘
```
```
resize後の fullscreen

┌─────────────────────┐ ← innerHeight の上端
│    safe-area-top    │
│                     │
│     innerHeight     │
│                     │
└─────────────────────┘
```

実際のfullscreenの判定は以下のように行います。

```js
// 画面サイズには数px程度の誤差が出る可能性があるため、完全一致ではなく2px以内を同一とみなす
const tolerance = 2;
function approximatelyEqual(a, b) {
  return Math.abs(a - b) <= tolerance;
}

function isFullscreen() {
  const { width: availWidth, height: availHeight } =
    getAvailableSize();

  const safeAreaTop = getSafeAreaTop();

  const widthMatches =
    approximatelyEqual(window.innerWidth, availWidth);

  const heightMatches =
    approximatelyEqual(window.innerHeight, availHeight) ||
    approximatelyEqual(window.innerHeight + safeAreaTop, availHeight);

  return widthMatches && heightMatches;
}
```

## 回転への対処

実機で確認したところ、iPadを横向きにしても以下のように縦向きの値のまま返ってきます。
```js
// 計測した値
screen.availWidth // 954
screen.availHeight // 1373
```
このままだとfullscreen判定がうまくいかなくなってしまいます。
そのため、`screen.orientation`から現在の向きを取得し、比較に使う`availWidth`と`availHeight`を縦横に合わせて入れ替えます。
```js
const shortSide = Math.min(
  screen.availWidth,
  screen.availHeight
);

const longSide = Math.max(
  screen.availWidth,
  screen.availHeight
);

function getAvailableSize() {
  const portrait =
    screen.orientation.type.startsWith("portrait");

  return portrait
    ? {
        width: shortSide,
        height: longSide,
      }
    : {
        width: longSide,
        height: shortSide,
      };
}
```

## 実際に回避する部分

fullscreenかどうかが判定できたら以下を呼びます。
```js
function updateFullscreen() {
  document.documentElement.toggleAttribute(
    "data-ipad-fullscreen",
    isFullscreen()
  );
}
```

この関数はiPadかつstandaloneであることを確認した直後と、windowのresizeや画面回転が発生したときに呼び出します。
```js
if (ipadStandalone) {
  document.documentElement.dataset.ipadStandalone = "";

  updateFullscreen();

  window.addEventListener("resize", updateFullscreen);
  screen.orientation.addEventListener("change", updateFullscreen);
}
```

最後にCSS側で回避するためのスペースを確保します

```css
/* ステータスバーを回避 */
html[data-ipad-standalone] .header {
  padding-block-start: env(safe-area-inset-top, 0px);
}

/* 信号機を回避 */
html[data-ipad-standalone]:not([data-ipad-fullscreen]) .header {
  padding-inline-start: max(
    5rem,
    env(safe-area-inset-left, 0px)
  );
}
```

# 全体のソースコード

Coming Soon（GitHub準備中）

# 注意点

`MacIntel && maxTouchPoints > 1`は厳密なiPad判定ではありません。
特にmacOS 27ではSidecarのtouch対応も拡張されているため、今後もこの手法が安全とは限りません。
iPadOS 26では正常ですがiPadOS 27からwindowを上に寄せると上の方にblurがかかって見えづらくなることが確認されています。

# まとめ

今回はiPadで左上の信号機がコンテンツの邪魔をしないように回避する方法についてまとめました。
これをしっかり作り込むとPWAを本物のネイティブアプリっぽく見せることができます！
