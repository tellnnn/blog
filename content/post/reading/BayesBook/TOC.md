---
title: 読書記録『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』
date: "2020-08-30"
toc: true
tags: ["Bayes", "Machine Learning","C++","R"]
categories: ["Book"]
---

須山敦志『機械学習スタートアップシリーズ ベイズ推論による機械学習入門』の読書記録です。

<!--more-->

この読書記録では自分の理解のため、[サポートページ](https://github.com/sammy-suyama/BayesBook) で公開されている Julia による実装を C++ と R で実装し直しています（[GitHub リポジトリ](https://github.com/tellnnn/BayesBook_mine)）。実装にあたり、サンプリングなど主な部分は C++ および [Stan Math Library](https://mc-stan.org/users/interfaces/math) を用い、結果の可視化などは R で行っています。

とくに C++ には不慣れなので、間違いなど見つけましたらご連絡いただければと思います。

## 記事目録

以下は記事の目録で、書籍の目次になるだけ沿うかたちにしています。記事が完成しだい追加していきます。

-----

### 第 4 章 混合モデルと近似推論

#### 3. ポアソン混合モデルにおける推論

1. デモデータの生成（[読む](../pmm/demo-data)）
2. ギブスサンプリング（[読む](../pmm/gibbs-sampling)）
3. 変分推論（作成中）
4. 崩壊型ギブスサンプリング（作成中）
5. まとめ（作成中）

-----

## 参考文献

すべての記事で必ず参照しているもの文献が複数ありますので、各記事で引用文献に挙げる代わりにここでまとめて挙げさせていただきます。

- 須山敦志. (2017). *機械学習スタートアップシリーズ ベイズ推論による機械学習入門*. 講談社. https://www.kspub.co.jp/book/detail/1538320.html
- Stan - Stan Math Library. https://mc-stan.org/users/interfaces/math
- Eigen. http://eigen.tuxfamily.org/index.php?title=Main_Page
- シ. (2020). *C++行列ライブラリEigenのメモ*. 暗黒通信団.