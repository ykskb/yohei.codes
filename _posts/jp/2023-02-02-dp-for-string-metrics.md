---
layout: default
title: "文字列に効く動的計画法 (DP)"
lang: jp
---

# 文字列に効く動的計画法 (DP)

動的計画法(DP)は複雑な問題をより小さく単純な部分問題に分割し解決する手法です。その中には文字列メトリクスに対して効果的なパターンもいくつかあり、このシリーズはそれらに関する私のまとめノートです。何か追加すべきものがあると思われる場合は、お気軽にお問い合わせください。

> ## 記事リスト
>
> - [レーベンシュタイン距離](/2023/02/03/dp-levenshtein.html)
> - [最長共通部分列(LCS)](/2023/02/05/dp-lcs.html)
> - [正規表現チェック](/2023/02/06/dp-regex.html)
> - Distinct Subsequence (coming soon)
> - Longest Repeating Subsequence (coming soon)
> - Hamming Distance (coming soon)

## 読む前提条件

出来るだけ噛み砕いて説明するよう努力していますが、基本的な動的計画法自体の説明になっているかどうかは自信がありませんので、DP 自体への入門はこちらの[記事](https://qiita.com/drken/items/a5e6fe22863b7992efdb)を参照していただけたらと思います。

## 理解のコツ：DP テーブルの初期化と更新

DP の解法や実装方法はいくつかあるのですが、このシリーズでは所謂ボトムアップというアプローチを使っています。トップダウンでは再帰が使われるのに対し、ボトムアップはテーブル（タブとも呼ばれる）が使われ可視化しやすいので自分もボトムアップで理解することの方が多いです。

レーベンシュタイン距離の例：

<img src="/assets/images/levenshtein.png" style="background-color: #FFF;">
<!-- ![levenshtein](/assets/images/levenshtein.png) -->

このシリーズは文字列に対する動的計画法をまとめていますが、問題によりその応用は様々です。それらを頭の中で整理するコツは DP テーブルが目的に応じてどのように初期化されるか、そしてどの様に更新されるかを問題毎に理解することだと思います。

鍵となるテーブル更新の実装部分：

```python
if s[i-1] == target[j-1]:  # if characters are identical, cost is 0
    substitute_cost = 0
else:
    substitute_cost = 1
dp[i][j] = min(dp[i-1][j] + 1,  # delete
                dp[i][j-1] + 1,  # insert
                dp[i-1][j-1] + substitute_cost)  # substitute
```