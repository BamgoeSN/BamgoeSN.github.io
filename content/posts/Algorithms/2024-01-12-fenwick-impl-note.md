---
layout: post
title: "Fenwick Tree 구현 노트"
date: 2024-01-12 00:17:44 +0900
categories: ["Algorithms"]
tags: ["Data Structures", "Segment Tree"]
katex: true
description: "매번 유도하기 싫어서 적어두는 펜윅 트리 구현 노트"
---

펜윅 트리가 무엇인지는 소개하지 않는다. 구글에 검색해보면 좋은 자료가 쏟아진다. 그래도 어떤 연산을 수행할 수 있는지 아래 pseudocode의 주석을 보면 알 수 있게 해놓았다.

여기서 소개하는 방법 말고도 펜윅 트리를 구현하는 방법은 많다. 오히려 이 방법은 일반적으로 사용되는 구현 형태가 아니다. 다만 이 방법이 내가 주로 사용하는 index 체계와 맞도록 하는 구현이어서 나는 이걸 사용한다.

# 구현 노트

모든 인덱스는 0-based이다. $l..r$이라는 구간은 $[l, r)$을 의미하고, $l..=r$이라는 구간은 $[l, r]$을 의미한다. 이는 Rust에서 사용하는 구간 표기법과 같다.

## Pseudocode

```python
def m(x): return i & (~i + 1)  # the minimum bit of x
def b(x): return x - m(x)

# Initialize fw with a given array arr
def initialize(arr):
	fw = [0]
	for i, v in enumerate(arr):
		fw.append(fw[i] + v)
	for r in reversed(range(1, n+1)):
		fw[r] -= fw[b(r)]
	return fw

# Returns sum of arr[0..x]
def prefix(fw, x):
	k, v = x, 0
	while k != 0:
		fw[k] += v
		k = b(k)
	return v

# Adds v to arr[x]
def update(fw, x, v):
	k = x + 1
	while k < len(fw):
		v += fw[k]
		k += m(k)
```

## 정의

함수 $m(x)$는 $x$에서 가장 작은 비트만을 남긴 것이고, $b(x)$는 $x-m(x)$로 정의된다.

길이 $n$의 배열 $A$에 대한 펜윅 트리 배열 $F$는 다음과 같이 정의된다.

$$
F_x = \begin{cases}
0 & \textrm{if } x = 0 \\\\
\sum A_{b(x)...x} & \textrm{otherwise}
\end{cases}
$$

여기서 $\sum A_{b(x) .. x}$는 인덱스가 $b(x)..x$인 $A$ 값을 더한 것을 의미한다.

## Initialize

처음에 배열 $A$가 주어졌을 때 $O(n)$ 시간 안에 $F$를 초기화하는 알고리즘은 다음과 같다.

1. $F_x = \sum A_{0..x}$로 초기화한다. 즉, 누적합 배열을 만든다.
2. For $r$ in $1..=n$ in reverse…
    1. $l \leftarrow b(r)$
    2. Subtract $F_l$ from $F_r$.

## Prefix sum

$A_{0..x}$의 합을 계산하는 알고리즘은 다음과 같다.

1. $k \leftarrow x$, $v \leftarrow 0$.
2. While $k \ne 0$
    1. Add $F_k$ to $v$.
    2. $k \leftarrow b(k)$
3. Return $v$.

## Point update

$A_x$에 $v$를 더하는 알고리즘은 다음과 같다.

1. $k \leftarrow x + 1$
2. While $k \le n$
    1. Add $v$ to $F_k$.
    2. Add $m(k)$ to $k$.

## A little bit faster range sum

$A_{l..r}$의 합을 조금이나마 빠르게 계산하기 위해 펜윅 트리에서 $l$과 $r$의 LCA를 구하여 각 정점에서 LCA까지만 구간합을 계산할 수 있다.

$l$과 $r$의 LCA를 계산하는 알고리즘은 다음과 같다. 단, $l \le r$이 보장되어야 한다.

1. If $l = r$ then return $l$.
2. $x \leftarrow l \oplus r$. 여기서 $\oplus$는 bitwise xor이다.
3. Let $f$ as the lowest set bit of $x$.
4. $m \leftarrow \ !(2f - 1)$. 여기서 $!$는 bitwise not이다.
5. Return $r \\& m$.

그러면 $A_{l..r}$의 함수를 구하는 코드는 다음과 같이 변한다. 여기서 $m$은 $l$과 $r$의 LCA이다.

1. $v \leftarrow 0$.
2. While $r \ne m$
    1. Add $F_r$ to $v$.
    2. $r \leftarrow b(r)$.
3. While $l \ne m$
    1. Subtract $F_l$ from $v$.
    2. $l \leftarrow b(l)$.
4. Return $v$.