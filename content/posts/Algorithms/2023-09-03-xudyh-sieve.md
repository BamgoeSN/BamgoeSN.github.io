---
layout: post
title: "xudyh's Sieve"
date: 2023-09-03 16:00:00 +0900
categories: ["Algorithms"]
katex: true
description: " "
---

**xudyh's sieve**는 multiplicative function $f$가 주어졌을 때, $\sum_{i=1}^n {f(i)}$를 $O(n^{2/3})$ 안에 계산하는 알고리즘입니다.

여기에는 알고리즘의 작동 방식에 대한 요약만 적어놓습니다. 구체적인 내용은 rkm0959님이 잘 다뤄주셨습니다. 아래 요약도 이 블로그 글을 보고 가져온 내용입니다. <https://rkm0959.tistory.com/190>

# 요약

함수 $f$의 prefix sum을 $s_f(n) = \sum_{i=1}^n f(i)$라고 표기합시다. 그리고, 두 함수 $f$와 $g$의 Dirichlet convolution $f*g$를 아래와 같이 정의합시다.

$$
(f*g)(n) = \sum_{d | n} {f(d) g(n/d)}
$$

주어진 $f$에 대해 적절한 $g$를 정하여, $g$와 $f*g$의 prefix sum을 쉽게 계산할 수 있도록 합시다. 그러면, 아래 점화식을 활용하여 $s_f(n)$을 $O(n^{2/3})$ 안에 계산할 수 있습니다.

$$
s_f(n) = s_{f*g}(n) - \sum_{d=2}^{\left\lfloor n/{\left\lfloor \sqrt{n} \right\rfloor} \right\rfloor} g(d) s_f \left(\left\lfloor\frac{n}{d}\right\rfloor\right) - \sum_{e=1}^{ {\left\lfloor \sqrt{n} \right\rfloor}-1} s_f (e) \left\\{ s_g\left( \left\lfloor \frac{n}{e} \right\rfloor \right) - s_g\left( \left\lfloor \frac{n}{e+1} \right\rfloor \right) \right\\}
$$

## 구현

실제 구현에선, $s_f$의 값을 $n^{2/3}$까지는 $\tilde{O}(n^{2/3})$ 시간 안에 계산해놓습니다. 여기에는 에라토스테네스의 체나 linear sieve를 사용할 수 있습니다. 그런 다음, 위 점화식을 활용하여 탑다운 DP로 정답을 구하면 됩니다.

탑다운 DP를 하는 과정에서 $s_f$의 값을 캐싱하게 될 텐데, 이때 캐싱하는 값들은 $s_f(\left\lfloor n/d \right\rfloor)$의 형태를 갖습니다. 캐싱의 키로 $s_f$에 들어가는 인자를 그대로 사용할 경우 그 크기가 매우 커질 수 있어 해시맵과 같은 자료구조를 사용해야 하는데, 그 대신 $d$를 쓰면 벡터만으로 캐싱을 할 수 있습니다.