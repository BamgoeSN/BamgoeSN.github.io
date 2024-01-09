---
layout: post
title: 컨벡스 헐 쉽게 구하기 - 모노톤 체인
date: 2023-09-10 23:30:00 +0900
categories: ["Algorithms"]
tags: ["Geometry"]
katex: true
description: " "
---

* 이 글은 독자가 CCW를 알고 있음을 전제합니다. 처음 들어보거나 뭘 말하는 건지 모르겠다면 구글에 '백준 CCW'를 검색해 보세요.

컨벡스 헐, 또는 볼록 껍질은 좌표평면에 주어진 $N$개의 점에 대해, 그 $N$개의 점을 모두 포함하면서 면적이 가장 작은 다각형을 의미합니다.

평면에서의 컨벡스 헐은 $O(N \log{N})$ 안에 구할 수 있음이 알려져 있으며, 그 방법으로 Graham scan, 그레이엄 스캔이라는 알고리즘이 널리 알려져 있습니다.
이 알고리즘은 모든 점을 한 점을 기준으로 각도 정렬을 하는 과정을 포함하는데, 여기서 정렬에 쓰는 비교 함수를 작성할 때 실수가 나기 매우 쉬워 처음 배우려는 사람들에게 고통을 선사합니다. (제가 그리 느꼈습니다.)

[좌표평면에서 컨벡스 헐을 구하는 알고리즘은 그레이엄 스캔만 있지 않습니다.](https://en.wikipedia.org/wiki/Convex_hull_algorithms#Algorithms) 심지어 그중에서 그레이엄 스캔이 가장 쉬운 방법인 것도 아닙니다.
이 글에서는 그레이엄 스캔만큼 빠르고, 난이도도 훨씬 쉬우며, 컨벡스 헐의 한 변 위에 여러 점이 있어도 구현이 거의 동일한 **모노톤 체인 컨벡스 헐** 알고리즘을 소개하려고 합니다.

# 알고리즘

<center><figure>
    <img src="https://upload.wikimedia.org/wikipedia/commons/9/9a/Animation_depicting_the_Monotone_algorithm.gif">
    <figcaption><a href="https://en.wikibooks.org/wiki/File:Animation_depicting_the_Monotone_algorithm.gif">image by Maonus</a>, distributed under a <a href="https://creativecommons.org/licenses/by-sa/4.0/deed.en">CC BY-SA 4.0 license</a>.<br>좀 검은색 점으로 해주지 징그럽게...</figcaption>
</figure></center>

모노톤 체인 컨벡스 헐 알고리즘은, 주어진 모든 점의 위쪽을 덮는 **위쪽 껍질**을 만든 다음, 아래쪽을 덮는 **아래쪽 껍질**을 만들어서, 그 둘을 합치는 형태로 작동합니다.

위 사진은 점이 여러 개 주어졌을 때 그에 대해 모노톤 체인 알고리즘을 실행하는 사진입니다. 왼쪽부터 오른쪽으로 점을 훑으면서 위쪽 껍질을 덮은 후, 반대 방향으로 돌아오면서 아래쪽 껍질을 덮는 모습을 볼 수 있습니다.

바로 구현체부터 보시겠습니다.

```py
def ccw(a, b, c):
    u = (b[0]-a[0], b[1]-a[1]);
    v = (c[0]-b[0], c[1]-b[1]);
    return u[0]*v[1] - u[1]*v[0]

def cvh(arr):
    arr.sort()
    
    upper = []
    for p in arr:
        while len(upper) >= 2 and ccw(upper[-2], upper[-1], p) >= 0:
            upper.pop()
        upper.append(p)

    lower = []
    for p in reversed(arr):
        while len(lower) >= 2 and ccw(lower[-2], lower[-1], p) >= 0:
            lower.pop()
        lower.append(p)

    return upper[:-1] + lower[:-1]
```

여기서 `cvh` 함수는 `(int, int)` 꼴의 튜플의 리스트인 `arr`를 인자로 받습니다.
각도 정렬 같은 건 필요 없습니다. 그냥 똑같이 생긴 루프 두 번 돌려주면 끝입니다.

그럼, 위 코드가 어떤 일을 하는지 들여다봅시다.

```py
cvh.sort()
```
`cvh`는 제일 먼저 `arr`을 정렬합니다. 즉, 모든 점을 *x*좌표에 대해 오름차순 정렬하되, *x*좌표가 같은 점들은 *y*좌표에 대해 오름차순으로 정렬합니다.

```py
upper = []
```
그런 다음, `upper`라는 빈 리스트를 만듭니다.
이 다음에 나오는 루프가 지나면 `upper`는 위쪽 껍질을 이루는 점의 리스트가 되며, *x*좌표 *오름차순*으로 정렬된 형태가 됩니다.

*x*좌표 오름차순으로 정렬된다는 얘기는, `upper`의 모든 점은 순서대로 봤을 때 **모든 점이 시계 방향으로 배치된다**는 걸 의미하기도 합니다.
위쪽 절반을 덮는 껍질을 *x*좌표 오름차순으로 본다면 당연히 시계 방향 순서로 나오겠죠?
이 성질이 다음 루프에서 `upper`를 구성해 주는 과정의 핵심입니다.

```py
for p in arr:
    while len(upper) >= 2 and ccw(upper[-2], upper[-1], p) >= 0:
        upper.pop()
    upper.append(p)
```
이어지는 `for` 루프의 내용을 봅시다.
이 루프는 `arr`를 순서대로 보면서 진행합니다. 즉, 주어진 모든 점을 *x*좌표 오름차순으로 순회하며, *x*좌표가 같은 점은 *y*좌표가 작은 것부터 봅니다.

이 루프 안에서, 우리는 `upper` 안에 `p`를 추가하고 싶습니다. 그런데, `upper`는 **모든 점이 시계 방향으로 배치되어 있어야 합니다.**
그래서 그 안의 `while` 루프에서, `upper`의 길이가 2 이상이고, `upper`의 마지막 두 점과 `p`가 반시계 방향이라면 `upper`에서 점 하나를 제거합니다.
그래야 루프 후에 `upper`에 `p`를 추가해도 시계 방향 배치의 성질이 유지될 테니까요.

```py
lower = []
for p in reversed(arr):
    while len(lower) >= 2 and ccw(lower[-2], lower[-1], p) >= 0:
        lower.pop()
    lower.append(p)
```

다음으로 아래쪽 껍질 `lower`를 구성합니다.
루프 후 `lower`는 아래쪽 껍질을 이루는 점의 리스트가 되며, 이번엔 *x*좌표 *내림차순*으로 정렬된 형태가 됩니다.

*x*좌표 내림차순으로 정렬된다는 얘기는, `lower`의 모든 점을 순서대로 봤을 때 **모든 점이 시계 방향으로 배치된다**는 걸 의미하기도 합니다.
여기서 주의해야 하는 게, 정렬 순서는 반대이지만, 시계 방향이라는 점은 `upper`와 동일합니다. 이는 정렬이 반대지만 덮는 방향도 반대여서 그렇습니다.

루프 안에서는, `lower`에 `p`를 추가해도 시계 방향이 유지될 수 있도록 점을 제거한 후 `p`를 추가하게 됩니다. `upper`에서 한 것과 동일합니다.

```py
return upper[:-1] + lower[:-1]
```
마지막으로 두 껍질을 합쳐서 반환합니다. 이때, `upper`의 마지막 점은 `lower`의 첫 번째 점이고, `lower`의 마지막 점은 `upper`의 첫 번째 점이므로, 이들을 제외하기 위해 두 리스트 모두 `[:-1]`을 한 다음 합칩니다.

그러면 끝입니다. 그렇게 반환된 리스트는 컨벡스 헐 위의 모든 점을 시계 방향으로 정렬한 리스트가 됩니다.

모노톤 체인 알고리즘과 함께라면 컨벡스 헐이 쉬워집니다. 각도 정렬처럼 실수 오차 때문에 머리 썩히고 할 것 없습니다. 그냥 위에 덮고 아래 덮으면 끝입니다.


## 알아둘 점

만약 컨벡스 헐을 구할 때, 한 변 위에 여러 점이 있는 경우 그 점도 모두 포함하고 싶다면, `upper`와 `lower`를 구하는 루프를 아래와 같이 고치면 됩니다.
```py
while len(upper) >= 2 and ccw(upper[-2], upper[-1], p) > 0:
```
두 번째 조건식의 부등호를 `>=`에서 `>`로 바꾸면 됩니다.

그리고, 위에 제시한 구현체는 `arr` 안에 같은 점이 중복해서 들어가 있으면 잘못된 결과를 냅니다. 사용 시 유의해 주세요.


# 예제

## <a href="https://boj.kr/1708">BOJ 1708 - 볼록 껍질</a>

```py
# 여기에 빠른 입출력 삽입...
# 여기에 모노톤 체인 컨벡스 헐 코드 삽입...
n = int(input())
arr = [tuple(map(int, input().split())) for _ in range(n)]
ans = cvh(arr)
print(len(ans))
```

## <a href="https://boj.kr/3679">BOJ 3679 - 단순 다각형</a>

위쪽 껍질을 먼저 덮어주고, 위쪽 껍질에 포함되지 않은 모든 점을 좌표 내림차순으로 이어 붙입니다. 그러면 단순 다각형이 됩니다.

단, 위쪽 껍질에서 한 변 위에 점이 여러 개 있는 경우 그 모든 점도 다 포함해서 넣어줘야 합니다. 즉, 위에서 언급한 부등호에서 등호를 빼는 것을 해줘야 합니다.
