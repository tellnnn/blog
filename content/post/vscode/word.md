---
title: "VSCode x Markdown で MS Word 文書の作成"
date: "2020-08-01"
toc: true
tags: ["Visual Studio Code", "Markdown", "Pandoc"]
categories: ["Tips"]
---

普段メモやノートをとったり、レポートを書いたりするときに、みなさんは普段どのようにしていますか？

<!--more-->

メモ帳アプリや Word、グーグルドキュメントなど様々な選択肢があるかと思いますが、Markdown を使うという方は少なくないと思います。わたしもそのうちのひとりで、VSCode 上で Markdown 文書を編集する環境を整え、日々 Markdown ライフを過ごしています。

とは言っても、どこかに提出するさいに .docx ファイルを指定されることも多く、その度に渋々 Word を開いてきました。ただ、せっかく Markdown を使っているのですから、.html や .pdf といった出力先の候補に .docx も含めてしまい、文書の編集そのものは .md で完結させてしまったほうがしあわせになれそうです。

ということで、この記事では VSCode とその拡張機能を使って作成した .md ファイルを .docx ファイルに書き出すことを目指します。似たようなことはもうすでにいくつも記事[^先例1]<sup>,</sup>[^先例2]にされていて、車輪の再発明を地で行っていますが、自分なりに発見もあったので記録しておきます。

## 準備

さて、ここでは以降のために下準備をします。必要なものは、以下の3つです。

1. [VSCode](https://code.visualstudio.com/)
2. [Markdown Preview Enhanced](https://shd101wyy.github.io/markdown-preview-enhanced/#/)（VSCode の拡張機能）
3. [Pandoc](https://pandoc.org/MANUAL.html)

インストール方法については、上のリンクから公式ドキュメントに飛べるので、そちらからご確認ください。

なお、Markdown Preview Enhanced は、ほかにも ImageMagick など組み合わせることで、できることの広がるツールがいくつかあり[^外部ツール1]<sup>,</sup>[^外部ツール2]<sup>,</sup>[^外部ツール3]<sup>,</sup>[^外部ツール4]、この記事では ImageMagick が少しだけ登場します。ImageMagick のインストール方法は[公式ドキュメント](https://imagemagick.org/script/download.php) などを適宜参照してください。

## 実践 - 1

はじめに VSCode x Markdown で Word 文書を作成する流れはだいたい以下の3ステップにわけられます。

1. VSCode 上で .md ファイルを編集する
2. 編集した .md ファイルにフロントマターを追加する
3. Markdown Preview Enhanced を使って .docx ファイルを書き出す

### 1 - Markdown 文書の編集

普段どおり .md ファイルを編集します。Markdown の書き方は検索するとたくさん出てきますのでそちらをご確認ください。なお、必須ではありませんが、今回紹介する Pandoc や Markdown Preview Enhanced に特有の書き方もあるので、気になる方は公式ドキュメントもご確認ください[^mdの書き方1]<sup>,</sup>[^mdの書き方2]。

さて、VSCode 上で .md ファイルを作成したら、VSCode のコマンドパレットから 「Markdown: Markdown Preview Enhanced: Open Preview to the Side」を選択して横にプレビューを開きます（このプレビューは簡易的なもので、最終的な見た目とは異なります）。

{{< figure library="true" src="./vscode/word/command-palette.png" title="VS Code のコマンドパレット" numbered="true" >}}

### 2 - フロントマターの追加

作成した .md ファイルの先頭に **（YAML）フロントマター** を追加します。

```yaml
---
output: word_document
---
```

この `---` で挟まれた要素がフロントマターで、これだけあれば既に編集中の .md ファイルを .docx ファイルに書き出すことができます。ひとまず .docx ファイルを書き出してみたい方は[3](#3---書き出す) まで飛ばしてもらって構いません。

.docx ファイルに書き出すうえでの必須項目は `output: word_document` だけですが、以下では .docx ファイルに出力する状況に焦点をあてて、この必須項目以外に指定できる要素を一部紹介します。はじめに、紹介する項目を全て列挙すると、次のような感じなります。

```yaml
title:
author:
date:
abstract:
keywords:
subject:
description:
category:
lang:
toc-title:
output:
  word_document:
    toc:
    toc_depth:
    highlight:
    reference_doc:
    pandoc_args:
```

なお、ここでのインデントは実際にフロントマターを記述するさいにも必要なものになります。

#### 基本情報

まず多くの人がフロントマターに追加することになるであろう、文書の基本情報と呼ぶべき要素を紹介します。

title
: 文書のタイトルを指定できます。

subtitle
: 文書のサブタイトルを指定できます。

author
: 著者名を指定できます。複数名書くばあいには次のようにします。
```yaml
author:
  - 著者1
  - 著者2
```

date
: 日付を指定できます。

abstract
: アブストラクトが書けます。改行を含むためには次のようにします。ここでは「第一段落」の後に半角スペースが2つ挿入されていて、このようにすると、Word 上では段落内改行を挿入できます。一方、「第一段落」と「第二段落」の間に空白行を挿入する方法でも書くことができ、このばあい「第二段落」は新しい段落として認識されます。
```yaml
abstract: |
  第1段落  
  第2段落
```

toc_title
: 目次の見出しを指定できます。何も指定しないと「Table of Contents」になります。後述する `toc` が設定されている必要があります。

かりに、次のようにフロントマターを指定したばあいの出力は下図のようになります。
```yaml
---
title: 文書のタイトル
subtitle: 文書のサブタイトル
author: 氏名
date: 2222/02/22
abstract: |
  概要の第一段落  
  概要の第二段落
output: word_document
---
```

{{< figure library="true" src="./vscode/word/simple-example.png" title="基本情報を追加した例" numbered="true" >}}

#### Word 文書のメタデータ

次に .docx ファイルのメタデータに関連する要素を紹介します。ここで指定できる要素は、上のように .docx ファイルの内容に反映されるのではなく、.docx ファイルのプロパティに反映されます。

keywords
: 文書のキーワードを指定できます。

subject
: 文書の題材を指定できます。

description
: 文書の説明を記述できます。書き方は `abstract` と同じです。

category
: 文書のカテゴリを指定できます。

lang
: 文書の言語を指定できます。日本語を指定する「ja-JP」にすると欧文に対して和文フォントが適用されるようです。ただ、「ja-JP」を指定すると数式がイタリックになるので、数式を含むばあいは `lang` は指定しない方がよさそうです。

ただ、ここで解決できなかった点がいくつかあります。というのも、本来であればこれらの項目をフロントマターに追加したばあい、書き出された .docx ファイルのプロパティに反映されているはず[^メタデータ]なのですが、手元の Word for Mac で確認できませんでした（Word の「ファイル > プロパティ」から確認できます）。例えばフロントマターを次のように設定したとします。
```yaml
---
title: タイトル
subtitle: サブタイトル
author: 氏名
date: 日付
abstract: |
  概要　第一段落  
  概要　第二段落
keywords: 
  - キーワード 1
  - キーワード 2
subject: サブジェクト
description: |
  説明　第一段落  
  説明　第二段落
category: カテゴリ
lang: ja-JP
---
```
このとき、.docx ファイルのプロパティを確認すると次のようになります。ここで、本来であれば「サブタイトル」や「分類」、「コメント」などにそれぞれ `subtitle`、`category`、`description` などで指定した内容が反映されるはずですが、手元の環境だと反映されていないようです。

{{< figure library="true" src="./vscode/word/word-metadata.png" title=".docx ファイルのメタデータ" numbered="true" >}}

※ Word 文書のメタデータをあえて指定したい場面がどれだけあるのかは未知数で、実際に「Word メタデータ」で検索すると、むしろ消したい場面が多そうです……。

#### Output

一番最初の例では `output: word_document` として終わってしまいましたが、この項にネストさせる形でさらにいくつかの項目を指定できます。

path
: 書き出し先およびファイル名を指定することができます。指定しないばあい、例えば `example.md` を編集していれば `example.docx` が作業ディレクトリに書き出されます。

hightligt
: コードチャンクでのハイライトスタイルが選べます。選べるスタイルとその見た目は [こちら](https://www.garrickadenbuie.com/blog/pandoc-syntax-highlighting-examples/) で確認できます。デフォルトは `pygments` になっています。

toc
: 目次を生成します。デフォルトではオフになっています。

toc_depth
: 目次にどの深さの見出しまで含めるかを指定できます。最大は 6 で、デフォルトは 3 です。

reference_doc
: Word スタイルが定義された .docx ファイルへのパスを指定することで Word スタイルを出力するファイルに適用できます。これについては [後述](#Word-スタイルの指定) します。何も指定しないと Pandoc が用意しているスタイルが適用されます。Markdown Preview Enhanced のマニュアルでは `reference_docx` となっていますが、`reference_doc` が正しいようです。

pandoc_args
: そのほか Pandoc に渡せるオプションをここで指定できます。今回は `lua-filter` を追加します。詳しくは [後述](#改ページ) します。

ここで紹介した項目は、たとえば、次のように指定することができます。上で紹介した `toc_title` は `toc` が `true` になっていれば反映されます。
```yaml
toc_title: 目次の見出し
output:
  word_document:
    path: /output/output.docx
    toc: true
    toc_depth: 6
    highlight: zenburn
    reference_doc: styles.docx
    pandoc_args: [
      "--lua-filter=path-to-lua-filter"
    ]
```

#### フロントマターの例

以上を踏まえ、指定できるところそれぞれ指定していくと、以下のような感じになります。もちろん、これらを全て指定する必要はありません。
```yaml
---
title: タイトル
subtitle: サブタイトル
author: 氏名
date: 日付
abstract: |
  概要　第一段落  
  概要　第二段落
toc-title: 目次
keywords: 
  - キーワード 1
  - キーワード 2
subject: サブジェクト
description: |
  説明　第一段落  
  説明　第二段落
category: カテゴリ
lang: ja-JP
output:
  word_document:
    path: output.docx
    toc: true
    toc_depth: 6
    highlight: zenburn
    reference_doc: styles.docx
    pandoc_args: [
      "--lua-filter=path-to-lua-filter"
    ]
---
```

### 3 - 書き出す

さて、好みのフロントマターが追加できたら、書き出してみます（手元の文書で上の例を試してみるばあいは、`reference_doc` と `pandoc_args` を除いてください）。

書き出す方法は、VSCode 上でプレビューを横に開いている状態にし、プレビューの上で右クリックして「Pandoc」を選択するだけです（下図）。うまくいくと勝手に .docx ファイルが開きます。エラーがあるばあいは、右下にエラーメッセージが出てくるのでそこで確認ができます。

{{< figure library="true" src="./vscode/word/output.png" title=".docx ファイルの書き出し" numbered="true" width="200" >}}

## 実践 - 2

さて、書き出すだけなら以上で十分なのですが、Word のスタイルを適用するといった、より実用的な操作が可能なので、以下ではそれに触れていきます。

### Word スタイルの指定

上でも触れたように、 `reference_doc` に Word スタイルを定義した .docx ファイルへのパスを指定することで、 Word スタイルを出力ファイルに適用できます。この Word スタイルは完全に自由に作れるわけではなく、指定できるスタイルの要素はある程度決まっています。詳しく知りたい方は [Pandoc User's Guide](https://pandoc.org/MANUAL.html) の --reference-doc の項を参照してください。

#### Word スタイルの準備

Word スタイルをいちからつくるのは骨が折れるので、代わりに Pandoc が用意している `reference.docx` を修正する方法をとります。これには 2 通りの方法があって、それぞれ説明していきますが、個人的にはひとつめの方法をおすすめします。

-----

1 つめの方法は、ターミナルなどで
```zsh
pandoc --print-default-data-file=reference.docx > my_styles.docx
```
として、Pandoc の用意している reference.docx をコピーし、それを編集する方法です。コピーして得られたファイルでは見出しや本文などがその名前とともにすべて列挙されています。わかりにくそうなところとしては

First Paragraph
: 各セクションで最初の段落にこのスタイルが適用されます。Body Text（本文）を参照しているので、まずは Body Text（本文）を編集しておいてから、最初の段落だけに適用したい特殊なスタイル（最初の段落だけ字下げをしないとか）を First Paragraph に反映させるといいと思います。

Verbatim Char
: インラインコードなどに適用されるスタイルです。フロントマターで指定した `highlight` とは別のものです。

などがあります。適当にスタイルを編集したら .docx 形式で保存します。そして、フロントマターの `reference_doc` でそのファイルへのパスを指定すると、書き出すさいに自動的にスタイルが適用されます。

-----

2 つめの方法では、まずフロントマターの `reference_doc` を指定せずに、編集中の .md ファイルを一度 .docx ファイルに書き出します。この段階では Pandoc の用意する reference.docx が適用された .docx ファイルが書き出されます。そして、書き出した .docx ファイル上でスタイルを編集し、名前を my_styles.docx など適当なものに変えて保存します。その後改めて .md ファイルを .docx ファイルを書き出し、そのさい `reference_doc` に先程 Word スタイルを編集した .docx ファイルへのパスを指定するようにします。

#### コードチャンクのスタイル

ふたつめの方法がおすすめできない理由は、文書にコードチャンクが含まれているとうまくいかないことがあるためです。

かりに、Word スタイルを編集するために 1 回目は次のようなフロントマターで .docx ファイルを書き出したとします。
```yaml
output:
  word_document:
    highlight: tango
```
この時点で、書き出された .docx ファイルの Word スタイルは、`reference.docx` 内で指定されている Word スタイルに、 `highlight` で指定されたハイライトスタイルを Word スタイル形式で加えたものになっています。さて、コードチャンクの Word スタイルをいじらずに「見出し 1」などを適当に編集して my_styles.docx として保存し、2 回目以降は次のようなフロントマターを指定して .docx ファイルを書き出すとします。
```yaml
output:
  word_document:
    highlight: zenburn
    reference_doc: my_styles.docx
```
変更点は、`reference_doc` が追加されたことと、`highlight` を最初に指定していた `tango` とは別のものにしたことです。しかし、先述のように、`my_styles.docx` にはすでに 1 回目に `highlight` に指定したハイライトスタイル（`tagno`）が含まれています。したがって、コードチャンクのスタイルにかんしては今回指定している `zenburn` と `my_styles.docx` に含まれている `tango` が競合している状態になります。このように競合しているばあいはどうやら `reference_doc` が優先されるらしく、あとから `highlight` を変更することができなくなってしまいます。

ひとつめの方法をとるばあいはこのような競合が発生せず、いつでも `highlight` を変更できます。一方で、ふたつめの方法は、実際に自分が書いた文面を見ながら Word スタイルを調整できるため、コードチャンクが含まれない場合はこちらの方が便利かもしれません。

#### 章・節番号

章・節番号を付与したいばあいは、スタイルとして指定する .docx ファイル内で「見出し 1」や「見出し 2」などにアウトラインを定義します。やりかたについてはサポートページ[^サポートページ]などを参照してください。

なお、目次の見出しには「目次の見出し」というスタイルが適用されます。この「目次の見出し」は「見出し 1」を参照しているので、目次の見出しに章番号を振りたくないときは、「見出し 1」などにアウトラインを定義したあと、「目次の見出し」でアウトラインを「なし」に設定するとうまくいきます。

#### ヘッダー・フッター・ページ番号

ヘッダー、フッター、ページ番号も章・節番号と同じく、スタイルとして指定する .docx ファイル内でスタイルとして定義することで挿入できます。

### 改ページ

文書を書いているとページを改めたくなるときがあります。改ページを行うために、上で後述するとした、`lua-filter` を使う方法をご紹介します。

では、まず改ページを実装するのに必要な .lua ファイルを準備します。準備するファイルは lua フィルターを集めた [lua-filters](https://github.com/pandoc/lua-filters) というリポジトリに含まれる `pagebreak.lua` です。お好みの方法でお手元の環境にこのファイルを準備してください。

ファイルが準備できたらフロントマターでファイルまでのパスを指定してあげます。例えば、`/Users/USERNAME/.pandoc/pagebreak.lua` がファイルの絶対パスだとしたら
```yaml
output:
  word_document:
    pandoc_args: [
      "--lua-filter=/Users/USERNAME/.pandoc/pagebreak.lua"
    ]
```
としてあげます。

実際に改ページを挿入するときには、文書の適当なところで
```markdown
\newpage
```
としてあげるとそこでページが改められます。

### 図の挿入

結論から言えば、個人的には
```markdown
![図のキャプション](test.png)
```
のように挿入するのがよいです。

念のため、とりうる挿入方法を列挙すると、以下の 3 つがあります。

1. `![図のキャプション](test.png)` （Markdown）
2. `<img src="text.png">` （html）
3. `@import "test.png"` （Markdown Preview Enhance）

しかし、2 つめの img タグで挿入すると、どうやら .docx ファイルに書き出すさいに画像を挿入できないようです。

3 つめは Markdown Preview Enhanced でサポートされている記法で、様々な形式のファイルを挿入することができます。しかし、本来は
```markdown
@import "test.png" {width="300px" height="200px" title="my title" alt="my alt"}
```
のように図の大きさやタイトルなどを指定できるはずなのですが、`{}` の要素があると .docx ファイルに書き出すさいに画像を挿入できなくなってしまうようです……（そのほか詳しい記法については Markdown Preview Enhanced の[マニュアル](https://shd101wyy.github.io/markdown-preview-enhanced/#/file-imports)を参照してください）。

結果として、1 つめでも 3 つめでもできることに変わりがないので、環境に依存しない 1 つめの方法がよさそうというわけです。

### コードの実行と出力

Markdown Preview Enhanced の設定で項目 `enableScriptExecution` を on にすると、Jupyter Notebook や RMarkdown のようにコードチャンクに含まれるコードを実行して、その出力も文書に含めることができます。この項目は VSCode で、「Code > 基本設定 > 設定」から「設定の検索」を開き、「enableScriptExecution」を検索するとすぐに見つかります。

なお、出力に図が含まれるときには図をエクスポートするのに ImageMagick が必要になります。出力された図は assets ディレクトリに格納されます。

コードの実行については、プレビューでコードチャンクにカーソルを合わせると右上に矢印が表示されるので、そこを押すとそのチャンクが実行されます。「all」を押すと全てのチャンクが実行されます（コマンドパレットから実行することもできます）。

{{< figure library="true" src="./vscode/word/code-execution.png" title="コードチャンクの実装" numbered="true">}}

コードを実行する機能はかなり便利ですが、チャンク間で変数を共有するようなので、連続した処理を複数チャンクにわたって書くのには向かなさそうです。

## 解決しなかった点・その他

ここからは、紹介や説明というより、理解がおよんでいなかったり、もっと上手くできるのではないか……と悩んでいたりする点や、雑多な気づきに触れていきます。

### あなたの Pandoc はどこから？

唐突ですが、あなたの手元の環境に Pandoc はいくつ入っているでしょうか……。ちなみにわたしは 3 つ入っていました……。一応それぞれ挙げていくと、

1. Homebrew でインストールしたもの
```zsh
$ /usr/local/bin/pandoc -v

> pandoc 2.10.1
> Compiled with pandoc-types 1.21, texmath 0.12.0.2, skylighting 0.8.5
```
2. Anaconda についてきたもの
```zsh
$ /Users/USERNAME/anaconda3/bin/pandoc -v

> pandoc 2.2.3.2
> Compiled with pandoc-types 1.17.5.1, texmath 0.11.0.1, skylighting 0.7.2
```
3. RStudio についてきたもの
```zsh
$ /Applications/RStudio.app/Contents/MacOS/pandoc/pandoc -v

> pandoc 2.7.3
> Compiled with pandoc-types 1.17.5.4, texmath 0.11.2.2, skylighting 0.8.1
```

先述したように Markdown Preview Enhanced は Pandoc を呼び出しているので、このように複数あるばあいはどれかを指定しておいた方が無難かもしれません。指定の仕方は、VSCode で、「Code > 基本設定 > 設定」から「設定の検索」を開いて「Pandoc Path」を検索し、そこに好みの Pandoc のパスを指定してあげるだけです。

### Pandoc がユーザーデータディレクトリを参照してくれない？

上の「[改ページ](#改ページ)」でファイルの絶対パスを `/Users/username/.pandoc/pagebreak.lua` としたのは実はこれに関連しています。Pandoc User's Guide を覗いてみると `--data-dir` の項に

> Specify the user data directory to search for pandoc data files. If this option is not specified, the default user data directory will be used. On *nix and macOS systems this will be the pandoc subdirectory of the XDG data directory (by default, $HOME/.local/share, overridable by setting the XDG_DATA_HOME environment variable). If that directory does not exist, $HOME/.pandoc will be used (for backwards compatibility).

とあり、どうやら Pandoc にはユーザーデータディレクトリなるものがあるとわかります。実際、ターミナルなどで
```zsh
pandoc --v
```
としてやると、`Default user data directory` としてパスが指定されていることがわかります。

これを見て、なるほど `$HOME/.pandoc` に参照したい .lua ファイル、あるいは style.docx なんかを置いて相対パスを指定すればよいのかと考えたのですが、どうもそうはいかないようで、ファイルが見つからないとエラーが返ってきてしまいました。今のところは絶対パスを指定するか、.md ファイルと同じディレクトリにファイルを置くことで済ませています。

### 見出しの深さが謎

`toc-depth` オプションはどうやら 1~6 の範囲しかうけつけないようなのですが、Word のスタイルは深さ 9 まで指定することができるようになっています。一方で、markdown で深さ 8 以上の見出しを単純に「#」を並べて書こうとすると上手く認識してくれません……。結果として、`toc-option`、Word のスタイル、Markdown で見出しの深さの最大値が異なっている状況にあります。

ただ、実のところそんなに深い見出しをつくることはないので、これについてはさして問題ではないとも言えます。

### 数式をもっときれいに変換できないのか……?

Markdown で書かれた数式は、Word の数式に自動的に変換されます。しかし、積分記号が単なる記号として入ってしまって大きさが微妙になるなど、完璧に変換されるわけではありません。

例えば下図の1行目は、 Pandoc を介したもので、2行目は同じ数式を Word の数式ツールでポチポチ挿入したものですが、前者は積分記号の大きさがイケてないのがわかると思います。

{{< figure library="true" src="./vscode/word/math.png" title="Word の数式" numbered="true" width="200" >}}

ただ、個人的には Word でそんなに数式を書くこともないので、致命的というわけでもありません。

### 著者の所属やメールアドレスを挿入したい

自分の所属やメールアドレスを氏名に併記することはときおりあって、これをぜひいい感じに処理したいものです。そこで上で説明した [lua-filters](https://github.com/pandoc/lua-filters) というリポジトリにある `scholarly-metadata` と `author-info-blocks` を紹介します。使い方としては、[改ページ](#改ページ) で使った `pagebreak` と同じく `--lua-filter` オプションでパスを指定してあげるだけです（今回は `pagebreak` でつかった `\newpage` のような記述も不要です）。
```yaml
output:
  word_document:
    pandoc_args: [
      "--lua-filter=[scholarly-metadataへのパス]",
      "--lua-filter=[author-info-blocksへのパス]"
    ]
```
ここで注意が必要なのは、後者が前者を参照している関係上、必ずこの順で指定しなければならない点です。

さて、ここで悩ましいのは、目次やアブストラクトを入れていると、著者の氏名が表示されるブロックと、著者の所属やメールアドレスが表示されるブロックとが、目次やアブストラクトで分断されてしまうことです（下図）。

{{< figure library="true" src="./vscode/word/author-affiliation.png" title="著者の所属・メールアドレス" numbered="true" >}}

色々調べて見たのですが、これよりもよい方法はなかなかなさそうです……。

## Enjoy!

最期に、私が適当につくった .md ファイルを置いておきます。

<pre style="max-height: 300px;">
<code>
---
title: タイトル
subtitle: サブタイトル
author: 氏名
date: 日付
abstract: |
  概要　第一段落  
  概要　第二段落
toc-title: 目次
keywords: 
  - キーワード 1
  - キーワード 2
subject: サブジェクト
description: |
  説明　第一段落  
  説明　第二段落
category: カテゴリ
output:
  word_document:
    toc: true
    toc_depth: 6
    highlight: zenburn
    pandoc_args: [
      "--lua-filter=/Users/teruaki/.pandoc/lua-filters-master/pagebreak/pagebreak.lua"
    ]
---

\newpage

$$
\int \, \frac{1}{\sqrt{x}} \, dx
$$

\newpage

# Heading 1

## Heading 2

### Heading 3

#### Heading 4

##### Heading 5

###### Heading 6

\newpage

# 見出し 1

## 見出し 2

### 見出し 3

#### 見出し 4

##### 見出し 5

###### 見出し 6

\newpage

# 詳細

## 本文 - 1

これは第一段落です。

これは第二段落です。

## 本文 - 2

これは第一段落です。

これは第二段落です。

## 本文 - 3

本文。_斜体_。**太字。**

## 数式

$$
f(x) = \frac{1}{\sqrt{2\pi\sigma^2}} \exp{\left(-\frac{(x-\mu)^2}{2\sigma^2}\right)} \quad (x \in \mathbb{R})
$$

\newpage

## リスト

### Disc 型リスト

- リスト 1
  - あいうえお
- リスト 2
  - かきくけこ

### Decimal 型リスト

1. リスト 1
   1. あいうえお
2. リスト 2
   1. かきくけこ

### Definition 型リスト

項目 1
: 定義 1

項目 2
: 定義 2

## 引用

> 吾輩は猫である。名前はまだ無い。

\newpage

## インラインコード・コードチャンク

これは `inline code`です。

これは実行可能なコードチャンクで、実行結果を出力に含めることができます。

```python {cmd=true matplotlib=true .line-numbers}
import matplotlib.pyplot as plt
plt.plot([1,2,3,4])
plt.show() # show figure
```

\newpage

## 表

Table: **表1.** これは表1です。

| Left align | Right align | Center align |
| :--------- | ----------: | :----------: |
| This       |        This |     This     |
| column     |      column |    column    |
| will       |        will |     will     |
| be         |          be |      be      |
| left       |       right |    center    |
| aligned    |     aligned |   aligned    |

## 図

![**図1.** これは図1です。](https://cdn.pixabay.com/photo/2016/04/25/18/07/halcyon-1352522_1280.jpg)

</code>
</pre>

## まとめ

実はこれが初めて Pandoc にふれる機会だったのですが、その多機能さに感動しました。例えば .docx ファイルへの出力だけでなく、LuaLaTex を介した日本語 PDF への出力も可能です。

また実のところ、今回紹介した Markdown Preview Enhanced は、基本的には Pandoc のコマンドをうまい具合に実行してくれているだけなので、この記事で触れたことは Pandoc さえあれば実現できると思います。詳しく知りたい方は、ぜひ [Pandoc User's Guide](https://pandoc.org/MANUAL.html) を参照してください。

[^先例1]: [R Markdownで直接Wordドキュメント生成](https://qiita.com/kazutan/items/7e19323dae46e523adc5)
[^先例2]: [Visual Studio Codeでソフトウェア設計書(Word文書)を書く環境を作る(環境構築2)](https://yun-craft.com/software-crafts/vscode/03-3)
[^外部ツール1]: [(メモ) ドキュメント楽に作りたいのでMarkdown Preview Enhancedを使い始めた](https://niszet.hatenablog.com/entry/2018/01/11/214410)
[^外部ツール2]: [Visual Studio Code + Markdown Preview Enhancedはチームでデファクト化したいMarkdown環境だ！、と思う.(2017/10月時点)](https://qiita.com/kitfactory/items/2fde799fa092f0d8f0f1)
[^外部ツール3]: [Atom Markdown Preview Enhancedで業務に関する全てのドキュメントを書くためのTips](https://qiita.com/cha2maru/items/9ba4413bd91083e7c984)
[^外部ツール4]: [【ドキュメントが書きたくなる】Markdownライブプレビュー + インライン数式/UML/図表 + 綺麗にPDF/Wordエクスポートまで](https://qiita.com/tomo_makes/items/da4e8fe7d8cf168b545f)
[^mdの書き方1]: Markdown Preview Enhanced のうち [Markdown Basics](https://shd101wyy.github.io/markdown-preview-enhanced/#/markdown-basics) の節
[^mdの書き方2]: [Pandoc User's Guide](https://pandoc.org/MANUAL.html) のうち Pandoc’s Markdown の節
[^メタデータ]: [(Pandoc) Wordのカスタムプロパティを使う](https://niszet.hatenablog.com/entry/2019/12/07/000000)
[^サポートページ]: [見出しに番号を付ける](https://support.microsoft.com/ja-jp/office/%E8%A6%8B%E5%87%BA%E3%81%97%E3%81%AB%E7%95%AA%E5%8F%B7%E3%82%92%E4%BB%98%E3%81%91%E3%82%8B-ce24e028-4cb4-4d4a-bf25-fb2c61fc6585)