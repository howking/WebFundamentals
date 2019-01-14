project_path: /web/_project.yaml
book_path: /web/updates/_book.yaml
description: ブラウザレンダリングエンジンの内部動作

{# wf_published_on: 2018-09-20 #}
{# wf_updated_on: 2019-01-14 #}
{# wf_featured_image: /web/updates/images/inside-browser/cover.png #}
{# wf_featured_snippet: ブラウザがページデータを受信したら、レンダラプロセス内でページを表示するために何が起こりますか？ #}
{# wf_blink_components: N/A #}

<style>
  figcaption {
    font-size:0.9em;
  }
</style>

# モダンブラウザの内部を見る(パート3) {: .page-title }

{% include "web/_shared/contributors/kosamari.html" %}

## レンダラープロセスの内部動作

この記事は、ブラウザがどのように動作しているのかを見ていく計4回のブログ連載の3回
目です。前の記事では、「[マルチプロセスアーキテク
チャ](/web/updates/2018/09/inside-browser-part1)」と「[ナビゲーションフ
ロー](/web/updates/2018/09/inside-browser-part2)」について説明しました。今回は、
レンダラープロセスの内部で何が起こるのかを見ていきます。

レンダラープロセスはWebパフォーマンスのさまざまな側面に影響を与えます。 レンダラー
プロセスの内部では多くのことが行われているため、この投稿は一般的な概要にすぎませ
ん。 さらに詳しく知りたい場合は、[Web Fundamentalsのパフォーマン
ス](/web/fundamentals/performance/why-performance-matters/)にさらに多くのリソー
スがあります。

## レンダラープロセスにおけるWebコンテンツの処理

レンダラープロセスは、タブの内部で発生するすべてのことを担当します。
レンダラープロセスでは、メインスレッドがユーザに送信するコードの大部分を処理します。
Webワーカーまたはサービスワーカーを使用している場合、JavaScriptの一部がワーカースレッド
によって処理されることがあります。コンポジットスレッドとラスタースレッドもレンダラー
プロセス内で実行され、ページを効率的かつ円滑にレンダリングします。

レンダラープロセスの中心的な仕事は、HTML、CSS、およびJavaScriptをユーザーが対話
できるWebページに変えることです。

<figure>
  <img src="/web/updates/images/inside-browser/part3/renderer.png" alt="レンダラープロセス">
  <figcaption>
    図1: メインスレッド、ワーカースレッド、コンポジタースレッド、ラスタースレッドを内部に
        持つレンダラープロセス
  </figcaption>
</figure>

## パース(構文解析)

### DOMの構造

レンダラープロセスがナビゲーションのコミットメッセージを受信して​​HTMLデータを受信
し始めると、メインスレッドはテキスト文字列（HTML）の解析を開始して**D**ocument
**O**bject **M**odel (**DOM**)に変換します。

DOMとは、Web開発者がJavaScriptを介して対話できるデータ構造およびAPIと同様に、ブ
ラウザにおけるページの内部表現です。

HTMLドキュメントをDOMに解析することは、 [HTML標準](https://html.spec.whatwg.org/)に
よって定義されています。ブラウザにどんなHTMLを入力してもエラーが発生しないことに
気付いたかもしれません。例えば、終了タグ</p>がなくても有効なHTMLです。`Hi!
<b>I'm <i>Chrome</b>!</i>`のような誤ったマークアップ(bタグがiタグの間で閉じられ
ています)は、あたかも`Hi! <b>I'm <i>Chrome</i></b><i>!</i>`と書いたように扱われ
ます。
これは、HTML仕様がこれらのエラーを適切に処理するように設計されているためで
す。これらのことがどうやって行われるのか知りたいのであれば、HTML仕様書の「[エラー
処理とパーサーの奇妙なケースの紹介](https://html.spec.whatwg.org/multipage/parsing.html#an-introduction-to-error-handling-and-strange-cases-in-the-parser)
」を読んでください。

### サブリソースのロード

Webサイトは通常、画像、CSS、およびJavaScriptなどの外部リソースを使用します。 こ
れらのファイルはネットワークまたはキャッシュからロードする必要があります。メイン
スレッドはDOMを構築するためにパースしながらそれらを見つけると _一つずつそれらを
要求することもできます_ が、高速化するために "preload scanner"が同時に実行されま
す。
HTML文書に `<img>`や `<link>`のようなものがある場合、スキャナーはHTMLパーサー
によって生成されたトークンをチェックして、ブラウザプロセスのネットワークスレッド
にリクエストを送信します。

<figure>
  <img src="/web/updates/images/inside-browser/part3/dom.png" alt="DOM">
  <figcaption>
    図2: HTMLを解析してDOMツリーを構築するメインスレッド
  </figcaption>
</figure>

### JavaScriptが解析をブロックする可能性がある

HTMLパーサーが `<script>`タグを見つけると、HTMLドキュメントの解析を一時停止し、
JavaScriptコードをロード、解析、および実行する必要があります。なぜなら、
JavaScriptはDOM構造全体を変更する `document.write()`のような関数でドキュメントを
変更することができるためです（HTML仕様の [解析モデルの概要](https://html.spec.whatwg.org/multipage/parsing.html#overview-of-the-parsing-model)
は良い図が載っています）。 よってHTMLパーサーは、HTMLドキュメントの解析を再開す
る前にJavaScriptの実行を待たなければなりません。JavaScriptの実行で何が起きるのか
知りたい場合は、[V8チームがこれに関するトークとブログを行っています](https://mathiasbynens.be/notes/shapes-ics)。

## ブラウザにどのようにリソースをロードしてほしいのか伝える

Web開発者がリソースをうまくロードするためにヒントをブラウザにあたえる方法はたく
さんあります。JavaScriptが `document.write()`を使用していない場合は、
`<script>`タグに `async`属性または `defer`属性を追加できます。ブラウザは
JavaScriptコードを非同期にロードして実行し、解析をブロックしません。適宜、
[JavaScriptモジュール](/web/fundamentals/primers/modules)を使うこともできます。
`<link rel="preload">`は、現在のナビゲーションにリソースが確実に必要であることや
できるだけ早くダウンロードしたいことをブラウザに通知する方法です。この詳細につい
ては、[リソースの優先順位付け - ブラウザによる支
援](/web/fundamentals/performance/resource-prioritization)を参照してください。

## スタイルの計算

CSSを使ってページ要素をスタイルすることができるので、DOMを持つだけではページがど
のように見えるかを知るには不十分です。メインスレッドはCSSを解析し、各DOMノードに
計算されたスタイル(computed style)を持ちます。これは、CSSセレクターに基づいて各
要素にどのようなスタイルが適用されるかについての情報です。この情報はDevToolsの
`calculate`セクションで見ることができます。

<figure>
  <img src="/web/updates/images/inside-browser/part3/computedstyle.png" alt="計算されたスタイル">
  <figcaption>
    図3: CSSを解析して計算スタイルを追加するメインスレッド
  </figcaption>
</figure>

CSSを提供しなくても、各DOMノードは計算されたスタイル(computed style)を持ちます。
`<h1>`タグは `<h2>`タグより大きく表示され、余白は各要素に対して定義されます。こ
れは、ブラウザにデフォルトのスタイルシートがあるためです。 Chromeの既定のCSSがど
のようなものかを知りたい場合は、[ここでソースコードを見ることができます](https://cs.chromium.org/chromium/src/third_party/blink/renderer/core/css/html.css)
。

## レイアウト

これで、レンダラープロセスはドキュメントの構造と各ノードのスタイルを認識しますが、
それだけではページをレンダリングするのに十分ではありません。あなたが電話であなた
の友人に絵を描こうとしていると想像してください。「大きな赤い丸と小さな青い四角が
あります」とは、あなたの友人が絵が正確にどのように見えるかを知るのに十分な情報に
はなりません。

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part3/tellgame.png" alt="ファックスになってみるゲーム">
  <figcaption>
    図4: 絵の前に立っている人、他の人につながる電話線
  </figcaption>
</figure>

レイアウトは要素の配置を見つけるためのプロセスです。 メインスレッドはDOMと計算さ
れたスタイルを調べ、x、y座標、バウンディングボックスサイズなどの情報を持つレイア
ウトツリーを作成します。レイアウトツリーはDOMツリーと似た構造になることがありま
すが、ページに表示される内容に関連する情報のみが含まれています。`display：
none`が適用されている場合、その要素はレイアウトツリーの一部ではありません（ただ
し、`visibility：hidden`を持つ要素はレイアウトツリーに含まれます）。 同様に、
`p::before{content:"Hi!"}`のような内容を持つ疑似クラスが適用されると、それがDOM
になくてもレイアウトツリーに含まれます。

<div class="clearfix"></div>

<figure>
  <img src="/web/updates/images/inside-browser/part3/layout.png" alt="レイアウト">
  <figcaption>
    図5: メインスレッドは計算スタイルを使ってDOMツリーの上を通り、レイアウトツリー
        を生成します
  </figcaption>
</figure>

<figure class="attempt-right">
  <a href="/web/updates/images/inside-browser/part3/layout.mp4">
    <video src="/web/updates/images/inside-browser/part3/layout.mp4"
           autoplay loop muted playsinline controls alt="改行レイアウト">
    </video>
  </a>
  <figcaption>
    図6: 改行のために移動する段落のボックスレイアウト
  </figcaption>
</figure>

ページのレイアウトを決定するのは難しい作業です。上から下へのブロックフローのよう
な最も単純なページレイアウトでも、段落のサイズと形状に影響を与えるため、フォント
の大きさと改行位置を考慮する必要があります。それは次の段落がどこにあるべきかに影
響します。

CSSは要素を片側にフロートさせ、オーバーフロー項目をマスクし、書き込み方向を変更
できます。想像できるように、このレイアウトをするにはかなり骨が折れる作業です。
Chromeでは、エンジニアチーム全員がレイアウトに取り組んでいます。あなたが彼らの仕
事の詳細を見たいならば、[BlinkOn会議のいくつかの会議](https://www.youtube.com/watch?v=Y5Xa4H2wtVA)
が記録されていて、見るのが非常に面白いです。

<div class="clearfix"></div>
## ペイント

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part3/drawgame.png" alt="描画ゲーム">
  <figcaption>
    図7: キャンバスの前に絵筆を持っている人。先に円を描くか四角を描くか
  </figcaption>
</figure>

DOM、スタイル、およびレイアウトを持つだけでは、ページをレンダリングするのに十分
ではありません。あなたが絵を複製しようとしているとしましょう。あなたは要素のサイ
ズ、形、そして位置を知っています、しかしあなたはまだそれらを描く順番を判断しなけ
ればなりません。

例えば、 `z-index`が特定の要素に設定されているかもしれません、その場合、HTMLで書
かれた要素の順序でペイントすることは誤ったレンダリングをもたらすでしょう。

<div class="clearfix"></div>

<figure>
  <img src="/web/updates/images/inside-browser/part3/zindex.png" alt="z-indexの失敗">
  <figcaption>
    図8: HTML要素のマークアップの順序でページ要素が表示され、z-indexが考慮されていなかった
        ために誤ったレンダリング画像が表示される
  </figcaption>
</figure>

このペイントステップでは、メインスレッドがレイアウトツリーをたどってペイントレコー
ドを作成します。レコードのペイントは、「背景、テキスト、そして長方形」のようなペ
イント処理の注意です。 JavaScriptを使って `<canvas>`要素を描いたことがあるなら、
このプロセスはあなたにはおなじみのものでしょう。

<figure>
  <img src="/web/updates/images/inside-browser/part3/paint.png" alt="paint records">
  <figcaption>
    図9: メインスレッドはレイアウトツリーをたどり、ペイントレコードを作成します
  </figcaption>
</figure>

### レンダリングパイプラインの更新はコストがかかります

<figure class="attempt-right">
  <a href="/web/updates/images/inside-browser/part3/trees.mp4">
    <video src="/web/updates/images/inside-browser/part3/trees.mp4"
           autoplay loop muted playsinline controls alt="DOMとスタイル、レイアウト、そしてペイントツリー">
    </video>
  </a>
  <figcaption>
    図10: 生成された順番に並ぶDOMとスタイル、レイアウト、そしてペイントツリー
  </figcaption>
</figure>

レンダリングパイプラインで最も重要なことは、各ステップで前の操作の結果を使用して
新しいデータが作成されることです。たとえば、レイアウトツリーで何かが変更された場
合は、ドキュメントの影響を受けている各部分に対してペイント順を再生成する必要があ
ります。

<div class="clearfix"></div>

要素にアニメーションを使用している場合、ブラウザはすべてのフレームの間にこれらの
操作を実行する必要があります。私たちのディスプレイのほとんどは、画面を1秒間に60
回更新します（60fps）。すべてのフレームで画面上で物事を動かしているとき、アニメー
ションは人間の目には滑らかに見えます。ただし、アニメーションがその間のフレームを
見逃した場合、そのページは「ぎくしゃく」したように見えます。

<figure>
  <img src="/web/updates/images/inside-browser/part3/pagejank1.png" 
       alt="jage jank by missing frames">
  <figcaption>
    図11: タイムライン上のアニメーションフレーム
  </figcaption>
</figure>

レンダリング操作が画面の更新に追いついていなくても、これらの計算はメインスレッド
で実行されているため、アプリケーションがJavaScriptを実行しているときにはブロック
される可能性があります。

<figure>
  <img src="/web/updates/images/inside-browser/part3/pagejank2.png" alt="jage jank by JavaScript">
  <figcaption>
    図12: タイムライン上のアニメーションフレーム、ただし1フレームはJavaScriptによってブロックされている
  </figcaption>
</figure>

`requestAnimationFrame()`を使用してJavaScriptの操作を小さな塊に分割し、毎フレー
ムで実行するようにスケジュールすることができます。このトピックに関する詳細は、
[JavaScript実行の最適化](/web/fundamentals/performance/rendering/optimize-javascript-execution)
をご覧ください。メインスレッドがブロックされないように、
[WebワーカーのJavaScript](https://www.youtube.com/watch?v=X57mh8tKkgE)を実行することもできます。

<figure>
  <img src="/web/updates/images/inside-browser/part3/raf.png" alt="request animation frame">
  <figcaption>
    図13: アニメーションフレームのあるタイムライン上で実行されるJavaScriptの小さな塊
  </figcaption>
</figure>

## コンポジティング

### どのようにしてページを描きますか？

<figure class="attempt-right">
  <a href="/web/updates/images/inside-browser/part3/naive_rastering.mp4">
    <video src="/web/updates/images/inside-browser/part3/naive_rastering.mp4"
           autoplay loop muted playsinline controls alt="ネイティブラスタリング">
    </video>
  </a>
  <figcaption>
    図14: ネイティブラスタリングプロセスのアニメーション
  </figcaption>
</figure>

ドキュメントの構造、各要素のスタイル、ページのジオメトリ、そしてペイントの順番を
知ることにより、ブラウザはどうやってページを描画するのでしょうか。この情報を画面
上でピクセルに変換することをラスタライズと呼びます。

おそらくこれを処理する単純な方法は、ビューポートの内側にパーツをラスタ化すること
です。ユーザーがページをスクロールする場合は、ラスタフレームを移動し、さらにラス
タすることによって欠けている部分を埋めます。これがChromeが最初のリリース時にラス
タライズを処理した方法です。しかし、最近のブラウザはコンポジティングと呼ばれるよ
り洗練されたプロセスを実行します。

<div class="clearfix"></div>

### コンポジティングとは

<figure class="attempt-right">
  <a href="/web/updates/images/inside-browser/part3/composit.mp4">
    <video src="/web/updates/images/inside-browser/part3/composit.mp4"
           autoplay loop muted playsinline controls alt="コンポジティング">
    </video>
  </a>
  <figcaption>
    図15: コンポジティングのアニメーション
  </figcaption>
</figure>

コンポジティングとは、ページの一部をレイヤーに分割し、それらを別々にラスタライズ
し、ページとしてコンポジットスレッドと呼ばれる別のスレッドに合成する技術です。ス
クロールが発生した場合、レイヤーは既にラスタライズされているので、新しいフレーム
を合成するだけです。アニメーションは、レイヤーを移動して新しいフレームを合成する
という同じ方法で実現されています。

あなたはDevToolsの[レイヤーパネル](https://blog.logrocket.com/eliminate-content-repaints-with-the-new-layers-panel-in-chrome-e2c306d4d752?gi=cd6271834cea)
を使ってウェブサイトがどのようにレイヤーに分割されているかを見ることができます 。

<div class="clearfix"></div>

### レイヤーに分ける

どの要素がどのレイヤにある必要があるかを見つけるために、メインスレッドはレイアウ
トツリーをたどってレイヤツリーを作成します（この部分はDevToolsパフォーマンスパネ
ルでは「Update Layer Tree」と呼ばれます）。別のレイヤーにする必要があるページの
特定の部分（スライドインサイドメニューなど）がはっきりしない場合は、CSSの
`will-change`属性を使用してブラウザにヒントを与えることができます。

<figure>
  <img src="/web/updates/images/inside-browser/part3/layer.png" alt="レイヤーツリー">
  <figcaption>
    図16: レイアウトツリーをたどるメインスレッド
  </figcaption>
</figure>

すべての要素にレイヤーを付けたいと思うかもしれませんが、余分な数のレイヤーにまた
がって合成すると、フレームごとにページの小さい部分をラスタライズするよりも操作が
遅くなる可能性があるため、アプリケーションのレンダリングパフォーマンスを測定する
ことが重要です。トピックの詳細については、[合成用プロパティにこだわり、レイヤー
を管理す
る
](/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count)
を参照してください。

### メインスレッドのラスタとコンポジット

レイヤツリーが作成されてペイントの順序が決定されると、メインスレッドはその情報を
コンポジットスレッドにコミットします。次にコンポジットスレッドは各レイヤーをラス
タライズします。レイヤーはページ全体の長さのように大きくなることがあるため、合成
スレッドはそれらをタイルに分割し、各タイルをラスタスレッドに送ります。ラスタスレッ
ドは各タイルをラスタライズしてGPUメモリに保存します。

<figure>
  <img src="/web/updates/images/inside-browser/part3/raster.png" alt="ラスター">
  <figcaption>
    図17: タイルのビットマップを作成してGPUに送信するラスタースレッド
  </figcaption>
</figure>

コンポジットスレッドは、ビューポート内（または近く）のものが最初にラスタ処理され
るように、異なるラスタスレッドに優先順位を付けることができます。レイヤーには、ズー
ムインアクションのようなものを処理するために、さまざまな解像度に対して複数の傾斜
をつけることもあります。

タイルがラスタ化されると、コンポジットスレッドは**描画ガード**と呼ばれるタイル情
報を収集した**コンポジットフレーム**を作成します。

<table class="responsive">
  <tr>
    <td>描画ガード</td>
    <td>
      ページの合成を処理するのに、メモリ内のタイルの位置やタイルを描画するページ
      内の位置などの情報が含まれています。
    </td>
  </tr>
  <tr>
    <td>コンポジットフレーム</td>
    <td>ページのフレームを表す描画ガードのコレクション。</td>
  </tr>
</table>

次に、コンポジットフレームがIPCを介してブラウザプロセスに送信されます。この時点
で、ブラウザのUIを変更するためのUIスレッドや拡張機能のための他のレンダラープロセ
スから別のコンポジットフレームが追加されます。これらのコンポジットフレームはGPU
に送信されて画面に表示されます。スクロールイベントが発生した場合、コンポジットス
レッドはGPUに送信される別のコンポジットフレームを作成します。

<figure>
  <img src="/web/updates/images/inside-browser/part3/composit.png" alt="composit">
  <figcaption>
    図18: コンポジットフレームを作成するコンポジットスレッド。 フレームは
          ブラウザプロセスに送られ、次にGPUに送られます。
  </figcaption>
</figure>

コンポジティングの利点は、メインスレッドを使用せずに行われることです。 コンポジッ
トスレッドは、スタイルの計算やJavaScriptの実行を待つ必要はありません。これが、
「アニメーションのみを合成する」ことがスムーズなパフォーマンスのために最善と考え
られる理由です。レイアウトやペイントを再計算する必要がある場合は、メインスレッド
を使用する必要があります。

## さいごに

この記事では、構文解析から合成までのレンダリング・パイプラインについて説明しまし
た。あなたがウェブサイトのパフォーマンスの最適化についてもっと詳しく知れるよう願っ
ています。

この連載の次回、最後の投稿では、コンポジットスレッドをもっと詳しく調べ、「マウス
移動」や「クリック」などのユーザー入力が入ったときに何が起こるかを確認します。

この記事は楽しめましたか？今後の投稿について質問や提案がある場合は、下記のコメン
ト欄またはTwitterの[@kosamari](https://twitter.com/kosamari)までご連絡ください。

<a class="button button-primary gc-analytics-event attempt-right"
   href="/web/updates/2018/09/inside-browser-part4"
   data-category="InsideBrowser" data-label="Part3 / Next">
  次へ: 入力がコンポジットにくるまで
</a>

<div class="clearfix"></div>

## フィードバック {: .hide-from-toc }

{% include "web/_shared/helpful.html" %}

<div class="clearfix"></div>

{% include "web/_shared/rss-widget-updates.html" %}

{% include "comment-widget.html" %}
