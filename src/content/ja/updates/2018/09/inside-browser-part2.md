project_path: /web/_project.yaml
book_path: /web/updates/_book.yaml
description: ブラウザがナビゲーション要求を処理する方法を学びます。

{# wf_published_on: 2018-09-07 #}
{# wf_updated_on: 2019-01-10 #}
{# wf_featured_image: /web/updates/images/inside-browser/cover.png #}
{# wf_featured_snippet: アドレスバーにURLを入力すると、その後はどうなりますか？セキュリティチェックはいつ行われ、ブラウザはプロセスをどのようにスピードアップしますか？ブラウザでページナビゲーションプロセスを見てみましょう！ #}
{# wf_blink_components: N/A #}

<style>
  figcaption {
    font-size:0.9em;
  }
</style>

# モダンブラウザの内部を見る (パート2) {: .page-title }

{% include "web/_shared/contributors/kosamari.html" %}


## ナビゲーションで起こること

これはChromeの内部の仕組みを見ていく計4回のブログ連載の2回目です。
[前回の記事](/web/updates/2018/09/inside-browser-part1)では、さまざまなプロセスやスレッド
がブラウザのさまざまな部分をどのように処理するかを調べました。この記事では、Webサイトを表
示するために各プロセスとスレッドがどのように通信するのかを詳しく説明します。

Webブラウジングの簡単な使用例を見てみましょう。ブラウザにURLを入力すると、ブラウザはインター
ネットからデータを取得してページを表示します。この記事では、ユーザーがサイトを要求し、ブラ
ウザがページのレンダリングを準備する部分（ナビゲーションとも呼ばれる）に焦点を当てます。

## まずはブラウザプロセスから始まります

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part2/browserprocesses.png" alt="ブラウザプロセス">
  <figcaption>
    図1: 上部のブラウザUI、下部のUI、ネットワーク、およびストレージスレッドを含むブラウザプロセスの図
  </figcaption>
</figure>

[パート1: CPU、GPU、メモリ、およびマルチプロセスの仕組
み](/web/updates/2018/09/inside-browser-part1)で説明したように、タブの外側にあるものはすべ
てブラウザプロセスによって処理されます。
ブラウザプロセスには、ブラウザのボタンや入力フィールドを描画するUIスレッド、インターネット
からデータを受け取るためにネットワークスタックを扱うネットワークスレッド、ファイルへのアク
セスを制御するストレージスレッドなどのスレッドがあります。アドレスバーにURLを入力すると、
入力はブラウザプロセスのUIスレッドによって処理されます。

<div class="clearfix"></div>

## 簡単なナビゲーション

### ステップ1: 入力処理

ユーザーがアドレスバーに入力し始めると、UIスレッドは最初に「これは検索クエリですか？それと
もURLですか？」と尋ねます。 Chromeでは、アドレスバーは検索入力フィールドでもあるため、UIス
レッドは、ユーザーを検索エンジンに送信するか、リクエストしたサイトに送信するかを解析して決
定する必要があります。

<figure>
  <img src="/web/updates/images/inside-browser/part2/input.png" alt="ユーザー入力の処理">
  <figcaption>
    図1: 入力が検索クエリかURLかを尋ねるUIスレッド
  </figcaption>
</figure>

### ステップ2: ナビゲーションを開始

ユーザーがEnterキーを押すと、UIスレッドはサイトコンテンツを取得するためのネットワーク呼び
出しを開始します。ローディングスピナーがタブの隅に表示され、ネットワークスレッドはDNSルッ
クアップや要求に対するTLS接続の確立などの適切なプロトコルを通過します。

<figure>
  <img src="/web/updates/images/inside-browser/part2/navstart.png" alt="ナビゲーション開始">
  <figcaption>
    図2: mysite.comにナビゲートするためにネットワークスレッドと通信するUIスレッド
  </figcaption>
</figure>

この時点で、ネットワークスレッドは、HTTP 301のよ​​うなサーバリダイレクトヘッダを受信すること
ができます。その場合、ネットワークスレッドは、サーバがリダイレクトを要求していることをUIス
レッドと通信します。その後、別のURLリクエストが開始されます。

### ステップ3: 読み取り応答

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part2/response.png" alt="HTTPレスポンス">
  <figcaption>
    図3: Content-Typeと実際のデータであるペイロードを含むレスポンスヘッダ
  </figcaption>
</figure>

応答の本体（ペイロード）が入ってくると、ネットワークスレッドは必要に応じてストリームの最初
の数バイトを調べます。レスポンスのContent-Typeヘッダは、それがどんな種類のデータであるかを
示すべきですが、それが欠けているか間違っているかもしれないので、ここでは [MIMEタイプ
チェック](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)を行います。
[ソースコード](https://cs.chromium.org/chromium/src/net/base/mime_sniffer.cc?sq=package:chromium&dr=CS&l=5)
でコメントされているように、これは「きわどい仕事」です。コメントを読んで、さまざまなブラウ
ザがコンテンツタイプとペイロードのペアをどのように扱うかを確認できます。

<div class="clearfix"></div>

応答がHTMLファイルの場合、次のステップはデータをレンダラープロセスに渡すことですが、それが
zipファイルまたはその他のファイルの場合は、ダウンロード要求であり、ダウンロードマネージャ
にデータを渡す必要があります。

<figure>
  <img src="/web/updates/images/inside-browser/part2/sniff.png" alt="MIMEタイプチェック">
  <figcaption>
    図4: 応答データが安全なサイトからのHTMLかどうかを尋ねるネットワークスレッド
  </figcaption>
</figure>

これは[セーフブラウジング](https://safebrowsing.google.com/)チェックが行われる場所でもあり
ます。ドメインとレスポンスデータが悪意のある既知のサイトと一致すると思われる場合、ネットワー
クスレッドは警告ページを表示するように警告します。さらに、機密のクロスサイトデータがレンダ
ラープロセスに影響を与えないようにするために、[**C**ross **O**rigin **R**ead **B**locking
(**CORB**)](https://www.chromium.org/Home/chromium-security/corb-for-developers) チェック
が行われます。

### ステップ3: レンダラープロセスを探す

すべてのチェックが完了し、ネットワークスレッドがブラウザが要求されたサイトに移動する必要が
あることを確信すると、ネットワークスレッドはデータが準備できたことをUIスレッドに伝えます。
その後、UIスレッドはWebページのレンダリングを続行するためのレンダラープロセスを見つけます。

<figure>
  <img src="/web/updates/images/inside-browser/part2/findrenderer.png" alt="レンダラープロセスを探す">
  <figcaption>
    図5: UIスレッドにレンダラープロセスを見つけるように指示するネットワークスレッド
  </figcaption>
</figure>

ネットワーク要求が応答を返すまでに数百ミリ秒かかることがあるので、このプロセスを高速化する
ための最適化が適用されます。ステップ2でUIスレッドがURL要求をネットワークスレッドに送信して
いるときは、どのサイトに移動しているのかは既にわかっています。 UIは、ネットワーク要求と並
行してレンダラプロセスを予防的に検索または開始するためのスレッドです。このようにして、すべ
てが期待通りに動作する場合、ネットワークスレッドがデータを受信したときにレンダラープロセス
はすでにスタンバイの位置にあります。ナビゲーションがサイト間をリダイレクトする場合、このス
タンバイプロセスは使用されないかもしれません。


### ステップ4: ナビゲーションをコミット

データとレンダラープロセスの準備ができたので、ナビゲーションをコミットするためにIPCがブラ
ウザプロセスからレンダラープロセスに送信されます。また、レンダラープロセスがHTMLデータを受
信し続けることができるようにデータストリームを渡します。ブラウザプロセスが、レンダラープロ
セスでコミットを確認すると、ナビゲーションは完了し、ドキュメントの読み込みフェーズが開始さ
れます。

この時点で、アドレスバーが更新され、セキュリティインジケータとサイト設定のUIに新しいページ
のサイト情報が反映されます。タブのセッション履歴が更新されるため、戻る/進むボタンで移動し
たばかりのサイトに移動します。タブまたはウィンドウを閉じるときのタブ/セッションの復元を容
易にするために、セッション履歴はディスクに保存されます。

<figure>
  <img src="/web/updates/images/inside-browser/part2/commit.png" alt="ナビゲーションをコミット">
  <figcaption>
    Figure 6: ブラウザとレンダラプロセス間のIPC。ページのレンダリングを要求します。
  </figcaption>
</figure>

### 追加ステップ: 初期ロード完了

ナビゲーションがコミットされると、レンダラープロセスはリソースのロードを続け、ページをレン
ダリングします。次の投稿では、この段階で何が起こるのかについて詳しく説明します。レンダラー
プロセスがレンダリングを「完了」すると、ブラウザプロセスにIPCを送り返します（これは、すべ
ての `onload`イベントがページ内のすべてのフレームで発生し、実行が終了した後です）。この時
点で、UIスレッドはタブ上のローディングスピナーを停止します。

「完了」といったのは、この後でもクライアント側のJavaScriptは追加のリソースをロードして新し
いビューをレンダリングする可能性があるためです。

<figure>
  <img src="/web/updates/images/inside-browser/part2/loaded.png" alt="ページのロード完了">
  <figcaption>
    図7: ページに「ロード」されたことを通知するための、レンダラからブラウザプロセスへのIPC
  </figcaption>
</figure>

## 別のサイトに移動する

簡単なナビゲーションは完了しました！しかし、ユーザーがアドレスバーに別のURLを入力した場合
はどうなるでしょうか、ご想像の通り、ブラウザのプロセスは別のサイトに移動するために同じ手順
を経ます。ただし、その前に
[beforeunload](https://developer.mozilla.org/en-US/docs/Web/Events/beforeunload)イベントを
使うかどうかを現在レンダリングされているサイトで確認されます。

`beforeunload`はタブを移動しようとしたり閉じたりしようとするとする時に「このサイトを離れま
すか？ 」というアラートを出すこことができます。JavaScriptコードを含むタブ内のものはすべて
レンダラープロセスによって処理されるため、新しいナビゲーション要求が発生したときにブラウザ
プロセスは現在のレンダラープロセスを確認する必要があります。

注意: 無条件に `beforeunload`ハンドラを追加しないでください。ナビゲーションを開始する前に
ハンドラを実行する必要があるため、待ち時間が長くなります。このイベントハンドラは、必要な場
合にのみ追加する必要があります。たとえば、ユーザーがページに入力したデータを失う可能性があ
ることをユーザーに警告する必要がある場合などです。

<figure>
  <img src="/web/updates/images/inside-browser/part2/beforeunload.png" 
    alt="beforeunloadイベントハンドラ">
  <figcaption>
    図8: ブラウザプロセスからレンダラープロセスへのIPCで、別のサイトに移動することを伝える
  </figcaption>
</figure>

ナビゲーションがレンダラープロセスから開始された場合（ユーザーがリンクをクリックした場合や
クライアントサイドのJavaScriptが `window.location ="https://newsite.com"`を実行した場合）、
レンダラープロセスはまず `beforeunload`ハンドラーをチェックします。その後、ブラウザプロセ
スがナビゲーションを開始したのと同じプロセスを経ます。唯一の違いは、ナビゲーション要求がレ
ンダラープロセスからブラウザプロセスへと開始されることです。

新しいナビゲーションが現在レンダリングされているサイトとは異なるサイトに作成された場合、新
しいナビゲーションを処理するために別のレンダリングプロセスが呼び出され、現在のレンダリング
プロセスはunloadなどのイベントを処理するために保持されます。詳細については、「[ページライ
フサイクル状態の概要](/web/updates/2018/07/page-lifecycle-api#overview_of_page_lifecycle_states_and_events)」
および「[ページライフサイクルAPI](/web/updates/2018/07/page-lifecycle-api)」を使用してイベ
ントにフックする方法を参照してください。

<figure>
  <img src="/web/updates/images/inside-browser/part2/unload.png" alt="新しいナビゲーションとアンロード">
  <figcaption>
    図9: ブラウザプロセスから新しいレンダラープロセスへの2ページのIPCで、ページのレンダリング
      を指示し、古いレンダラープロセスにアンロードを指示します。
  </figcaption>
</figure>

## Service Workerの場合

このナビゲーションプロセスに対する最近の変更の1つは、「[Service
Worker](/web/fundamentals/primers/service-workers/)」の導入です。Service Workerはあなたの
アプリケーションコードでネットワークプロキシを書く方法です。 Web開発者が、何をローカルに
キャッシュし、いつネットワークから新しいデータを取得するかをより詳細に制御できるようにしま
す。Service Workerがキャッシュからページをロードするように設定されている場合は、ネットワー
クからデータを要求する必要はありません。

覚えておくべき重要な部分は、Service Workerがレンダラープロセスで実行されるJavaScriptコード
であるということです。しかし、ナビゲーション要求が入ったとき、ブラウザプロセスはどのように
してそのサイトにService Workerが登録されているのかを知るのでしょうか。

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part2/scope_lookup.png" 
    alt="Service Workerのスコープ検索">
  <figcaption>
    図10: Service Workerスコープを検索するブラウザプロセス内のネットワークスレッド
  </figcaption>
</figure>

Service Workerが登録されると、Service Workerの参照範囲が保持されます（この「[Service
Workerのライフサイクル](/web/fundamentals/primers/service-workers/lifecycle)」の記事で範囲
の詳細について読むことができます）。

ナビゲーションが発生すると、ネットワークスレッドは登録されているService Workerスコープに対
してドメインをチェックします。Service WorkerがそのURLに登録されている場合、UIスレッドは
Service Workerコードを実行するためにレンダラープロセスを見つけます。Service Workerは、ネッ
トワークからデータを要求することなくキャッシュからデータをロードすることができ、もしくは常
にネットワークから新しいリソースを要求することもできます。

<figure>
  <img src="/web/updates/images/inside-browser/part2/serviceworker.png" 
    alt="serviceworker navigation">
  <figcaption>
    図11: Service Workerを処理するためにレンダラープロセスを起動するブラウザプロセスのUIス
       レッド。レンダラープロセスのワーカースレッドは、ネットワークからデータを要求します。
  </figcaption>
</figure>

## ナビゲーションプリロード

Service Workerが最終的にネットワークからデータを要求する場合、ブラウザープロセスとレンダ
ラープロセスの往復の結果、遅延が生じる場合があります。「[ナビゲーションプリロー
ド](/web/updates/2017/02/navigation-preload)」は、Service Workerの起動と並行してリソース
をロードすることによってこのプロセスを高速化するためのメカニズムです。それはヘッダのリクエ
ストにマークをつけ、サーバがこれらのリクエストのために異なるコンテンツを送信することを可能
にします。たとえば、文書全体ではなく一部のデータを更新したいだけの場合です。

<figure>
  <img src="/web/updates/images/inside-browser/part2/navpreload.png" alt="Navigation preload">
  <figcaption>
    図12: ネットワーク要求を並行して開始しながらService Workerを処理するためにレンダラープロセスを起動するブラウザプロセスのUIスレッド
  </figcaption>
</figure>

## さいごに

この投稿では、ナビゲーション中に何が起こるのか、そしてレスポンスヘッダやクライアントサイド
JavaScriptのようなあなたのWebアプリケーションコードがブラウザとどのようにやり取りするのか
を調べました。ブラウザがネットワークからデータを取得するための手順を知っていると、ナビゲー
ションプリロードなどのAPIが開発された理由を理解しやすくなります。次の投稿では、ブラウザが
どのようにHTML/CSS/JavaScriptを評価してページをレンダリングするかについて詳しく説明します。

この投稿は楽しめましたか？今後の投稿について質問や提案がある場合は、下記のコメント欄または
Twitterの[@kosamari](https://twitter.com/kosamari)までご連絡ください。

<a class="button button-primary gc-analytics-event attempt-right"
   href="/web/updates/2018/09/inside-browser-part3"
   data-category="InsideBrowser" data-label="Part2 / Next">
  次へ: レンダラープロセスの内部動作
</a>

<div class="clearfix"></div>

## フィードバック {: .hide-from-toc }

{% include "web/_shared/helpful.html" %}

<div class="clearfix"></div>

{% include "web/_shared/rss-widget-updates.html" %}

{% include "comment-widget.html" %}
