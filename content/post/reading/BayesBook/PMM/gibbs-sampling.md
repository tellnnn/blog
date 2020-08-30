---
title: ポアソン混合モデルにおける推論 - 2. ギブスサンプリング
date: "2020-08-31"
toc: true
tags: ["Bayes", "Machine Learning","C++","R"]
categories: ["Book"]
draft: true
---

この記事では『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』のうち、「4.3 ポアソン混合モデルにおける推論」でのギブスサンプリングを実装します。

<!--more-->

## はじめに

この記事は、いわゆる須山ベイズ本の「4.3 ポアソン混合モデルにおける推論」にかんする一連記事のうちのひとつです。この節での実装は

PMM.cpp
: ギブスサンプリングなど推論の実装

PMM.R
: 推論の実行やその推論結果の可視化などの実装

の 2 つのファイルにまとめており [GitHub のリポジトリ](https://github.com/tellnnn/BayesBook_mine) で公開しています。

各記事では、これらのファイルの該当箇所を順に説明していくかたちをとるので、関心のある方は適宜参照してください。なお、この記事は書籍を傍らに置きながら読まれることを想定しております。

付されている行番号は参照しているファイルと一致するように気をつけています（ずれていたらすみません）。さいごに、C++ には不慣れなので、改善点等ありましたらご指摘いただけると幸いです。

## 実装 (PMM.cpp)

### 準備

まずは一連の実装で必要となるインクルードまわりです。ここは前回の記事と同じですので、折りたたんでいます。

<details><summary>ひらく</summary>
{{< highlight cpp "linenos=table,linenostart=1" >}}
#include <stan/math.hpp>
#include <random>
#include <string>
#include <vector>
#include <iostream>
#include <fstream>
#include <sstream>

using namespace stan;
using namespace math;
using namespace Eigen;
{{< / highlight >}}
</details>

## 実行 (PMM.R)

