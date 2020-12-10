---
title: ポアソン混合モデルにおける推論 - 1. デモデータの生成
subtitle: 読書記録『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』
date: "2020-08-31"
toc: true
tags: ["Bayes","Machine Learning","C++","R"]
categories: ["Book"]
---

この記事では[読書記録『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』](../../toc)のうち、「4.3 ポアソン混合モデルにおける推論」で使用するデモデータの生成を実装します。

<!--more-->

## はじめに

この記事は、いわゆる須山ベイズ本「4.3 ポアソン混合モデルにおける推論」にかんする一連の記事のひとつです。この節での実装は

PMM.cpp
: ギブスサンプリングなど推論の実装

PMM.R
: 推論の実行やその推論結果の可視化などの実装

の 2 つのファイルにまとめており [GitHub のリポジトリ](https://github.com/tellnnn/BayesBook_mine) で公開しています。

各記事では、これらのファイルの該当箇所を順に説明していくかたちをとるので、関心のある方は適宜参照してください。数式やその説明をどこまで記載してよいかわからなかったので、この記事は書籍を傍らに置きながら読まれることを想定しております。

なお、改善点等ありましたらご指摘いただけると幸いです。

## 実装 （PMM.cpp）

### 準備

まずは一連の実装で必要となるライブラリを読み込みます。

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

### デモデータ生成関数

さっそく、デモデータを生成するための関数 `generate_data` を定義します。今回はポアソン混合モデルにしたがってデータを生成するため、入力として以下のものを考えます。

|入力|型|概要|
|:--|:--|:--|
|`N`|整数|サンプルサイズ|
|`K`|整数|クラスター数|
|`lambda`|長さ `K` のベクトル|各ポアソン分布のパラメータ|
|`pi`|長さ `K` のベクトル（総和が 1）|混合比率|
|`seed`|整数|乱数生成のためのシード値|

まずは、`generate_data` 関数全体をお見せします。

{{< highlight cpp "linenos=table,linenostart=13" >}}
void generate_data(int N, int K, VectorXd lambda, VectorXd pi, int seed) {
    // function to generate random data
    // inputs:
    //   N: the number of data points
    //   K: the number of clusters
    //   lambda: the rate parameter in poisson distribution
    //   pi: the mixing parameter
    //   seed: the random seed value

    // set random engine with the random seed value
    std::default_random_engine engine(seed);

    // set variables
    int s; // the latent variable
    int X; // the data
    
    // set the output file
    std::ofstream data("data.csv");
    // set the header in the file
    data << "s,X" << std::endl;
    for (int n = 0; n < N; n++) {
        // sample s and X
        s = categorical_rng(pi,engine);
        X = poisson_rng(lambda(s-1),engine);
        // output s and X
        data << s << "," << X << std::endl;
    }
}
{{< / highlight >}}

ほしい変数をサンプリングしつつ `data.csv` にその結果を書き出していきます。

1 では、サンプリングしたい潜在変数 $s_{n}$ とデータ $X$ をそれぞれ `s` および `X` として宣言しています。

3 の `for` 文の中では、$x_{n}$ をサンプリングすることを考え、まずは混合比率 `pi` をパラメータにもつカテゴリ分布から潜在変数 `s` をサンプリングします。このとき、書籍では潜在変数が 1 of K (one-hot) 表現になっていますが、`s` は $1 \ldots K$ の値をとります。

つぎに、得られた `s` をもとに、$\lambda_{k}$ すなわち `lambda(s-1)` をパラメータにもつポアソン分布からデータ `X` をサンプリングします。

さいごに、サンプリングした `s` や `X` を `data.csv` に順に書き出したら完了です。

### main 関数

さて、つぎに PMM.cpp をコンパイルしたあと、実行するさいに `N` や `lambda` といった入力を渡しつつ `generate_data` 関数を実行できるよう、`main` 関数を定義し、入力はコマンドライン引数として渡すことにします。

まず、該当箇所をお見せします。

{{< highlight cpp "linenos=table,linenostart=370" >}}
int main(int argc, char *argv[]) {
    
    // get inputs 1 ~ 4
    std::string method = argv[1];
    int N = atoi(argv[2]);
    int K = atoi(argv[3]);
    int seed = atoi(argv[4]);

    if (method == "data") {
        
        // get parameters
        VectorXd lambda; // the rate parameter in poisson distribution
        VectorXd pi; // the mixing parameter
        lambda = VectorXd::Zero(K); // initialize with zeros
        pi = VectorXd::Zero(K); // initialize with zeros
        
        for (int k = 0; k < K; k++) lambda(k) = atof(argv[5+k]);
        for (int k = 0; k < K; k++)     pi(k) = atof(argv[5+K+k]);
        
        std::cout << "Random Data Generation" << std::endl;
        generate_data(N, K, lambda, pi, seed);
{{< / highlight >}}

ここで、コマンドライン引数の 1 番目として `method` を受け取っていますが、これはこのプログラムを実行するさいに、`method` として `"data"` を渡せばデモデータを生成し、`"GS"` を渡せばギブサンプリングを、`"VI"` を渡せば変分推論を、`"CGS"` を渡せば崩壊型ギブスンサンプリングを実行するように意図したものです。

それ以外のコマンドライン引数は、冒頭で説明したとおり、`N`、`K`、`seed`、長さ `K` のベクトルとして `lambda` および `pi` です。

## 実行 (PMM.R)

では、PMM.cpp をコンパイルして実行し、生成されたデモデータが確認できるように、PMM.R を実装します。

### 準備

ワーキングディレクトリの設定やライブラリのインポートなどは以下のとおりです。なお、サンプリング結果の可視化のため `make_plot` 関数を定義しています。

{{< highlight r "linenos=table,linenostart=1" >}}
# setwd("./PoissonMixtureModel/")


# library ------------------------------
library(tidyverse)
library(colorspace)
library(patchwork)


# functions ------------------------------
make_plot <- function(method, K, s, X, bins = 30) {
  title <- str_c("Poisson Mixture Model (",method,")")
  
  tibble(s = s, X = X) %>% 
    ggplot(aes(x = X, fill = factor(s))) +
    geom_histogram(bins = bins, alpha = 0.6, position = "identity") +
    scale_fill_discrete_sequential(palette = "Viridis", labels = LETTERS[1:K]) +
    labs(title = title, fill = "cluster") +
    theme(plot.title = element_text(hjust = 0.5),
          legend.position = "bottom")
}
{{< / highlight >}}

### コンパイル

PMM.cpp をコンパイルします。いちいちシェルを開かなくて済むように `system` 関数を使用します。

{{< highlight r "linenos=table,linenostart=24" >}}
# compile c++ file ------------------------------
stan_math_standalone <- "$HOME/.cmdstanr/cmdstan-2.24.0/stan/lib/stan_math/make/standalone"
str_c("make -j4 -s -f", stan_math_standalone, "math-libs", sep = " ") %>% system()
str_c("make -j4 -s -f", stan_math_standalone, "PMM",       sep = " ") %>% system()
{{< / highlight >}}

まずは、`stan_math/make/standalone` へのパスを `stan_math_standalone` として格納しておきます。これはお手元の環境次第だと思いますが、私は CmdStanR のものを使ったので、上のようになりました。

26 行目はどうやらセッション中に 1 度実行すればよいらしく、27 行目で PMM.cpp をコンパイルしています。

### 実行

ではコンパイルして得られた実行ファイルを実行します。今回は、サンプルサイズが 1000、クラスター数が 2、ポアソン分布のパラメータがそれぞれ 44、77、混合比率が 0.5、0.5 であるようなデモデータを生成してみます。

コマンドライン引数が用意できたら、それらを渡しつつ `system` 関数を使って実行します。

{{< highlight r "linenos=table,linenostart=30" >}}
# generate data ------------------------------
method <- "data"
N <- 1000
K <- 2
gen_seed <- 6

lambda <- c(44,  77)
pi     <- c(0.5, 0.5)

str_c("./PMM", method, N, K, gen_seed, 
      str_c(lambda, collapse = " "), 
      str_c(pi, collapse = " "),
      sep = " ") %>% 
  system()

# read csv
demo_data <- read_csv(file = "data.csv", col_names = TRUE, col_types = "ii")

# plot
demo_data_plot <- 
  make_plot(
    method = method,
    K = K,
    s = demo_data$s, 
    X = demo_data$X, 
    bins = 30
  )

ggsave(filename = "demo_data.png", plot = demo_data_plot, width = 100, height = 75, units = "mm")
{{< / highlight >}}

39 行目ですが、今回の設定だと `str_c` の結果は
{{< highlight zsh >}}
./PMM data 1000 2 6 44 77 0.5 0.5
{{< / highlight >}}
となっています。

さて、実行すると結果が格納された `data.csv` が生成されるので、それを読み込み、可視化すると次のようなデモデータが確認できます。

{{< figure library="true" src="./content/post/reading/BayesBook/PMM/demo_data.png" title="デモデータ" numbered="true" >}}

図は載せませんが、クラスター数を増やしたり、サンプルサイズを増やしたりすることもできます。

というわけで、今回はデモデータを生成する関数を実装しました。以降の記事ではこの関数を使って生成されたデモデータに対して推論をします。

## おまけ実装 (PMM.cpp)

このままだと少し短いので、以降の記事で使用する PMM.cpp 内の関数の定義をここで紹介してしまいます。

### split 関数

ひとつは、ギブスサンプリングや変分推論などを実行するさいに、生成した `data.csv` を読み込むためのものです。これはほかの方の記事[^csv]からお借りしてきたものです。

{{< highlight cpp "linenos=table,linenostart=42" >}}
std::vector<std::string> split(std::string& input, char delimiter) {
    // function to get a line and split it
    // based on https://cvtech.cc/readcsv/
    std::istringstream stream(input);
    std::string field;
    std::vector<std::string> result;
    while (getline(stream, field, delimiter)) {
        result.push_back(field);
    }
    return result;
}
{{< / highlight >}}

### calc_ELBO 関数

もうひとつは、最後にギブスサンプリング、変分推論、崩壊型ギブスサンプリングを比較するさいに指標として用いられる ELBO を計算するためのものです。導出については、書籍の Appendix に記載があります。

{{< highlight cpp "linenos=table,linenostart=54" >}}
double calc_ELBO(int N, int K, VectorXi X,
                 VectorXd a_pri, VectorXd b_pri, VectorXd alpha_pri,
                 VectorXd a_pos, VectorXd b_pos, VectorXd alpha_pos) {
    // function to calculate ELBO
    // inputs:
    //   N: the number of data points
    //   K: the number of clusters
    //   X: the data
    //   a_pri: the shape parameter before the update
    //   b_pri: the rate parameter before the update
    //   alpha_pri: the concentration parameter before the update
    //   a_pos: the shape parameter after the update
    //   b_pos: the rate parameter after the update
    //   alpha_pos: the concentration parameter after the update

    // calc E[lambda], E[ln lambda], and E[ln pi]
    VectorXd expt_lambda = a_pos.array() / b_pos.array();
    VectorXd expt_ln_lambda = stan::math::digamma(a_pos.array()) - stan::math::log(b_pos.array());
    VectorXd expt_ln_pi = stan::math::digamma(alpha_pos.array()) - stan::math::digamma(stan::math::sum(alpha_pos.array()));

    // calc E[ln eta], E[S], E[ln S]
    // S is a variable translated from s with one-hot-labeling
    MatrixXd expt_ln_eta(N,K);
    MatrixXd expt_ln_S(N,K);
    MatrixXd expt_S(N,K);
    for (int n = 0; n < N; n++) {
        expt_ln_eta.row(n) = X(n) * expt_ln_lambda - expt_lambda + expt_ln_pi;
        expt_ln_eta.row(n) = expt_ln_eta.row(n) - rep_matrix(stan::math::log_sum_exp(expt_ln_eta.row(n)),1,K);
        expt_ln_S.row(n) = expt_ln_eta.row(n);
    }
    expt_S = exp(expt_ln_S);

    // calc log-likelihood
    double expt_ln_lkh = 0;
    for (int n = 0; n < N; n++) {
        expt_ln_lkh += expt_S.row(n) * (X(n) * expt_ln_lambda - expt_lambda - rep_matrix(stan::math::lgamma(X(n)+1),K,1));
    }
    
    // calc E[ln p(S)] and E[ln q(S)]
    double expt_ln_pS = sum(expt_S * expt_ln_pi);
    double expt_ln_qS = sum(expt_S.array() * expt_ln_S.array());

    // calc KL[q(lambda) || p(lambda)]
    double KL_lambda = stan::math::sum(
        a_pos.array() * log(b_pos.array()) - a_pri.array() * log(b_pri.array()) -
        stan::math::lgamma(a_pos.array()) + stan::math::lgamma(a_pri.array()) +
        (a_pos.array() - a_pri.array()) * expt_ln_lambda.array() +
        (b_pri.array() - b_pos.array()) * expt_lambda.array()
    );

    // calc KL[q(pi) || p(pi)]
    double KL_pi = 
        stan::math::lgamma(sum(alpha_pos)) - stan::math::lgamma(sum(alpha_pri)) -
        sum(stan::math::lgamma(alpha_pos)) + sum(stan::math::lgamma(alpha_pri)) +
        (alpha_pos - alpha_pri).transpose() * expt_ln_pi;
    
    return expt_ln_lkh + expt_ln_pS - expt_ln_qS - (KL_lambda + KL_pi);
}
{{< / highlight >}}

[^csv]: [C++でCSVファイルを読み込む](https://cvtech.cc/readcsv/)