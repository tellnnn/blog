---
title: ポアソン混合モデルにおける推論 - 4. 崩壊型ギブスサンプリング
subtitle: 読書記録『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』
date: "2020-12-16"
toc: true
tags: ["Bayes","Machine Learning","C++","R"]
categories: ["Book"]
---

この記事では[読書記録『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』]({{< ref "/post/reading/BayesBook/TOC.md" >}})のうち、「4.3 ポアソン混合モデルにおける推論」にかんして、崩壊型ギブスサンプリングによる推論を実装します。

<!--more-->

[前回の記事](../variational-inference)をお読みになってない方は、まずはそちらをご覧ください。

## はじめに

この記事は、いわゆる須山ベイズ本「4.3 ポアソン混合モデルにおける推論」にかんする一連の記事のひとつです。この節での実装は

PMM.cpp
: ギブスサンプリングなど推論の実装

PMM.R
: 推論の実行やその推論結果の可視化などの実装

の 2 つのファイルにまとめており [GitHub のリポジトリ](https://github.com/tellnnn/BayesBook_mine) で公開しています。

各記事では、これらのファイルの該当箇所を順に説明していくかたちをとるので、関心のある方は適宜参照してください。数式やその説明をどこまで記載してよいかわからなかったので、この記事は書籍を傍らに置きながら読まれることを想定しております。

なお、改善点等ありましたらご指摘いただけると幸いです。

## 実装 (PMM.cpp)

### 準備

まずは一連の実装で必要となるライブラリを読み込みます（前回の記事に同じです）。

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

### CollapsedGibbsSampling 関数

さっそく、崩壊型ギブスサンプリングによる推論を行う関数 `CollapsedGibbsSampling` を定義します。今回は `generate_data` 関数（第1回の記事参照）によって生成されるデモデータに対し、崩壊型ギブスサンプリングによる推論を行うため、入力として以下のものを考えます。

|入力|型|概要|
|:--|:--|:--|
|`N`|整数|`generate_data` によって生成されたデータのサンプルサイズ|
|`K`|整数|クラスター数|
|`X`|長さ `N` のベクトル|`generate_data` によって生成されたデータ|
|`MAXITER`|整数|イテレーション数|
|`seed`|整数|乱数生成のためのシード値|

{{< highlight cpp "linenos=table,linenostart=280" >}}
void CollapsedGibbsSampling(int N, int K, VectorXi X, int MAXITER, int seed) {
    // function to do Collapsed Gibbs Sampling
    // inputs:
    //   N: the number of data points
    //   K: the number of clusters
    //   X: the data
    //   MAXITER: the maximum number of iterations
    //   seed: the random seed value

    // set random engine with the random seed value
    std::default_random_engine engine(seed);
    
    // set the output file
    std::ofstream samples("CGS-samples.csv");
    // set the headers in the file
    for (int k = 0; k < K; k++) samples << "a." << k << ",";
    for (int k = 0; k < K; k++) samples << "b." << k << ",";
    for (int k = 0; k < K; k++) samples << "alpha." << k << ",";
    for (int n = 0; n < N; n++) samples << "s." << n << ",";
    samples << "ELBO" << std::endl;
    
    // set variables
    VectorXd a; // the shape parameter
    VectorXd a_hat; // the modified shape parameter
    VectorXd b; // the rate parameter
    VectorXd b_hat; // the modified rate parameter
    VectorXd alpha; // the concentration parameter
    VectorXd alpha_hat; // the modified concentration parameter
    VectorXd s; // the latent variable
    MatrixXd S(N,K); // the latent variable (one-hot-labeling)
    VectorXd sX_csum; // represents the sum_{n}^{N}(s_{n,k} * x_{n})
    VectorXd S_csum; // represents the sum_{n}^{n}(s_{n,k})
    VectorXd ln_lkh; // represents (log-) likelihood at Eq. (4.76)
    double ELBO; // ELBO
    
    // set initial values
    a = VectorXd::Constant(K,1,uniform_rng(0.1,2.0,engine));
    b = VectorXd::Constant(K,1,uniform_rng(0.005,0.05,engine));
    alpha = VectorXd::Constant(K,1,uniform_rng(10.0,200.0,engine));
    s = VectorXd::Zero(N,1); // initialize s with zeros
    S = MatrixXi::Zero(N,K); // initialize S with zeros
    for (int n = 0; n < N; n++) {
        s(n) = categorical_rng(rep_vector(1.0/K,K),engine);
        S(n,s(n)-1) = 1;
    }
    // calc a_hat, b_hat, and alpha_hat
    sX_csum = (S.array() * X.rowwise().replicate(K).array()).matrix().colwise().sum();
    S_csum = S.colwise().sum();
    a_hat = a + sX_csum;
    b_hat = b + S_csum;
    alpha_hat = alpha + S_csum;
    
    // sampling
    for (int i = 1; i <= MAXITER; i++) {
        for (int n = 0; n < N; n++) {
            // remove components related to x_{n}
            a_hat -= S.row(n) * X(n);
            b_hat -= S.row(n);
            alpha_hat -= S.row(n);

            // calc ln_lkh
            ln_lkh = VectorXd::Zero(K); // initialize ln_lkh with zeros
            for (int k = 0; k < K; k++) {
                ln_lkh(k) = neg_binomial_lpmf(X(n),a_hat(k),b_hat(k)) + stan::math::log(alpha_hat(k));
            }

            // sample s
            S = MatrixXi::Zero(N,K); // initialize S with zeros
            ln_lkh -= rep_vector(log_sum_exp(ln_lkh),K);
            s(n) = categorical_rng(stan::math::exp(ln_lkh),engine);
            S(n,s(n)-1) = 1;

            // add components related to x_{n}
            a_hat += S.row(n) * X(n);
            b_hat += S.row(n);
            alpha_hat += S.row(n);
        }

        // calc ELBO
        ELBO = calc_ELBO(N, K, X, a, b, alpha, a_hat, b_hat, alpha_hat);
        
        // output
        for (int k = 0; k < K; k++) samples << a_hat(k) << ",";
        for (int k = 0; k < K; k++) samples << b_hat(k) << ",";
        for (int k = 0; k < K; k++) samples << alpha_hat(k) << ",";
        for (int n = 0; n < N; n++) samples << s(n) << ",";
        samples << ELBO << std::endl;
    }
}
{{< / highlight >}}

乱数生成器と、結果を吐き出すファイル `CGS-samples.csv` を準備したうえで変数をサンプリングしつつファイルに書き込んできます。

大まかな流れは「アルゴリズム 4.4」（書籍 p.142）にある通りで、$\boldsymbol{s}_1,\ldots,\boldsymbol{s}_N$ に初期値を設定したうえで、$\hat{\alpha}, \hat{a}, \hat{b}$ を計算し、各イテレーションにおいて順に更新していきます。

ここでの実装では、$\boldsymbol{s}_1,\ldots,\boldsymbol{s}_N$ を初期化するにあたって、
$$
\begin{align*}
    a &\sim \text{Uniform}(0.1,2), \\\\
    b &\sim \text{Uniform}(0.005,0.05), \\\\
    \alpha &\sim \text{Uniform}(10,200),
\end{align*}
$$
および $a_{k} = a, b_{k} = b, \alpha_{k} = \alpha\\ (k = 1,\ldots, K)$ としています。さらに、$\pi_{k} = 1/k$ のもと $\textbf{S}$ を一度サンプリングしています。

残りは上述の通りで、最後に他の推論手法と比較するために ELBO を計算しています（ELBO 計算の実装は[こちら](../demo-data#calc_elbo-関数)）。

また、今回もラベルスイッチングへの対応は行っていません[^stanref]。

[^stanref]: [Stan User's Guide. 20.2 Label Switching in Mixture Models](https://mc-stan.org/docs/2_25/stan-users-guide/label-switching-problematic-section.html#highly-multimodal-posteriors)

### main 関数

PMM.cpp を実行するさいに `N`, `K`, `X`, `MAXITER`, `seed` といった入力を渡しつつ `CollapsedGibbsSampling` 関数を実行できるよう、`main` 関数を定義し、入力はコマンドライン引数として渡すことにします。

{{< highlight cpp "linenos=table,linenostart=370" >}}
int main(int argc, char *argv[]) {
    
    // get inputs 1 ~ 4
    std::string method = argv[1];
    int N = atoi(argv[2]);
    int K = atoi(argv[3]);
    int seed = atoi(argv[4]);

    if (method == "data") {
{{< / highlight >}}
{{< highlight cpp "linenos=table,linenostart=392">}}
    } else {
        
        int MAXITER = atoi(argv[5]);

        // read data.csv
        VectorXi X;
        X = VectorXi::Zero(N);
        std::ifstream ifs("data.csv");
        std::string line;
        int idx = -1;
        while (getline(ifs, line)) {
            std::vector<std::string> strvec = split(line, ',');
            if (idx == -1) {
                idx++;
                continue;
            }
            X(idx) = stoi(strvec.at(1));
            idx++;
        }
        
        if (method == "GS") {
{{< / highlight >}}
{{< highlight cpp "linenos=table,linenostart=422">}}
        } else if (method == "CGS") {

            std::cout << "Collapsed Gibbs Sampling" << std::endl;
            CollapsedGibbsSampling(N, K, X, MAXITER, seed);
{{< / highlight >}}
{{< highlight cpp "linenos=table,linenostart=431">}}
        }
    }
}
{{< / highlight >}}

ここで、コマンドライン引数の 1 番目として `CGS` を受け取っており、事前に `generate_data` 関数で生成した `data.csv` を読み込んでいます。

## 実行 (PMM.R)

PMM.cpp をコンパイルして実行し、推論結果が確認できるように、PMM.R を実装します。

### 準備

（前回の記事と同じです）

{{< highlight r "linenos=table,linenostart=1" >}}
# setwd("./PoissonMixtureModel/")


# library ------------------------------
library(tidyverse)
library(colorspace)
library(patchwork)
library(ggdist)


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

（前回の記事と同じです）

{{< highlight r "linenos=table,linenostart=25" >}}
# compile c++ file ------------------------------
stan_math_standalone <- "$HOME/.cmdstanr/cmdstan-2.24.0/stan/lib/stan_math/make/standalone"
str_c("make -j4 -s -f", stan_math_standalone, "math-libs", sep = " ") %>% system()
str_c("make -j4 -s -f", stan_math_standalone, "PMM",       sep = " ") %>% system()
{{< / highlight >}}

### 実行

では、次のような設定で生成されたデモデータ（第1回の記事と同じ）に対して崩壊型ギブスサンプリングによる推論を実行します。

|入力|概要|値|
|:--|:--|:--|
|`N`|サンプルサイズ|1000|
|`K`|クラスター数|2|
|`lambda`|各ポアソン分布のパラメータ|44, 77|
|`pi`|混合比率|0.5, 0.5|
|`seed`|乱数生成のためのシード値|6|

推論のパラメータにかんしては、デモデータと同じく `N = 1000`、`K = 2` とするとともに、`seed = 1`, `MAXITER = 1e+2` としておき、`system` 関数を使って実行します。

{{< highlight r "linenos=table,linenostart=261" >}}
# Collapsed Gibbs Sampling ------------------------------
method <- "CGS"
cgs_seed <- 1
MAXITER <- 1e+2

str_c("./PMM", method, N, K, cgs_seed, MAXITER, sep = " ") %>% system()
{{< / highlight >}}

生成されたサンプルは `CGS-samples.csv` に保存されるので、それを読み込みます。

{{< highlight r "linenos=table,linenostart=268" >}}
CGS <- list()

# set col_types to read samples.csv
pars_type <- c("d","d","d","i","d")
pars_dim <- c(K,K,K,N,1)
col_types <- rep(pars_type, pars_dim) %>% str_c(collapse = "")

# read samples.csv
CGS$samples <- 
  read_csv(file = "CGS-samples.csv", col_names = TRUE, col_types = col_types) %>% 
  mutate(iteration = 1:MAXITER)
{{< / highlight >}}

次に必要な変数 $\textbf{S}$ を取り出します。

{{< highlight r "linenos=table,linenostart=280" >}}
# pull s samples
CGS$s <- 
  CGS$samples %>% 
  select(iteration, starts_with("s.")) %>% 
  pivot_longer(cols = starts_with("s."), 
               names_to = "n", 
               names_pattern = "s.([0-9]+)", 
               names_transform = list(n = as.integer),
               values_to = "s", 
               values_transform = list(s = as.integer)) %>% 
  mutate(X = demo_data$X[n+1])

# calculate MLE for lambda and sort them to control the order of clusters
sorted_lambda <- 
  CGS$s %>% 
  group_by(s) %>% 
  summarise(lambda_MLE = sum(X) / n(), .groups = "drop") %>% 
  pull(lambda_MLE) %>% 
  sort.int(index.return = TRUE)
map_s <- 1:K
names(map_s) <- sorted_lambda$ix

# recode cluster indices
CGS$s <- 
  CGS$s %>% 
  mutate(s = recode(s, !!!map_s))
{{< / highlight >}}

「# calculate MLE for lambda and sort them to control the order of clusters」における処理の意味については[ギブスサンプリングの記事](../gibbs-sampling)で説明してあるので、興味のある方は覗いてみてください（よりきれいな可視化のための操作で、省略しても問題のないものです）。

手元のデータ $x_{n}$ に対して、推論された潜在変数 $s_{n}$ の結果を見てみます。

{{< highlight r "linenos=table,linenostart=307" >}}
# plot
tmp_df <- 
  CGS$s %>% 
  filter(iteration >= MAXITER / 2)
CGS_plot <- 
  make_plot(
    method = method, 
    K = K,
    s = tmp_df$s, 
    X = tmp_df$X, 
    bins = 30
  )

(
  (demo_data_plot / CGS_plot) +
    plot_layout(guides = "collect") &
    theme(aspect.ratio = 0.6,
          legend.position = "bottom")
) %>% 
  ggsave(filename = "CGS_result.png", width = 100, height = 150, units = "mm")
{{< / highlight >}}

{{< figure library="true" src="./content/post/reading/BayesBook/PMM/CGS_result.png" title="デモデータと推論結果" numbered="true" width="400">}}

上の図は、ヒストグラムを描くにあたってデモデータと同じビンを設定し、各ビンに含まれている $x_{n}$ が推論を通して、各クラスターに割り当てられた回数を数えたものです。たとえば、推論が比較的難しい2つの分布の中間、x 軸上で 50 付近のデータは、デモデータ上ではすべてクラスターAに属していますが、推論の最中に何度かクラスターBに割り当てられていることがわかります。ただ、全体を通して見てみるとかなり正確に推論ができていることがわかります。

というわけで、今回は崩壊型ギブスサンプリングによる推論を実装しました。以降の記事では、今回も計算した ELBO を使って手法間の比較をしてみます。