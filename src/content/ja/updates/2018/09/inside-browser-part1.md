project_path: /web/_project.yaml
book_path: /web/updates/_book.yaml
description: 上位アーキテクチャからレンダリングパイプラインの詳細に至るまで、ブラウザがどのようにしてコードを機能的なWebサイトに変えるかを学びます。

{# wf_published_on: 2018-09-05 #}
{# wf_updated_on: 2019-01-20 #}
{# wf_featured_image: /web/updates/images/inside-browser/cover.png #}
{# wf_featured_snippet: 上位アーキテクチャからレンダリングパイプラインの詳細にいたるまで、ブラウザがどのようにしてコードを機能的なWebサイトに変換していくのかを学びます。パート1では、コアコンピューティング用語とChromeのマルチプロセスアーキテクチャについて説明します。 #}
{# wf_blink_components: N/A #}

<style>
  figcaption {
    font-size:0.9em;
  }
</style>

# モダンブラウザの内部を見る (パート1) {: .page-title }

{% include "web/_shared/contributors/kosamari.html" %}

## CPU・GPU・メモリおよびマルチプロセスの仕組み

この計4回のブログ連載では、上位アーキテクチャからレンダリングパイプラインの詳細にいたるま
で、Chromeブラウザの内部について説明します。ブラウザがどのようにしてコードを機能的なWebサ
イトに変換していくのか疑問に思ったことがある場合、パフォーマンスの向上のために推奨されてい
る特定の手法についてその理由がよくわからない場合は、この連載を参照してください。

この連載の第1回として、コアとなるコンピュータ用語とChromeのマルチプロセス・アーキテクチャ
について説明します。

Note: CPU/GPUおよびプロセス/スレッドの概念に精通している場合は、[ブラウザの仕組み](#browser-architecture)に進んでください。

## コンピュータの中核にあるのはCPUとGPUです

ブラウザが実行されている環境を理解するために、いくつかのコンピュータの部品及びそれらが何を
するのか理解する必要があります。

### CPU

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part1/CPU.png" alt="CPU">
  </a>
  <figcaption>
    図1: 各デスクに座っている事務作業者としての4つのCPUコア
  </figcaption>
</figure>

はじめは **C**entral **P**rocessing **U**nit - もしくは**CPU**です。CPUはコンピュータの頭
脳と見なすことができます。ここではオフィスワーカーとして描かれているCPUのコアが、さまざま
なタスクを1つずつ処理しています。顧客の電話への返信方法を知りつつも、数学から芸術まですべ
てを処理できます。過去のほとんどのCPUはシングルチップでした。コアとは、同じチップ内に存在
する別のCPUのようなものです。最近のハードウェアでは、多くの場合、複数のコアを使用し、携帯
電話やノートPCの処理能力を高めています。

<div class="clearfix"></div>

### GPU

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part1/GPU.png" alt="GPU">
  <figcaption>
    図2: レンチを持ったたくさんのGPUコアが小さなタスクを処理するイメージ
  </figcaption>
</figure>

**G**raphics **P**rocessing **U**nit - もしくは**GPU**はもう一つのコンピュータの部品です。
CPUとは異なり、GPUは単純なタスクを扱うのが得意ですが、同時に複数のコアにまたがっています。
名前が示すように、それは最初グラフィックを処理するために開発されました。これがグラフィック
関連の用語として"using GPU(GPUを使用する)"や"GPU-backed(GPUを活用した)"が、高速なレンダリ
ングと円滑なインタラクションを示している理由です。近年では、GPUが高速になっていくことで、
GPU単独でますます多くのコンピューティングが可能になりつつあります。

<div class="clearfix"></div>

コンピュータや携帯電話でアプリケーションを起動すると、CPUとGPUが作動します。通常、アプリケー
ションはオペレーティングシステムによって提供されるメカニズムを使用してCPUとGPU上で実行され
ます。

<figure>
  <img src="/web/updates/images/inside-browser/part1/hw-os-app.png" alt="ハードウェア, OS, アプリケーション">
  <figcaption>
   図3: 3層のコンピュータアーキテクチャ。一番下はハードウェア、中間のオペレーティングシステム、
     一番上はアプリケーション。
  </figcaption>
</figure>

## プロセスとスレッドでプログラムを実行する

<figure class="attempt-right">
  <img src="/web/updates/images/inside-browser/part1/process-thread.png" alt="プロセスとスレッド">
  <figcaption>
  図4: プロセスは水槽のように、スレッドが魚のように泳いでいる
  </figcaption>
</figure>

ブラウザアーキテクチャに入る前に把握する必要があるもう1つの概念は、プロセスとスレッドです。
プロセスは、アプリケーションの実行プログラムとして記述されます。 スレッドはプロセス内に存
在し、そのプロセスのプログラムの任意の部分を実行するものです。

アプリケーションを起動すると、プロセスが作成されます。プログラムは必要であればスレッドを作
成するかもしれませんが、あくまでオプションです。オペレーティングシステムは、プロセスに作業
用の「部分的な」メモリを提供し、アプリケーションの状態はすべてそのプライベートメモリ空間に
保持されます。アプリケーションを閉じると、プロセスも終了し、オペレーティングシステムはメモ
リを解放します。

<div class="clearfix"></div>

<figure>
  <a href="/web/updates/images/inside-browser/part1/memory.svg">
    <img src="/web/updates/images/inside-browser/part1/memory.png" alt="プロセスとメモリ">
  </a>
  <b>
    <span class="material-icons">play_circle_outline</span>画像をクリックしてアニメーションを再生
  </b>
  <figcaption>
    図5: メモリ空間を使用してアプリケーションデータを格納するプロセスの図
  </figcaption>
</figure>

プロセスは、異なるタスクを実行するために別のプロセスを起動するようにオペレーティングシステ
ムへ依頼することができます。この依頼により、メモリのさまざまな部分が新しいプロセスに割り当
てられます。2つのプロセスが通信する必要がある場合は、**I**nter **P**rocess
**C**ommunication (**IPC**)を使用して通信します。多くのアプリケーションはこのように動作が
設計されているため、ワーカープロセスが応答しなくなっても、アプリケーションのさまざまな部分
を実行している他のプロセスを停止することなく再起動できます。

<figure>
  <a href="/web/updates/images/inside-browser/part1/workerprocess.svg">
    <img src="/web/updates/images/inside-browser/part1/workerprocess.png"
      alt="ワーカープロセスとIPC">
  </a>
  <b>
    <span class="material-icons">play_circle_outline</span>画像をクリックしてアニメーションを再生
  </b>
  <figcaption>
    図6: IPCを使って通信する別々のプロセス
  </figcaption>
</figure>


## ブラウザの仕組み {: #browser-architecture }

それでは、プロセスとスレッドを使いながらWebブラウザはどのようにつくられているのでしょうか。
おそらく、多くの異なるスレッドを持つ1つのプロセス、もしくは少数のスレッドを持つ多くの異な
るプロセスで、IPCを使って通信している可能性があります。

<figure>
  <img src="/web/updates/images/inside-browser/part1/browser-arch.png" alt="ブラウザアーキテクチャ">
  <figcaption>
   図7: プロセス/スレッドによるブラウザの仕組みの違い
  </figcaption>
</figure>

ここで注意すべき重要なことは、これらの異なるアーキテクチャは実装上の違いであるということで
す。Webブラウザの構築方法に関する標準仕様はありません。あるブラウザのアプローチは他のもの
とはまったく異なるかもしれません。

このブログ連載では、下の図に示すChromeの最近のアーキテクチャを使用します。

一番上にあるのは、アプリケーションのさまざまな部分を処理する他のプロセスと連携するブラウザ
プロセス(Browser Process)です。レンダラープロセス(Renderer Process)では、複数のプロセスが
作成されて各タブに割り当てられます。ごく最近まで、Chromeは可能な場合は各タブにプロセスを
割り当てていました。現在は各サイトにiframeを含む独自のプロセスを提供しようとしています 
([サイト分離](#site-isolation)を参照)。

<figure>
  <img src="/web/updates/images/inside-browser/part1/browser-arch2.png" alt="ブラウザアーキテクチャ2">
  <figcaption>
   図8: Chromeのマルチプロセスアーキテクチャの図。各タブに対して複数のレンダラープロセスが
   実行されていることを示すために、複数のレイヤーがレンダラープロセスの下に表示されています。
  </figcaption>
</figure>

## 各プロセスは何を制御していますか？

次の表は、Chromeの各プロセスとその制御の仕方について説明しています:

<table class="responsive">
  <tr>
    <th colspan="2">プロセスと制御しているもの</th>
  </tr>
  <tr>
    <td>ブラウザ</td>
    <td>
      アドレスバー、ブックマーク、戻る、進むボタンなど、アプリケーションとしてのChromeを
      制御します。<br>ネットワーク要求やファイルアクセスなど、Webブラウザの見えない特権
      部分も処理します。
    </td>
  </tr>
  <tr>
    <td>レンダラー</td>
    <td>Webサイトが表示されているタブ内のすべてのものを制御します。</td>
  </tr>
  <tr>
    <td>プラグイン</td>
    <td>Webサイトで使用されているすべてのプラグイン（Flashなど）を制御します。</td>
  </tr>
  <tr>
    <td>GPU</td>
    <td>
      他のプロセスから分離してGPUタスクを処理します。 GPUは複数のアプリケーションからの
      要求を処理し、それらを画面上に描画するため、プロセスは異なるプロセスに分けられます。
    </td>
  </tr>
</table>

<figure>
  <img src="/web/updates/images/inside-browser/part1/browserui.png" alt="Chromeプロセス">
  <figcaption>
   図9: ブラウザUIのさまざまな部分を指すさまざまなプロセス
  </figcaption>
</figure>

拡張プロセスやユーティリティプロセスのようなさらに多くのプロセスがあります。Chromeで実行中
のプロセス数を確認するには、右上隅にあるオプションメニューアイコン
<span class="material-icons">more_vert</span>をクリックし、[その他のツール]、[タスクマネー
ジャ]の順に選択します。これにより、現在実行中のプロセスと、それらが使用しているCPU/メモリ
のウィンドウが開きます。

## Chromeのマルチプロセスアーキテクチャの利点

最初のほうで、Chromeは複数のレンダラープロセスを使用すると述べました。最も単純なケースでは、
各タブに独自のレンダラープロセスがあると想像できます。3つのタブが開いていて、各タブが独立
したレンダラープロセスによって実行されているとしましょう。

1つのタブが反応しなくなった場合は、反応しないタブを閉じて他のタブを保持したまま先に進むこ
とができます。すべてのタブが1つのプロセスで実行されている場合、1つのタブが応答しなくなると、
すべてのタブが応答しなくなります。それは悲しいことです。

<figure>
  <a href="/web/updates/images/inside-browser/part1/tabs.svg">
    <img src="/web/updates/images/inside-browser/part1/tabs.png" alt="タブ用の複数のレンダラー">
  </a>
  <b>
    <span class="material-icons">play_circle_outline</span>画像をクリックしてアニメーションを再生
  </b>
  <figcaption>
    図10: 各タブを実行している複数のプロセスを示す図
  </figcaption>
</figure>

ブラウザの作業を複数のプロセスに分割することのもう1つの利点は、セキュリティとサンドボック
ス化です。オペレーティングシステムがプロセスの特権を制限する方法を提供するので、ブラウザは
特定の機能から特定のプロセスをサンドボックスすることができます。たとえば、Chromeブラウザは、
レンダラープロセスのように任意のユーザ入力を処理するプロセスに対する任意のファイルアクセス
を制限します。

プロセスには独自のプライベートメモリ空間があるため、多くの場合、共通のインフラストラクチャ
のコピー（ChromeのJavaScriptエンジンであるV8など）が含まれています。これは、同じプロセス内
のスレッドである場合と同じように共有することができないため、メモリ使用量が増加することを意
味します。メモリを節約するために、Chromeはスピンアップできるプロセス数に制限を設けています。
その制限は、デバイスのメモリとCPUの能力によって異なりますが、Chromeが制限に達すると、同じ
サイトから複数のタブが1つのプロセスで実行され始めます。

## メモリを節約する為に -  Chromeにおけるサービス化

同じアプローチがブラウザプロセスにも適用されています。Chromeはプロセスを簡単に分割したり、
1つにまとめられるように、ブラウザのプログラムの各部分をサービスとして実行するためのアーキ
テクチャ変更を進めています。

おおまかには、Chromeが強力なハードウェア上で実行されている場合、各サービスをさまざまなプロ
セスに分割して安定性を高めますが、リソースが限られているデバイスの場合はChromeでサービスを
1つのプロセスに統合します。この変更以前は、Androidなどのプラットフォームでも、メモリ使用量
を少なくするためにプロセスを統合するという同様のアプローチが採用されていました。

<figure>
  <a href="/web/updates/images/inside-browser/part1/servicfication.svg">
    <img src="/web/updates/images/inside-browser/part1/servicfication.png"
      alt="Chromeのサービス化">
  </a>
  <b>
    <span class="material-icons">play_circle_outline</span>画像をクリックしてアニメーションを再生
  </b>
  <figcaption>
   図11: 単一のブラウザプロセスとさまざまなサービスを提供する複数のプロセスに移行するChrome
     のサービス化
  </figcaption>
</figure>

## フレームごとのレンダラープロセス - サイト分離 {: #site-isolation }

[サイト分離](/web/updates/2018/07/site-isolation)はChromeで最近導入された機能で、クロスサイ
トのiframeごとに別々のレンダラープロセスを実行します。

これまでタブごとにクロスサイトのインラインフレームが許可された特定のレンダラープロセスが、
サイト間でメモリスペースを共有しながら、単一のレンダラープロセスで実行されることを説明しま
した。a.comとb.comを同じレンダラープロセスで実行しても問題ないでしょう。

[同一オリジンポリシー](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)
はWebの基本的なセキュリティモデルです。ある特定のサイトが他のサイトのデータを同意なしにア
クセスできないようにします。セキュリティ攻撃ではこのポリシーを回避することが主要なゴールに
なります。

プロセス分離は、サイトを分離するための最も効果的な方法です。[メルトダウンとスペク
ター](/web/updates/2018/02/meltdown-spectre)ではプロセスでサイトを分離する必要があることが
さらに明らかになりました。Chrome 67以降のデスクトップではサイト分離がデフォルトで有効になっ
ており、タブ内の各クロスサイトインラインフレームは個別のレンダラープロセスを取得します。

<figure>
  <img src="/web/updates/images/inside-browser/part1/isolation.png" alt="site isolation">
  <figcaption>
   図12: サイト分離について。サイト内のiframeが複数のレンダラープロセスを示す
  </figcaption>
</figure>

サイト分離を実現することは、長年にわたるエンジニアリングの成果でした。サイト分離は、異なる
レンダラープロセスを割り当てるほど単純ではありません。それはiframeが互いに通信する方法を根
本的に変えます。 iframeが異なるプロセスで実行されているページでdevtoolsを開くには、
devtoolsがシームレスに見えるような舞台裏作業(behind-the-scenes)を実装する必要がありました。
ページ内の単語を見つけるために単純なCtrl+Fを実行するにしても、さまざまなレンダラープロセス
を横断して検索しなければいけないことを意味します。ブラウザエンジニアがサイト分離のリリース
を重要なマイルストーンとして語っている理由がわかります。

## さいごに

この投稿では、ブラウザー・アーキテクチャの概要とマルチプロセス・アーキテクチャの利点につい
て説明しました。 また、マルチプロセスアーキテクチャに深く関連するChromeでのサービス化とサ
イト分離についても説明しました。次の投稿では、Webサイトを表示するために、これらのプロセ
スとスレッドの間で何が起こるのかを詳しく説明します。

この投稿は楽しめましたか？今後の投稿について質問や提案がある場合は、下記のコメント欄または
Twitterの[@kosamari](https://twitter.com/kosamari)までご連絡ください。

<a class="button button-primary gc-analytics-event attempt-right"
   href="/web/updates/2018/09/inside-browser-part2"
   data-category="InsideBrowser" data-label="Part1 / Next">次へ: ナビゲーションで何が起きるか</a>

<div class="clearfix"></div>

## フィードバック {: .hide-from-toc }

{% include "web/_shared/helpful.html" %}

<div class="clearfix"></div>

{% include "web/_shared/rss-widget-updates.html" %}

{% include "comment-widget.html" %}
