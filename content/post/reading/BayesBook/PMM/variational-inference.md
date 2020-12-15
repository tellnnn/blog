---
title: ポアソン混合モデルにおける推論 - 3. 変分推論
subtitle: 読書記録『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』
date: "2020-12-15"
toc: true
tags: ["Bayes","Machine Learning","C++","R"]
categories: ["Book"]
---

この記事では[読書記録『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』]({{< ref "/post/reading/BayesBook/TOC.md" >}})のうち、「4.3 ポアソン混合モデルにおける推論」にかんして、変分推論を実装します。

<!--more-->

[前回の記事](../gibbs-sampling)をお読みになってない方は、まずはそちらをご覧ください。

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

### VariationalInference 関数

さっそく、変分推論を行う関数 `VariationalInference` を定義します。今回は `generate_data` 関数（第1回の記事参照）によって生成されるデモデータに対し、変分推論を行うため、入力として以下のものを考えます。

|入力|型|概要|
|:--|:--|:--|
|`N`|整数|`generate_data` によって生成されたデータのサンプルサイズ|
|`K`|整数|クラスター数|
|`X`|長さ `N` のベクトル|`generate_data` によって生成されたデータ|
|`MAXITER`|整数|イテレーション数|
|`seed`|整数|乱数生成のためのシード値|

{{< highlight cpp "linenos=table,linenostart=192" >}}
void VariationalInference(int N, int K, VectorXi X, int MAXITER, int seed) {
    // function to do Variational Inference
    // inputs:
    //   N: the number of data points
    //   K: the number of clusters
    //   X: the data
    //   MAXITER: the maximum number of iterations
    //   seed: the random seed value

    // set random engine with the random seed value
    std::default_random_engine engine(seed);
    
    // set the output file
    std::ofstream samples("VI-samples.csv");
    // set the headers in the file
    for (int k = 0; k < K; k++) samples << "a." << k << ",";
    for (int k = 0; k < K; k++) samples << "b." << k << ",";
    for (int k = 0; k < K; k++) samples << "lambda." << k << ",";
    for (int k = 0; k < K; k++) samples << "alpha." << k << ",";
    for (int k = 0; k < K; k++) samples << "pi." << k << ",";
    for (int n = 0; n < N; n++) samples << "s." << n << ",";
    samples << "ELBO" << std::endl;
    
    // set variables
    VectorXd a; // the shape parameter
    VectorXd a_hat; // the modified shape parameter
    VectorXd b; // the rate parameter
    VectorXd b_hat; // the modified rate parameter
    VectorXd alpha; // the concentration parameter
    VectorXd alpha_hat; // the modified concentration parameter
    VectorXd lambda; // the rate parameter
    VectorXd pi; // the mixing parameter
    VectorXd s; // the latent variable
    MatrixXd expt_S(N,K); // E[S]
    MatrixXd ln_expt_S(N,K); // ln E[S]
    double ELBO; // ELBO
    
    // set initial values
    a = VectorXd::Constant(K,1,uniform_rng(0.1,2.0,engine));
    b = VectorXd::Constant(K,1,uniform_rng(0.005,0.05,engine));
    alpha = VectorXd::Constant(K,1,uniform_rng(10.0,200.0,engine));
    pi = dirichlet_rng(alpha,engine);
    s = VectorXd::Zero(N,1); // initialize s with zeros
    expt_S = MatrixXd::Zero(N,K); // initialize expt_S with zeros
    for (int n = 0; n < N; n++) {
        s(n) = categorical_rng(pi,engine);
        expt_S(n,s(n)-1) = 1;
    }
    a_hat = a + expt_S.transpose() * X;
    b_hat = b + expt_S.colwise().sum().transpose();
    alpha_hat = alpha + expt_S.colwise().sum().transpose();
    
    // sampling
    for (int i = 1; i <= MAXITER; i++) {
        // sample s
        VectorXd expt_lambda = a_hat.array() / b_hat.array();
        VectorXd expt_ln_lambda = stan::math::digamma(a_hat.array()) - stan::math::log(b_hat.array());
        VectorXd expt_ln_pi = stan::math::digamma(alpha_hat.array()) - stan::math::digamma(stan::math::sum(alpha_hat.array()));
        for (int n = 0; n < N; n++) {
            ln_expt_S.row(n) = X(n) * expt_ln_lambda - expt_lambda + expt_ln_pi;
            ln_expt_S.row(n) -= rep_row_vector(log_sum_exp(ln_expt_S.row(n)), K);
            expt_S.row(n) = exp(ln_expt_S.row(n));
            s(n) = categorical_rng(expt_S.row(n), engine);
        }
        
        // sample lambda
        a_hat = a + expt_S.transpose() * X;
        b_hat = b + expt_S.colwise().sum().transpose();
        lambda = to_vector(gamma_rng(a_hat,b_hat,engine));
        
        // sample pi
        alpha_hat = alpha + expt_S.colwise().sum().transpose();
        pi = dirichlet_rng(alpha_hat,engine);

        // calc ELBO
        ELBO = calc_ELBO(N, K, X, a, b, alpha, a_hat, b_hat, alpha_hat);
        
        // output
        for (int k = 0; k < K; k++) samples << a_hat(k) << ",";
        for (int k = 0; k < K; k++) samples << b_hat(k) << ",";
        for (int k = 0; k < K; k++) samples << lambda(k) << ",";
        for (int k = 0; k < K; k++) samples << alpha_hat(k) << ",";
        for (int k = 0; k < K; k++) samples << pi(k) << ",";
        for (int n = 0; n < N; n++) samples << s(n) << ",";
        samples << ELBO << std::endl;
    }
}
{{< / highlight >}}

乱数生成器と、結果を吐き出すファイル `VI-samples.csv` を準備したうえで変数をサンプリングしつつファイルに書き込んできます。

大まかな流れは「アルゴリズム 4.3」（書籍 p.137）にある通りで、パラメータの近似分布 $q(\boldsymbol{\lambda}), q(\boldsymbol{\pi})$ を初期化したうえで、各イテレーションにおいて $q(s_{n}), q(\lambda_{k}), q(\boldsymbol{\pi})$ を順に更新していきます。

ここでの実装では、$q(\boldsymbol{\lambda}), q(\boldsymbol{\pi})$ を初期化するにあたって、
$$
\begin{align*}
    a &\sim \text{Uniform}(0.1,2), \\\\
    b &\sim \text{Uniform}(0.005,0.05), \\\\
    \alpha &\sim \text{Uniform}(10,200),
\end{align*}
$$
および $a_{k} = a, b_{k} = b, \alpha_{k} = \alpha\\ (k = 1,\ldots, K)$ としておき、これらをもとに $\boldsymbol{\pi}$ を一度サンプリングしています。さらに、こうして得られた $\boldsymbol{\pi}$ を用いて $\textbf{S}$ を一度サンプリングしておき、それを用いて $q(\boldsymbol{\lambda}), q(\boldsymbol{\pi})$ すなわち、$\hat{a}_{k},\hat{b}_{k},\hat{\alpha}_{k}\\ (k = 1,\ldots,K)$ を初期化しています。

また、式(4.50)および式(4.51)における $\eta_{n,k}$ の算出について、実装では $\ln \eta_{n,k}$（変数としては `ln_expt_S`）として算出しておいて、あとで `exp` 関数に食わせるという手続きをとっています。

残りは上述の通りで、最後に他の推論手法と比較するために ELBO を計算しています（ELBO 計算の実装は[こちら](../demo-data#calc_elbo-関数)）。

また、今回もラベルスイッチングへの対応は行っていません[^stanref]。

[^stanref]: [Stan User's Guide. 20.2 Label Switching in Mixture Models](https://mc-stan.org/docs/2_25/stan-users-guide/label-switching-problematic-section.html#highly-multimodal-posteriors)

### main 関数

PMM.cpp を実行するさいに `N`, `K`, `X`, `MAXITER`, `seed` といった入力を渡しつつ `VariationalInference` 関数を実行できるよう、`main` 関数を定義し、入力はコマンドライン引数として渡すことにします。

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
{{< highlight cpp "linenos=table,linenostart=417">}}
        } else if (method == "VI") {

            std::cout << "Variational Inference" << std::endl;
            VariationalInference(N, K, X, MAXITER, seed);
{{< / highlight >}}
{{< highlight cpp "linenos=table,linenostart=431">}}
        }
    }
}
{{< / highlight >}}

ここで、コマンドライン引数の 1 番目として `VI` を受け取っており、事前に `generate_data` 関数で生成した `data.csv` を読み込んでいます。

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

では、次のような設定で生成されたデモデータ（第1回の記事と同じ）に対して変分推論を実行します。

|入力|概要|値|
|:--|:--|:--|
|`N`|サンプルサイズ|1000|
|`K`|クラスター数|2|
|`lambda`|各ポアソン分布のパラメータ|44, 77|
|`pi`|混合比率|0.5, 0.5|
|`seed`|乱数生成のためのシード値|6|

推論のパラメータにかんしては、デモデータと同じく `N = 1000`、`K = 2` とするとともに、`seed = 1`, `MAXITER = 1e+2` としておき、`system` 関数を使って実行します。

{{< highlight r "linenos=table,linenostart=163" >}}
# Variational Inference ------------------------------
method <- "VI"
vi_seed <- 1
MAXITER <- 1e+2

str_c("./PMM", method, N, K, vi_seed, as.integer(MAXITER), sep = " ") %>% system()
{{< / highlight >}}

生成されたサンプルは `VI-samples.csv` に保存されるので、それを読み込みます。

{{< highlight r "linenos=table,linenostart=170" >}}
VI <- list()

# set col_types to read samples.csv
pars_type <- c("d","d","d","d","d","i","d")
pars_dim <- c(K,K,K,K,K,N,1)
col_types <- rep(pars_type, pars_dim) %>% str_c(collapse = "")

# read samples.csv
VI$samples <- 
  read_csv(file = "VI-samples.csv", col_names = TRUE, col_types = col_types) %>% 
  mutate(iteration = 1:MAXITER)
{{< / highlight >}}

次に必要な変数 $\boldsymbol{\lambda},\boldsymbol{\pi},\textbf{S}$ を取り出します。

{{< highlight r "linenos=table,linenostart=182" >}}
# calculate EAP for lambda and sort them to control the order of clusters
sorted_lambda <- 
  VI$samples %>% 
  filter(iteration >= MAXITER / 2) %>% 
  summarise(across(starts_with("lambda."), .fns = mean)) %>% 
  pivot_longer(everything()) %>% 
  pull(value) %>% 
  sort.int(index.return = TRUE)
map_s <- 1:K
names(map_s) <- sorted_lambda$ix

# pull lambda samples
VI$lambda <- 
  VI$samples %>% 
  select(iteration, starts_with("lambda.")) %>% 
  pivot_longer(cols = starts_with("lambda."),
               names_to = "k", 
               names_pattern = "lambda.([0-9]+)", 
               names_transform = list(k = as.integer),
               values_to = "lambda", 
               values_transform = list(lambda = as.double)) %>% 
  mutate(k = recode(k + 1, !!!map_s))

# pull pi samples
VI$pi <- 
  VI$samples %>% 
  select(iteration, starts_with("pi.")) %>% 
  pivot_longer(cols = starts_with("pi."),
               names_to = "k", 
               names_pattern = "pi.([0-9]+)", 
               names_transform = list(k = as.integer),
               values_to = "pi", 
               values_transform = list(pi = as.double)) %>% 
  mutate(k = recode(k + 1, !!!map_s))

# pull s samples
VI$s <- 
  VI$samples %>% 
  select(iteration, starts_with("s.")) %>% 
  pivot_longer(cols = starts_with("s."), 
               names_to = "n", names_pattern = "s.([0-9]+)", names_transform = list(n = as.integer),
               values_to = "s", values_transform = list(s = as.integer)) %>% 
  mutate(s = recode(s, !!!map_s),
         X = demo_data$X[n+1])
{{< / highlight >}}

「# calculate EAP for lambda and sort them to control the order of clusters」における処理の意味については[ギブスサンプリングの記事](../gibbs-sampling)で説明してあるので、興味のある方は覗いてみてください（よりきれいな可視化のための操作で、省略しても問題のないものです）。

さて、$\lambda_{k}$ のサンプルからその平均と、95%信頼区間を算出してみると次のようになります。

{{< highlight r "linenos=table,linenostart=227" >}}
# Estimated lambda
VI$lambda %>% 
  filter(iteration >= MAXITER / 2) %>%
  group_by(k) %>% 
  mean_qi(lambda)
{{< / highlight >}}

```r
# A tibble: 2 x 7
      k lambda .lower .upper .width .point .interval
  <int>  <dbl>  <dbl>  <dbl>  <dbl> <chr>  <chr>    
1     1   43.9   43.3   44.5   0.95 mean   qi       
2     2   77.1   76.5   77.8   0.95 mean   qi  
```

実際の値は 44, 77 ですから、問題なく推論できていますね。

同様に $\pi_{k}$ のサンプルからその平均と、95%信頼区間を算出してみます。

{{< highlight r "linenos=table,linenostart=233" >}}
# Estimated pi
VI$pi %>% 
  filter(iteration >= MAXITER / 2) %>%
  group_by(k) %>% 
  mean_qi(pi)
{{< / highlight >}}

```r
# A tibble: 2 x 7
      k    pi .lower .upper .width .point .interval
  <int> <dbl>  <dbl>  <dbl>  <dbl> <chr>  <chr>    
1     1 0.501  0.478  0.525   0.95 mean   qi       
2     2 0.499  0.475  0.522   0.95 mean   qi    
```

実際の値が 0.5, 0.5 ですから、こちらも問題なさそうです。

さいごに、手元のデータ $x_{n}$ に対して、推論された潜在変数 $s_{n}$ の結果を見てみます。

{{< highlight r "linenos=table,linenostart=239" >}}
# plot
tmp_df <- 
  VI$s %>% 
  filter(iteration >= MAXITER / 2)
VI_plot <- 
  make_plot(
    method = method, 
    K = K,
    s = tmp_df$s, 
    X = tmp_df$X, 
    bins = 30
  )

(
  (demo_data_plot / VI_plot) +
    plot_layout(guides = "collect") &
    theme(aspect.ratio = 0.6,
          legend.position = "bottom")
) %>% 
  ggsave(filename = "VI_result.png", width = 100, height = 150, units = "mm")
{{< / highlight >}}

{{< figure library="true" src="./content/post/reading/BayesBook/PMM/VI_result.png" title="デモデータと推論結果" numbered="true" width="400">}}

上の図は、ヒストグラムを描くにあたってデモデータと同じビンを設定し、各ビンに含まれている $x_{n}$ が推論を通して、各クラスターに割り当てられた回数を数えたものです。たとえば、推論が比較的難しい2つの分布の中間、x 軸上で 50 付近のデータは、デモデータ上ではすべてクラスターAに属していますが、推論の最中に何度かクラスターBに割り当てられていることがわかります。ただ、全体を通して見てみるとかなり正確に推論ができていることがわかります。

というわけで、今回は変分推論を実装しました。以降の記事では、別の手法での推論を実装し、最後に今回も計算した ELBO を使って手法間の比較をしてみます。