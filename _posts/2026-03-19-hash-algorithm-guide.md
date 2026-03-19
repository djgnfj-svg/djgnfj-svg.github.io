---
title: 해시(Hash) 알고리즘 정리 — Python 풀이 중심
date: 2026-03-19 00:00:00 +0900
categories: [Algorithm]
tags: [hash, algorithm, python, coding-test]
---

코딩 테스트에서 해시는 빠지지 않는다. dict 하나로 O(n²)을 O(n)으로 바꿀 수 있기 때문이다. 패턴별로 정리한다.


# 해시의 핵심

해시 = **키로 값을 O(1)에 찾는 것**. Python에서는 `dict`와 `set`이 해시 테이블이다.

```python
# 리스트에서 찾기: O(n)
if x in my_list:  # 전부 순회

# set에서 찾기: O(1)
if x in my_set:   # 해시로 바로 접근
```

이 차이 하나로 시간 초과가 갈린다.


# 패턴 1: 빈도 세기 (Counter)

가장 기본적인 해시 활용이다. 원소의 등장 횟수를 센다.

## 예제: 완주하지 못한 선수

> 마라톤에 참여한 선수 목록과 완주한 선수 목록이 주어진다. 완주하지 못한 선수를 구하라.

```python
from collections import Counter

def solution(participant, completion):
    p = Counter(participant)
    c = Counter(completion)
    answer = p - c  # Counter끼리 뺄셈 가능
    return list(answer.keys())[0]
```

Counter 없이 직접 구현하면:

```python
def solution(participant, completion):
    d = {}
    for name in participant:
        d[name] = d.get(name, 0) + 1
    for name in completion:
        d[name] -= 1
    for name, count in d.items():
        if count > 0:
            return name
```

시간복잡도: O(n). 정렬(O(n log n))보다 빠르다.

## 예제: 최빈값 구하기

```python
from collections import Counter

def most_frequent(nums):
    count = Counter(nums)
    return count.most_common(1)[0][0]

# most_common(k): 상위 k개를 (값, 횟수) 튜플로 반환
```

## 예제: 문자열에서 첫 번째 고유 문자

```python
def first_unique_char(s):
    count = Counter(s)
    for i, c in enumerate(s):
        if count[c] == 1:
            return i
    return -1
```


# 패턴 2: 존재 여부 확인 (Set)

"이 값이 있는가?"만 판단하면 되면 set을 쓴다.

## 예제: 두 수의 합 (Two Sum)

> 배열에서 두 수의 합이 target이 되는 인덱스 쌍을 구하라.

```python
def two_sum(nums, target):
    seen = {}  # 값 → 인덱스
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
```

브루트포스 O(n²) → 해시로 O(n). 코딩 테스트의 고전이다.

## 예제: 중복 원소 존재 여부

```python
def contains_duplicate(nums):
    return len(nums) != len(set(nums))
```

한 줄이면 끝난다.

## 예제: 교집합 구하기

```python
def intersection(nums1, nums2):
    return list(set(nums1) & set(nums2))

# 교집합(&), 합집합(|), 차집합(-), 대칭차집합(^)
```


# 패턴 3: 그룹핑 (defaultdict)

같은 특성을 가진 원소끼리 묶을 때 쓴다.

## 예제: 아나그램 그룹핑

> 문자열 배열에서 아나그램끼리 묶어라.
> `["eat","tea","tan","ate","nat","bat"]` → `[["eat","tea","ate"],["tan","nat"],["bat"]]`

```python
from collections import defaultdict

def group_anagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))  # 정렬하면 아나그램은 같은 키
        groups[key].append(s)
    return list(groups.values())
```

정렬 대신 카운트 배열을 키로 쓰면 O(n·k)로 더 빠르다:

```python
def group_anagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        count = [0] * 26
        for c in s:
            count[ord(c) - ord('a')] += 1
        groups[tuple(count)].append(s)
    return list(groups.values())
```

## 예제: 전화번호 목록

> 접두어 관계인 번호가 있으면 False를 반환하라.

```python
def solution(phone_book):
    phone_set = set(phone_book)
    for phone in phone_book:
        for i in range(1, len(phone)):
            if phone[:i] in phone_set:
                return False
    return True
```


# 패턴 4: 슬라이딩 윈도우 + 해시

연속 구간에서 조건을 만족하는 경우를 찾을 때 해시와 슬라이딩 윈도우를 결합한다.

## 예제: 중복 없는 가장 긴 부분 문자열

```python
def length_of_longest_substring(s):
    seen = {}  # 문자 → 마지막 등장 인덱스
    left = 0
    max_len = 0

    for right, c in enumerate(s):
        if c in seen and seen[c] >= left:
            left = seen[c] + 1  # 중복 문자 다음으로 이동
        seen[c] = right
        max_len = max(max_len, right - left + 1)

    return max_len
```

## 예제: 길이 K인 부분 문자열의 고유 문자 수

```python
from collections import Counter

def unique_chars_in_substrings(s, k):
    if len(s) < k:
        return []

    window = Counter(s[:k])
    result = [len(window)]

    for i in range(k, len(s)):
        # 오른쪽 추가
        window[s[i]] += 1
        # 왼쪽 제거
        window[s[i - k]] -= 1
        if window[s[i - k]] == 0:
            del window[s[i - k]]
        result.append(len(window))

    return result
```


# 패턴 5: 누적합 + 해시 (Prefix Sum)

구간 합 문제에서 해시를 쓰면 O(n)에 풀 수 있다.

## 예제: 합이 K인 부분 배열의 개수

```python
def subarray_sum(nums, k):
    prefix_count = {0: 1}  # 누적합 → 등장 횟수
    curr_sum = 0
    result = 0

    for num in nums:
        curr_sum += num
        # curr_sum - k가 이전에 나온 적 있으면,
        # 그 지점부터 현재까지의 합이 k
        result += prefix_count.get(curr_sum - k, 0)
        prefix_count[curr_sum] = prefix_count.get(curr_sum, 0) + 1

    return result
```

핵심 아이디어: `sum(i..j) = prefix[j] - prefix[i]`이므로, `prefix[j] - k = prefix[i]`인 i가 몇 개인지를 해시로 O(1)에 찾는다.

## 예제: 합이 0인 가장 긴 부분 배열

```python
def longest_subarray_sum_zero(nums):
    prefix_map = {0: -1}  # 누적합 → 처음 등장한 인덱스
    curr_sum = 0
    max_len = 0

    for i, num in enumerate(nums):
        curr_sum += num
        if curr_sum in prefix_map:
            max_len = max(max_len, i - prefix_map[curr_sum])
        else:
            prefix_map[curr_sum] = i

    return max_len
```


# 패턴 6: 해시로 그래프/관계 표현

## 예제: 위장 (조합 문제)

> 의상 종류별로 하나씩 선택하거나 안 선택할 수 있다. 최소 1개는 입어야 한다. 가능한 조합 수를 구하라.

```python
from collections import Counter

def solution(clothes):
    type_count = Counter(t for _, t in clothes)
    result = 1
    for count in type_count.values():
        result *= (count + 1)  # 안 입는 경우 +1
    return result - 1  # 아무것도 안 입는 경우 -1
```

## 예제: 베스트앨범

> 장르별 재생 횟수가 높은 순으로 최대 2곡씩 선택하라.

```python
from collections import defaultdict

def solution(genres, plays):
    genre_plays = defaultdict(int)
    genre_songs = defaultdict(list)

    for i, (g, p) in enumerate(zip(genres, plays)):
        genre_plays[g] += p
        genre_songs[g].append((p, i))

    result = []
    # 총 재생 수가 많은 장르부터
    for genre in sorted(genre_plays, key=genre_plays.get, reverse=True):
        # 재생 수 내림차순, 같으면 인덱스 오름차순
        songs = sorted(genre_songs[genre], key=lambda x: (-x[0], x[1]))
        result.extend(idx for _, idx in songs[:2])

    return result
```


# Python 해시 관련 팁

## dict/set에 넣을 수 있는 것 (hashable)

```python
# hashable (키로 사용 가능)
d = {}
d[42] = "int"
d["hello"] = "str"
d[(1, 2)] = "tuple"
d[frozenset([1, 2])] = "frozenset"

# unhashable (키로 사용 불가)
d[[1, 2]] = "list"       # TypeError
d[{1, 2}] = "set"        # TypeError
d[{"a": 1}] = "dict"     # TypeError
```

리스트를 키로 쓰고 싶으면 `tuple()`로 변환하라.

## defaultdict vs Counter vs dict.get

```python
# 1. dict.get — 가장 기본
count = {}
count[x] = count.get(x, 0) + 1

# 2. defaultdict — 기본값 자동 생성
from collections import defaultdict
count = defaultdict(int)
count[x] += 1

# 3. Counter — 빈도 세기 특화
from collections import Counter
count = Counter(nums)
count.most_common(3)  # 상위 3개
count1 - count2       # 차집합
count1 & count2       # 교집합 (최솟값)
count1 | count2       # 합집합 (최댓값)
```

## OrderedDict로 LRU 캐시

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)  # 최근 사용으로 이동
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # 가장 오래된 것 제거
```


# 시간복잡도 정리

| 연산 | list | dict/set | 비고 |
|------|------|----------|------|
| 검색 (`in`) | O(n) | O(1) | 이것만 기억해도 절반은 해결 |
| 삽입 | O(1) 끝 | O(1) | |
| 삭제 | O(n) | O(1) | |
| 순회 | O(n) | O(n) | |

해시 문제의 본질: **"검색을 O(1)로 만들면 전체 복잡도가 줄어드는가?"** 그렇다면 dict/set을 써라.


# 마무리

해시 문제를 만나면 이 순서로 생각하라:

1. **빈도를 세는가?** → Counter
2. **존재 여부를 확인하는가?** → set
3. **키-값 매핑이 필요한가?** → dict
4. **그룹핑이 필요한가?** → defaultdict
5. **구간 합과 관련 있는가?** → 누적합 + dict

대부분의 해시 문제는 이 다섯 패턴 안에 있다.
