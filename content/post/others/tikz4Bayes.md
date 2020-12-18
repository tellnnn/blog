---
title: TikZ でグラフィカルモデルを書こう！
date: "2020-12-18"
toc: true
tags: ["Bayes","LaTeX","TikZ"]
categories: ["Tips"]
---

この記事は[ベイズ塾 Advent Calendar 2020](https://adventar.org/calendars/5083) の 18 日目の記事です。

<!--more-->

## グラフィカルモデル（プレート表現）

ベイジアンモデリングにおいて、モデルが複雑であるばあい、グラフィカルモデル（プレート表現）がその理解の補助的な役割を果たします（といいつつ、あんまり見かけない……人気ないのかな……）。

グラフィカルモデル（プレート表現）の記法はひとつでなく、少なくとも Lee & Wagenmakers. (2013). *Bayesian Cognitive Modeling*.（通称：怖い人本）で紹介されているものや、Kruschke. (2015). *Doing Bayesian data analysis*.（通称：犬4匹本）[^kruschke]で使われているものなどがあります。

[^kruschke]: この記事では触れませんが、犬4匹本スタイルでグラフィカルモデル（プレート表現）を書くためのツールもあります。 https://github.com/tinu-schneider/DBDA_hierach_diagram

個人的に前者のものの方がシンプルで好みなので、この記事は前者の記法を採用します。その記法は

{{< figure library="true" src="./content/post/others/Lee&Wagenmakers_style-figure0.png" title="原著 p.18 / 邦訳 p.16" numbered="true" width="300" >}}

という3つの要素からなり、さらにノードについて

{{< figure library="true" src="./content/post/others/Lee&Wagenmakers_style-figure1.png" title="原著 p.18 / 邦訳 p.16" numbered="true" width="400" >}}

1. 離散値（四角形）– 連続値（円）
2. 確率変数（一重）– 確定変数（二重）
3. 観測変数（影つき）– 非観測変数（影なし）

という3次元の組み合わせとして変数を表現します。

たとえばこんな感じ。

{{< figure library="true" src="./content/post/others/Lee&Wagenmakers_Fig10_4.png" title="Lee & Wagenmakers (2013) Fig.10.4 p.149" numbered="true" >}}

この図が例として最適なので、以降ではこの図を書くことを最終目標として話を進めていきます。

## 何で書くか

記法がわかったから早速「グラフィカルモデル（プレート表現）を書こう！」ということになるのですが、何で書くかということが問題になります。ぱっと思いつくだけでも

1. Office, Adobe, その他類似のツール
2. Graphviz + DOT
3. LaTeX + TikZ

などがあります。どういう訳か選択肢1は眼中にないので、選択肢2か選択肢3をとることにしましょう。

### Graphviz + DOT

[Graphviz](https://graphviz.org/) のうち、[DOT言語](https://graphviz.org/doc/info/lang.html) を使って書くやり方です。ここでは深入りしないのですが、R 使いのベイジアンに朗報なのは、R の [DiagrammR](https://rich-iannone.github.io/DiagrammeR/#page-top) というライブラリから DOT 言語を使えることです[^pv]。Python 使いのベイジアンにも [Graphviz](https://graphviz.readthedocs.io/en/stable/index.html) というパッケージがあります。

[^pv]: 余談ですが、公式ページのトップにある、イカす音楽の流れる PV がなんかすきです

DiagrammR を使ってグラフィカルモデル（プレート表現）を書く方法は、すでに他の方が記事にしているのでそちらを参照してください。

- [Rでグラフィカルモデルを書こう！](https://kunisatolab.github.io/main/how-to-graphicalModel.html)（この記事を書くきっかけになった記事）
- [DiagrammeRと仲良くなった話ーグラフィカルモデルのためのDiagrammeR速習ー](https://www.slideshare.net/TakashiYamane1/diagrammerdiagrammer)

ただ、2つめの資料で言及されているのですが、Graphviz + DOT を使うやり方は、[図3]({{< ref "tikz4Bayes#figure-lee--wagenmakers-2013-fig104-p149" >}}) にあるような部分的に重なったプレートを書くことができないようです（つらい）。

### LaTeX + TikZ

LaTeX のパッケージ TikZ を使って書くやり方です。本命です。

先に言っておくと、TikZ を使えば部分的に重なるプレートも書くことができます。

さらに、TikZ のライブラリに [BayesNet](https://github.com/jluttine/tikz-bayesnet) といういかにもなライブラリがあり、このライブラリに 99% のベイジアンが「満足」と答えています（当社調べ）。例えば下図の左のような図を書くことができます（ちなみに右の図は directed factor graph と呼ばれているようです）。

{{< figure src="https://camo.githubusercontent.com/a3214f7f1624a86e40b71d102c3152a0d2dff3612231135b35d65b65ead2a28a/687474703a2f2f692e696d6775722e636f6d2f437a4e796b2e706e67" title="GitHubリポジトリの例より" numbered="true" width="500" >}}

ただ、このライブラリ、プレートのキャプションを右下以外に配置することができません。これを看過できない人がいます。そう、残りの 1% のベイジアンです（実際に issue も立っていますが、数年前から開発が止まっています）。

では、仕方がないので BayesNet を解読してピュアな TikZ を使って書くことにしましょう。幸い、キャプションを右下以外に配置するにはオプションひとついじればよく、BayesNet 全体がたった137行のコードから構成されているので、

1. 部分的に重なるプレートを書きたい
2. プレートのキャプションを好きなところに配置したい

という2つの欲求をかんたんに満たすことができます。

## TikZ でグラフィカルモデルを書こう！

### 準備

{{< highlight latex >}}
\documentclass{standalone}

\usepackage{tikz}
\usetikzlibrary{positioning,fit,arrows.meta}

\begin{document}
    \begin{tikzpicture}
        % nodes ...
        % edges ...
        % plates ...
    \end{tikzpicture}
\end{document}
{{< / highlight >}}

`\usetikzlibrary{positionig,fit,arrows.meta}` としていますが、`positioning` はノードの相対配置をするため、`fit` はプレートを書くため、`arrows.meta` はエッジの矢印を整えるために使っています。長くなってしまうので深追いはしませんが、詳しい使い方やその他の TikZ ライブラリについては[こちら](https://tex.stackexchange.com/questions/42611/list-of-available-tikz-libraries-with-a-short-introduction) が大変参考になります（とくに `arrows.meta` については[こちら](https://tex.stackexchange.com/questions/5461/is-it-possible-to-change-the-size-of-an-arrowhead-in-tikz-pgf)、`positioning` については[こちら](https://zrbabbler.hatenablog.com/entry/20150201/1422745097) ）。

以下でも軽く説明しますが、TikZ の基本的な書き方は Overleaf のドキュメントが役に立ちます。

- [TikZ package - Overleaf, Online LaTeX Editor](https://www.overleaf.com/learn/latex/TikZ_package)
- [A Tutorial for Beginners (Part 1)—Basic Drawing - Overleaf, Online LaTeX Editor](https://www.overleaf.com/learn/latex/LaTeX_Graphics_using_TikZ:_A_Tutorial_for_Beginners_(Part_1)%E2%80%94Basic_Drawing)

### ノード

ノードの基本的な書き方は以下のとおりです。「オプション」でノードの見た目や位置を、「ラベル」でノードに付される $\theta$ や $y$ といったラベルを指定できます。「ノード名」はノードの相対配置をしたり、エッジを指定したり、プレートを書いたりするさいに使います。

{{< highlight latex >}}
\node[オプション] (ノード名) {ラベル};
{{< / highlight >}}

では、[図3]({{< ref "tikz4Bayes#figure-lee--wagenmakers-2013-fig104-p149" >}}) にあるノードを書いてみます。

{{< highlight latex >}}
% nodes
\node[draw,fill=gray!60,rectangle,minimum size=1cm]                                           (n)        {$n$};
\node[draw,fill=gray!60,rectangle,minimum size=1cm,above=1cm of n]                            (k_ij)     {$k_{ij}$};
\node[draw,circle,double,minimum size=1cm,above=1cm of k_ij]                                  (theta_ij) {$\theta_{ij}$};
\node[draw,fill=gray!60,circle,minimum size=1cm,left=1cm of theta_ij]                         (t_j)      {$t_{j}$};
\node[draw,circle,minimum size=1cm,above left =1.5cm and 0.1cm of theta_ij.north,anchor=east] (alpha_i)  {$\alpha_{i}$};
\node[draw,circle,minimum size=1cm,above right=1.5cm and 0.1cm of theta_ij.north,anchor=west] (beta_i)   {$\beta_{i}$};
{{< / highlight >}}

ここで登場するオプションについてはざっと以下の通りです。

|オプション|概要|
|:--|:--|
|`draw`|ノードを枠線を書くか（ないと枠線が書かれない）|
|`fill`|ノードの塗りつぶし<br>`色!パーセンテージ` のようにすると色の濃淡を調整できます<br>デフォルトは透明です<br>ここでは観測変数のために `gray!60` を指定しています|
|`circle`, `rectangle`|ノードの形<br>`shapes` という TikZ ライブラリで拡張することができます|
|`double`|枠線を二重にするか（ないと枠線が一重になる）|
|`minimum size`|ノードの最小サイズの指定<br>すべてのノードの大きさを均一にするために使っています|
|`相対位置=(距離) of ノード名`|`positioning` ライブラリで導入されるノードの相対配置<br>相対位置は `above, below, left, right` のいずれか、またはその組み合わせが使えます<br>距離のデフォルトは1cmです<br>基準点として単なるノード名以外に `ノード名.south east` のような指定が可能です|

上のノードを追加するとこのような出力が得られます。

{{< figure library="true" src="./content/post/others/Lee&Wagenmakers_Fig10_4_nodes.png" title="ノード" numbered="true" width="200" >}}

### エッジ

ノードの基本的な書き方は以下のとおりです。「オプション」でエッジの見た目を、「fromノード名」と「toノード名」でエッジの端点を指定します。

{{< highlight latex >}}
\draw[オプション] (fromノード名) edge (toノード名);
{{< / highlight >}}

では、[図3]({{< ref "tikz4Bayes#figure-lee--wagenmakers-2013-fig104-p149" >}}) にあるエッジを書いてみます。

{{< highlight latex >}}
% edges
\draw[-Stealth]
    (n)        edge (k_ij)
    (theta_ij) edge (k_ij)
    (t_j)      edge (theta_ij)
    (alpha_i)  edge (theta_ij)
    (beta_i)   edge (theta_ij);
{{< / highlight >}}

ここで登場する `-Stealth` は一方向の矢印で矢じりに `Stealth` という形を指定しています（`Stealth` は `arrows.meta` から来ています）。もちろんこの他にもたくさんのオプションがありますが、ここでは触れません。また、矢印スタイルを共有するエッジは上のようにまとめて書くことができます。

上のエッジを追加するとこのような出力が得られます。

{{< figure library="true" src="./content/post/others/Lee&Wagenmakers_Fig10_4_edges.png" title="ノード & エッジ" numbered="true" width="200" >}}

### プレート

さいごにプレートの書き方は以下のとおりです。BayesNet ライブラリを参考に3つのノードの組み合わせでプレートを書きます。

{{< highlight latex >}}
\node[inner sep=0cm,fit={(ノード名1)(ノード名2)...}] (wrap) {};
\node[inner sep=0cm,相対位置=(距離) of wrap.位置] (caption) {キャプション};
\node[draw,rounded corners,fit={(wrap)(caption)}] (plate) {};
{{< / highlight >}}

それぞれの役割については以下のとおりです（ノード名は便宜上のもので、自由に変更できます）。

|ノード|概要|
|:--|:--|
|`wrap` ノード| `fit={}` でプレートに含まれるべきノードをすべて列挙<br>`inner sep=0cm` でノード内の要素と枠線との距離を指定|
|`caption` ノード|上述のノードの相対配置を使い、`wrap` ノードに対する相対位置で配置<br>BayesNet ではここが `below left=5pt and 0pt of wrap.south east` で固定されています<br>ノードのラベルの位置にキャプションを指定します|
|`plate` ノード|プレートの輪郭<br>`fit={(wrap)(caption)}` として上の2つを包含するように指定します<br>`rounded corners` で角を丸めています|

では、[図3]({{< ref "tikz4Bayes#figure-lee--wagenmakers-2013-fig104-p149" >}}) にあるプレートを書いてみます。

{{< highlight latex >}}
% plates
% i-plate
\node[inner sep=0cm,fit={(alpha_i)(beta_i)(theta_ij)(k_ij)}] (wrap_i) {};
\node[inner sep=0cm,above=0.3cm of wrap_i.north west,anchor=west] (caption_i) {$i\ \textrm{people}$};
\node[draw,rounded corners,fit={(wrap_i)(caption_i)}] (plate_i) {};
% j-plate
\node[below right=0.05cm and 0.5cm of plate_i.south east] (dummy_j) {};
\node[inner sep=0cm,fit={(dummy_j)(t_j)(theta_ij)(k_ij)}] (wrap_j) {};
\node[inner sep=0cm,above=0.3cm of wrap_j.south west,anchor=west] (caption_j) {$j\ \textrm{times}$};
\node[draw,rounded corners,fit={(wrap_j)(caption_j)}] (plate_j) {};
{{< / highlight >}}

`j-plate` では、見た目をよりきれいにするために `dummy_j` という透明のノードを追加しています。このノードを加えることでどのように見た目が変わるかは、このノードをコメントアウトすることで確認できます（`wrap_j` 内からも除外してください）。

上のプレートを追加するとこのような出力が得られます。[図3]({{< ref "tikz4Bayes#figure-lee--wagenmakers-2013-fig104-p149" >}}) と見比べてみてください。

{{< figure library="true" src="./content/post/others/Lee&Wagenmakers_Fig10_4_plates.png" title="ノード, エッジ, & プレート" numbered="true" width="300" >}}

### tikzset

さて、以上で目的としていた図を出力できたのですが、ノードやプレートを書くのに似たようなオプションが反復されており、無駄なだけでなく可読性が下がってしまっています。そこで、ライブラリ化するまではいかずとも、一時的なスタイルを定義してちょっとだけ体系化してみましょう。

スタイルの定義は `\tikzset` を使います。プリアンブルで

{{< highlight latex >}}
\tikzset{スタイル名/.style={各種オプションの設定}}
{{< / highlight >}}

とすれば、文書内で共通のスタイルを使用することができます。では、繰り返し使いそうなスタイルをプリアンブルで定義してみます。

{{< highlight latex >}}
\tikzset{
  disc/.style={rectangle,minimum size=1cm}, % 離散値
  cont/.style={circle,minimum size=1cm},    % 連続値
  obs/.style={draw,fill=gray!60},           % 観測変数
  latent/.style={draw},                     % 非観測変数
  wrap/.style={inner sep=0cm},              % wrap ノード
  caption/.style={inner sep=0cm},           % caption ノード
  plate/.style={draw,rounded corners}       % plate ノード
}
{{< / highlight >}}

あとは個別にオプションを指定する代わりに、ここで定義したスタイルを指定してやれば各スタイルに含まれるオプションが反映されます。このスタイルも使ってこれまでをまとめると以下のような感じになります（折りたたんであります）。

<details>
    <summary>クリックして展開</summary>
    <script src="https://gist.github.com/tellnnn/81d27c82e444489d4ef6239fe58e725d.js"></script>
</details>

### エディター

R 使いのベイジアンにぜひともおすすめしたいのが RStudio です。ご存知のとおり RStudio 上で .tex ファイルをコンパイルできるだけでなく、今度リリースされる RStudio v1.4 では複数の source pane を横に並べることができるようになるそうです（[こんな感じ](https://blog.rstudio.com/2020/10/21/rstudio-1-4-preview-multiple-source-columns/) ）。つまり、.stan ファイルでモデルを定義しながら、.tex ファイルでそのモデルを描画していくなんてこともできてしまいます。

また、万人におすすめできるのは Overleaf です。なにより面倒な環境構築をしなくてよいのが魅力です。

## さいごに

この記事は「グラフィカルモデルが書きたい > DiagrammR は少し自由度が低い > BayesNet でキャプションの位置を変えられないのが気に食わない（些細）」という動機から生えたものです。先述のとおり、我ながら BayesNet でもよいのでは？という気がしますが、個人的にはこれをもとに LaTeX も TikZ も初めて触って学ぶことができ、結構たのしかったです（なのでお作法的にまずいところがあるかもしれません。もし見つけたらご連絡ください）。

また、ここで紹介した TikZ のオプションやライブラリはほんの一部なので、気になった方は色々調べてみてください。

この記事はこれで終わりですが、[図3]({{< ref "tikz4Bayes#figure-lee--wagenmakers-2013-fig104-p149" >}}) のようにグラフィカルモデル（プレート表現）と数式を併置して一枚の図にすることもでき、こちらについては、余力があるときにどこかにまとめるかもしれません。