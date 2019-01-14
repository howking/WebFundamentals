project_path: /web/_project.yaml
book_path: /web/updates/_book.yaml
description: コンポジットスレッドによる入力イベント処理

{# wf_published_on: 2018-09-21 #}
{# wf_updated_on: 2019-01-18 #}
{# wf_featured_image: /web/updates/images/inside-browser/cover.png #}
{# wf_featured_snippet: この計4回のブログ連載の最後の部分では、ユーザー入力が発生した時にどのようにコンポジットが円滑なインタラクションを可能にしているか調べます #}
{# wf_blink_components: N/A #}

<style>
  figcaption {
    font-size:0.9em;
  }
</style>

# モダンブラウザの内部を見る(パート4) {: .page-title }

{% include "web/_shared/contributors/kosamari.html" %}

## 入力がコンポジットにくるまで

これはChromeの内部の仕組みを見ていく計4回のブログ連載の最後で、Webサイトを表示するためのコー
ドの処理方法を調査します。前回の記事では、[レンダリング・プロセス](/web/updates/2018/09/inside-browser-part3)
を紹介し、コンポジットについて調べました。この記事では、ユーザー入力が発生したときにコンポ
サイトが円滑なインタラクションを可能にしている様子を説明します。

## ブラウザから見た入力イベント

「入力イベント」といえば、テキストボックスまたはマウスクリックで入力することだと思われるか
もしれませんが、ブラウザからは、入力とはユーザーからのジェスチャーを意味します。マウスホイー
ルのスクロールは入力イベントであり、タッチまたはマウスオーバーも入力イベントです。

画面へのタッチなどのユーザー操作が発生すると、ブラウザプロセスは最初にジェスチャーとして受
け取ります。ただし、ブラウザプロセスは、タブ内のコンテンツがレンダラープロセスによって処理
されているため、そのジェスチャーが発生した場所を認識するだけです。そのため、ブラウザプロセ
スはイベントタイプ(`touchstart`など)とその座標をレンダラープロセスに送ります。レンダラープ
ロセスは、イベントターゲットを検索し、付随しているイベントリスナーを実行することによって、
イベントを適切に処理します。

<figure>
  <img src="/web/updates/images/inside-browser/part4/input.png" alt="input event">
  <figcaption>
    図1: ブラウザプロセスを通じてレンダラープロセスにルーティングされた入力イベント
  </figcaption>
</figure>

## コンポジタが入力イベントを受け取る

<figure class="attempt-right">
  <a href="/web/updates/images/inside-browser/part3/composit.mp4">
    <video src="/web/updates/images/inside-browser/part3/composit.mp4"
           autoplay loop muted playsinline controls alt="コンポジット">
    </video>
  </a>
  <figcaption>
    図2: ページレイヤの上に浮ぶビューポート
  </figcaption>
</figure>

前回の記事では、ラスタライズされたレイヤーを合成することによってコンポジタがスクロールをス
ムーズに処理する方法について説明しました。もし入力イベントリスナーがページにアタッチされて
いなければ、コンポジタスレッドはメインスレッドから完全に独立して新しい複合フレームを作成で
きます。 しかし、一部のイベントリスナーがページに設定されているとどうなるでしょうか？その
イベントは処理する必要があるのかどうか、コンポジタスレッドはどのようにして判断すればよいで
しょうか。

<div class="clearfix"></div>

## 高速スクロールの不可領域について

JavaScriptの実行はメインスレッドの仕事であるため、ページが合成されると、コンポジタスレッド
はイベントハンドラが設定されているページの領域に「高速スクロールの不可領域」としてマークを
付けます。 この情報により、イベントがその領域で発生した場合、コンポジタスレッドは必ず入力
イベントをメインスレッドに送信することができます。 入力イベントがこの領域の外側から来た場
合、コンポジタスレッドはメインスレッドを待たずに新しいフレームの合成を続けます。

<figure>
  <img src="/web/updates/images/inside-browser/part4/nfsr1.png"
       alt="高速スクロールの不可領域の制限">
  <figcaption>
    図3: 高速スクロールの不可領域として記述された入力
  </figcaption>
</figure>

### イベントハンドラを書くときは注意してください

Web開発における一般的なイベント処理パターンはイベントの委任です。イベントが発生すると、一番上の要素に1つのイベントハンドラをアタッチし、イベントターゲットに基づいてタスクを委任することができます。あなたは下記のようなコードを見たり書いたことがあるかもしれません。

```javascript
document.body.addEventListener('touchstart', event => {
    if (event.target === area) {
        event.preventDefault();
    }
});
```

ページのすべての要素に対して1つのイベントハンドラを作成するだけでなので、このイベント委任
パターンは人間工学的には魅力的です。ただし、ブラウザの観点からこのコードを見ると、ページ全
体が高速スクロールの不可領域としてマークされてしまっています。 つまり、アプリケーションが
ページの特定部分からの入力を気にする必要がないのに、コンポジタスレッドはメインスレッドと通
信し、入力イベントが発生するたびにそれを待ってしまいます。よって、コンポジタのスムーズなス
クロール性能がだいなしになっています。

<figure>
  <img src="/web/updates/images/inside-browser/part4/nfsr2.png"
       alt="full page non fast scrollable region">
  <figcaption>
    図4: ページ全体をカバーする高速スクロールの不可領域へ記述された入力
  </figcaption>
</figure>

これを防ぐためには、イベントリスナーに`passive: true`オプションを渡すことができます。これ
はブラウザにメインスレッドのイベントを続けて受けるヒントとなるので、コンポジタは先に進み、
新しいフレームを合成することができます。

```javascript
document.body.addEventListener('touchstart', event => {
    if (event.target === area) {
        event.preventDefault()
    }
 }, {passive: true});
```

## イベントがキャンセル可能かどうかを確認

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part4/scroll.png" alt="ページスクロール">
  <figcaption>
    図5: ページの一部が水平スクロールに固定されているWebページ
  </figcaption>
</figure>

ページ内に、スクロール方向を水平スクロールのみに制限したいボックスがあるとします。

ポインタイベントで `passive: true`オプションを使うことはページスクロールをスムーズにするこ
とができることを意味しますが、スクロール方向を制限するために`preventDefault`したい時にに垂
直スクロールが始まっているかもしれません。これに対して `event.cancelable`メソッドを使って
確認することができます。

<div class="clearfix"></div>

```javascript
document.body.addEventListener('pointermove', event => {
    if (event.cancelable) {
        event.preventDefault(); // block the native scroll
        /*
        *  do what you want the application to do here
        */
    } 
}, {passive: true});
```

あるいは、`touch-action`のようなCSSルールを使ってイベントハンドラを完全に排除することもで
きます。

```css
#area { 
  touch-action: pan-x; 
}
```

## イベントターゲットを見つける

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part4/hittest.png" alt="hit test">
  <figcaption>
    図6: ペイントを見ているメインスレッドがx.yのどこに描画するのか尋ねる
  </figcaption>
</figure>

コンポジタスレッドがメインスレッドに入力イベントを送信するとき、最初に実行するのはイベント
ターゲットを見つけるためのヒットテストです。ヒットテストでは、レンダリングプロセスで生成さ
れたペイントレコードデータを使用して、イベントが発生したポイント座標の下に何があるかを調べ
ます。

<div class="clearfix"></div>

## メインスレッドへのイベント発行の最小化

前回の記事では、私たちの典型的なディスプレイが1秒間に60回画面を更新する方法と、スムーズなアニメーションのためにどのように歩調を合わせる必要があるかについて説明しました。入力の場合、一般的なタッチスクリーンデバイスは1秒間に60〜120回タッチイベントを配信し、一般的なマウスは1秒間に100回イベントを配信します。入力イベントは、画面が更新できるよりも忠実度が高いです。
`touchmove`のような継続的なイベントが1秒間に120回メインスレッドに送信されると、画面のリフレッシュ速度が遅くなるのに比べて、大量のヒットテストやJavaScriptの実行が引き起こされる可能性があります。

<figure>
  <img src="/web/updates/images/inside-browser/part4/rawevents.png" alt="unfiltered events">
  <figcaption>
    図7: フレームタイムラインをあふれさせるイベントによりページが乱れる
  </figcaption>
</figure>
To minimize excessive calls to the main thread, Chrome coalesces continuous events (such as 
`wheel`, `mousewheel`, `mousemove`, `pointermove`,  `touchmove` ) and delays dispatching until 
right before the next `requestAnimationFrame`. 

メインスレッドへの過度の呼び出しを最小限に抑えるために、Chromeは継続的なイベント（`wheel`、
`mousewheel`、`mousemove`、`pointermove`、`touchmove`など）を統合し、次の
`requestAnimationFrame`の直前までディスパッチを遅らせます。

<figure>
  <img src="/web/updates/images/inside-browser/part4/coalescedevents.png" alt="coalesced events">
  <figcaption>
    図8: 以前と同じタイムラインですが、イベントが合体し遅延しています
  </figcaption>
</figure>

`keydown`、`keyup`、`mouseup`、`mousedown`、`touchstart`、および`touchend`のような個別のイ
ベントは直ちに送出されます。

## フレーム内イベントを取得するには `getCoalescedEvents`を使用してください

ほとんどのWebアプリケーションでは、合体したイベントで十分なユーザーエクスペリエンスを提供
できます。しかし、描画アプリケーションや `touchmove`座標に基づいてパスを配置するようなもの
を作成している場合は、中間の座標を失って滑らかな線を描くことができます。その場合、ポインタ
イベントで `getCoalescedEvents`メソッドを使ってそれらの合体したイベントに関する情報を取得
することができます。

<figure>
  <img src="/web/updates/images/inside-browser/part4/getCoalescedEvents.png"
       alt="getCoalescedEvents">
  <figcaption>
    図9: 左側に滑らかなタッチジェスチャーパス、右側に合体した制限パス
  </figcaption>
</figure>

```javascript
window.addEventListener('pointermove', event => {
    const events = event.getCoalescedEvents();
    for (let event of events) {
        const x = event.pageX;
        const y = event.pageY;
        // draw a line using x and y coordinates.
    }
});
```

## 次のステップ

このシリーズでは、Webブラウザの内部動作について説明しました。 DevToolsがイベントハンドラに
`{passive：true}`を追加することを推奨する理由やスクリプトタグに `async`属性を記述する理由
について考えたことがないのであれば、このシリーズでブラウザがそれらの情報を必要とする理由に
ついて説明してください。より速くより滑らかなウェブ体験を提供します。

### Lighthouseを使用する

あなたのコードをブラウザにとって見やすくしたいけれどどこから始めるべきかわからないようにし
たいのなら、 `Lighthouse`はあらゆるウェブサイトの監査を実行してあなたに正しいことと改善が
必要なことを報告するツールです。監査リストを読むことで、ブラウザがどのようなことに関心を持っ
ているのかということもわかります。

### パフォーマンスを測定する方法を学ぶには

掲載結果の調整はサイトによって異なる可能性があるため、サイトの掲載結果を測定し、そのサイトに最適な内容を判断することが重要です。 Chrome DevToolsチームには、[サイトのパフォーマンスを測定する方法](/web/tools/chrome-devtools/speed/get-started)に関するチュートリアルはほとんどありません。

### サイトに機能ポリシーを追加する

特別な一歩を踏み出したいのなら、 [Feature Policy](/web/updates/2018/06/feature-policy)はあなたがあなたのプロジェクトを構築しているときあなたのためのガードレールになることができる新しいウェブプラットフォーム機能です。機能ポリシーを有効にすると、アプリの特定の動作が保証され、間違いを防ぐことができます。たとえば、アプリが解析をブロックしないようにするには、同期スクリプトポリシーでアプリを実行します。 `sync-script:`  `none`が有効になっていると、パーサーブロッキングJavaScriptは実行されません。これにより、コードがパーサーをブロックするのを防ぐことができ、ブラウザーはパーサーを一時停止することを心配する必要がありません。

## 最後に

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part4/thanks.png" alt="ありがとうございました">
</figure>

私がウェブサイトを作り始めたとき、私は自分のコードを書く方法と私がより生産的になるために何が役立つかについてほとんど気にしていました。これらのことは重要ですが、私たちはブラウザが私たちが書いたコードをどのように扱うかについても考えるべきです。最近のブラウザは、より良いWeb体験をユーザーに提供する方法に投資し続けてきました。私たちのコードを整理することでブラウザに親切になれば、今度はユーザーエクスペリエンスが向上します。私はあなたがブラウザに親切になるための探求に参加してくれることを願っています！

<div class="clearfix"></div>

このシリーズの初期のドラフトをレビューしてくれたすべての人に、ありがとうございます。:
[Alex Russell](https://twitter.com/slightlylate), 
[Paul Irish](https://twitter.com/paul_irish), 
[Meggin Kearney](https://twitter.com/MegginKearney), 
[Eric Bidelman](https://twitter.com/ebidel), 
[Mathias Bynens](https://twitter.com/mathias), 
[Addy Osmani](https://twitter.com/addyosmani), 
[Kinuko Yasuda](https://twitter.com/kinu), 
[Nasko Oskov](https://twitter.com/nasko), 
と Charlie Reis.

このブログ連載を楽しんでいましたか？今後の投稿について質問や提案がある場合は、下記のコメン
ト欄またはTwitterの「@ kosamari」でご連絡ください。

## フィードバック {: .hide-from-toc }

{% include "web/_shared/helpful.html" %}

<div class="clearfix"></div>

{% include "web/_shared/rss-widget-updates.html" %}

{% include "comment-widget.html" %}
