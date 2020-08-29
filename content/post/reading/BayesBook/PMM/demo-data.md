---
title: ポアソン混合モデルにおける推論 - 1. デモデータの生成
date: "2020-08-15"
toc: true
tags: ["Bayes", "Machine Learning","C++","R"]
categories: ["Book"]
draft: true
---

この記事では『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』のうち、「4.3 ポアソン混合モデルにおける推論」で使用するデモデータの生成を実装します。

<!--more-->

## はじめに

ここで紹介する

{{< highlight "c++" "linenos=table,linenostart=1" >}}
#include <stan/math.hpp>
#include <Eigen/Dense>
#include <iostream>
#include <fstream>
#include <random>
#include <string>
#include <sstream>
#include <vector>
#include <stdio.h>

using namespace stan;
using namespace math;
using namespace Eigen;

void generate_data(int N, int K, double a, double b, double alpha, int seed) {
    // function to generate random data
    // inputs:
    //   N: the number of data points
    //   K: the number of clusters
    //   a: the shape parameter in gamma distribution
    //   b: the rate parameter in gamma distribution
    //   alpha: the concentration parameter in dirichlet distribution
    //   seed: the random seed value

    // set random engine with the random seed value
    std::default_random_engine engine(seed);

    // set variables
    VectorXd lambda; // the rate parameter in poisson distribution
    VectorXd pi; // the mixing parameter
    MatrixXi s(N,1); // the latent variable
    MatrixXi X(N,1); // the data

    // sample lambda
    lambda = to_vector(gamma_rng(rep_vector(a,K),rep_vector(b,K),engine));
    // sample pi
    pi = dirichlet_rng(rep_vector(alpha,K),engine);

    // set output file
    std::ofstream params("params.csv");
    // set the header in the file
    params << "a,b,lambda,alpha,pi" << std::endl;
    for (int k = 0; k < K; k++) {
        // output parameters
        params << a         << "," << 
                  b         << "," << 
                  lambda(k) << "," <<
                  alpha     << "," << 
                  pi(k)     << std::endl;
    }

    // set the output file
    std::ofstream data("data.csv");
    // set the header in the file
    data << "s,X" << std::endl;
    for (int n = 0; n < N; n++) {
        // sample s and X
        s(n) = categorical_rng(pi,engine);
        X(n) = poisson_rng(lambda(s(n)-1),engine);
        // output s and X
        data << s(n) << "," << X(n) << std::endl;
    }
}
{{< / highlight >}}

ああ
