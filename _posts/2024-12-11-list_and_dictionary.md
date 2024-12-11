---
title: "[Python] List vs. Dictionary (feat. 'in' operator)"
author: Johin
date: 2024-12-11 15:00:00 +0900
categories: [Programming, Python]
tags: [python, tip]
mermaid: true
---

## 들어가기 전에

오늘은 간단하지만 효과적인 꿀팁을 한 번 살펴보려고 한다. 

python에서 굉장히 편리한 도구중 하나가 `in` operator 이다. `x in y` 형태의 구문으로 사용되며, x라는 아이템이 y라는 리스트안에 있는지 없는지를 검사하는 도구이다. **주의!** 오늘 살펴볼 꿀팁이 `in` operator가 아니다. 

결론만 먼저 스포하고 가자!
> list를 dictionary로 포장하면 검색속도가 기가막히게 빨라진다!
{: .prompt-info }

바이오인포매틱스 기준으로 `in` operator 를 쓰는 상황은 다음과 같은 상황들이 있을 수 있다.

* Target accession list에 내가 원하는 genome이 있는지 없는지 검사할 때
* Filter out 할 sequence ID를 확보하고, 해당 list에 없는 sequence ID를 확보할 때
* Contamination species 목록을 확보하고, 해당 목록에 있는 species를 걸러낼 때

그런데, 아무생각없이 편하게 썼던 코드가 굉장히 시간을 오래 잡아먹을 수 있다는걸 알고있을까?

## Linear search

우리가 자주 쓸 것같은 상황을 한 번 살펴보자.

```python
def filtering_out(data):
	outliers = []
	for datum in data:
		if /* some criteria */:
			outlier.append(datum)
	
	return outliers
	
data = /* some data */
outliers = filtering_out(data)
remains = [datum for datum in data if datum not in outliers]
```

대부분 주로 `outliers`와 같은 것들을 `list`로 관리한다. 그런데, python에서 `list`라는 자료구조는 내부의 데이터가 전혀 정렬되어있지 않다. 다른 말로 하자면 특별한 검색전략이 없기 때문에 `list` 안에 어떤 데이터가 있는지 확인하려면 순차적으로 **모든** 아이템을 확인할 수 밖에 없다.

이러한 검색 방법을 **Linear search**라고 부르는데, 굳이 코드로 보자면 다음과 같은 느낌이다.

```python
def linear_search(query, _list):
	for data in _list:
		if query == data:
			return True
	return False
```

계산 속도는 워낙 빠르기 때문에 대부분의 코드에서는 신경 안 써도 될 수준이지만, 이게 엄청나게 시간을 잡아먹는 time hog가 될 때가 있다. 다음의 상황을 생각해보자.

* outlier의 개수가 **10만**개
* query sequence ID의 개수가 **300만**개

 **Linear search**가 최악의 성능을 보여줄 때는, 검색한 데이터가 리스트 안에 없을 때이다. 만약 리스트안에 있다면 그 즉시 검색이 종료되지만, 리스트 안에 어떤 데이터가 없다는 것을 판단하기 위해선 해당 리스트를 모두 훑어보아야만 한다.
 
 위에서 가정한 상황에서라면, 최악의 경우 300만 x 10만 = 3,000억 번의 비교를 수행해야 검색이 종료된다는 것이다.

## Measuring running time

백문불여일견! 직접 한 번 재봅시다!

```python
import time
import random
import string

def generate_data(n_queries, n_outliers):
	def rand_str(slen=15):
		ret = ""
		for i in range(slen):
			ret += random.choice(string.ascii_uppercase)
		return ret
		
	queries = [rand_str() for i in range(n_queries)]
	outliers = random.sample(queries, n_outliers)
	
	return queries, outliers
	
queries, outliers = generat_data(3000000, 100000)

stime = time.time()
remains = [x for x in queries if x not in outliers]
etime = time.time()

print(f"{etime - stime:.2f} secs")
```

랜덤한 15길이의 문자열 300만개를 만들고, 그 중 10만개를 outlier로 선택했다. 그리고 300만개중 outlier에 속하지 않는 290만개를 걸러내는 작업을 지시했다. 과연 몇초나 걸릴까??

> 정답
> 10시간
{: .prompt-info }

물론 컴퓨터 스펙마다 차이가 좀 있겠지만, 생각했던 것 보다 상상할 수 없을 정도로 엄청나게 오래걸린 다는 것을 알 수 있다. 못 믿겠다면 직접 복붙해서 돌려보자. outlier가 몇 개 안 될땐 상관없지만, outlier와 query의 개수의 곱에 비례해서 수행시간이 증가하는 구조이다.

**NOTE**: 혹시 이상하게 내 스크립트가 오래걸리는 것 같다면, 이런 코드는 없는지 한 번 돌아보자!

## Dictionary

python의 기본 자료구조중 `dictionary`가 있다. `key`와 `value`의 쌍으로 이루어진 이 자료구조는 우리가 사전에서 단어의 뜻을 찾을 때처럼 `key (단어)`에 대응되는 `value (뜻)`을 저장하고 간편하게 접근할 수 있게 해주는 자료구조이다. 그렇다면 이게 왜? 여기서 등장할까??

`dictionary`의 핵심 동작 중 하나는 입력으로 들어온 `key`가 `dictionary`내에 있는지 없는지를 검사하고, 없다면 오류를 띄우는 것이다. 다른말로, 이 검색의 성능이 `dictionary`라는 자료구조의 성능에 큰 영향을 미친다는 것이다. 또 `key`가 입력된 순서는 상관 없기 때문에, `dictionary`에서는 `key`의 검색을 효율적으로 할 수 있게 관리하고 검색한다.

`dictionary`에서는 이 `key`를 `hash function`을 이용해 관리하기 때문에 `list`에 비해서 처리속도가 훨씬 빠르다. (대신 추가적인 value 값을 저장하는 메모리를 더 써야하긴 하다.)

앞에서 소개한 코드에서 **outliers**를 `dictionary`로 포장하는 간단한 트릭만으로 검색 속도를 엄청나게 높일 수 있다.

```python
import time
import random
import string

def generate_data(n_queries, n_outliers):
	def rand_str(slen=15):
		ret = ""
		for i in range(slen):
			ret += random.choice(string.ascii_uppercase)
		return ret
		
	queries = [rand_str() for i in range(n_queries)]
	outliers = random.sample(queries, n_outliers)
	
	return queries, outliers
	
def pack_to_dict(_list):
	return {x:True for x in _list}
	
queries, outliers = generat_data(3000000, 100000)
outliers = pack_to_dict(outliers)

stime = time.time()
remains = [x for x in queries if x not in outliers]
etime = time.time()

print(f"{etime - stime:.2f} secs")
```

추가된 코드는 단 3줄이다. 검색하는 구문은 굳이 건들 필요가 없다, 동일한 역할을 수행하기 때문이다. `x in y` 구문에서 **y**에 `list`대신 `dictionary`가 들어올 경우 **x**라는 `key`가 `dictionary`에 있는지를 검사하게된다. 이 구문을 이용하여 위 트릭을 완성할 수 있었던 것이다.

**NOTE**: 여기서 dictionary로 포장할 때, 어차피 사용하지 않은 value값은 용량을 덜 차지하기위해 boolean값으로 세팅했다.

그렇다면 수행시간은?

> 정답
> 0.78초
{: .prompt-info }

10시간 걸리던 코드가 1초도 안 걸리게 바뀌었다. 단 3줄의 힘이다. 이게 바로 효율성의 힘이다.

## np.array, pd.Series

여기서 끝내기 조금 아쉬우니 실험을 몇개만 더 해보자. `list`와 `dictionary`는 python의 기본자료형이지만 우리는 자주 활용하는 패키지들이 많다. **NumPy** 패키지의 `array`, **Pandas** 패키지의 `Series`가 `list`의 역할을 대신할 수 있다. 그리고 많은 경우 코드를 짜다보면 데이터가 `array` 또는 `Series`로 패킹되어있을 때가 많을 것이다.

그렇다면 `list`대신 `array`거나 `Series`인 경우는 속도가 어떻게 될까?

```python
import numpy as np
import pandas as pd

outliers_list = outliers.copy() # list
outliers_dict = pack_to_dict(outliers) # dictionary
outliers_arr = np.array(outliers) # np.array
outliers_ser = pd.Series(index=outliers) # pd.Series
```

이렇게 4개의 자료형을 놓고 테스트를 한다면?

| list | dictionary |       array | Series |
|---:|----------:|----------:|------:|
| 10h | 0.78s      | 15m 36s   | 6.61s  |

확실히 `list`보다는 효율적으로 데이터가 관리되는 것 같지만, `dictionary`만은 못한 것 같다.

**NOTE**: pd.Series의 경우 그냥 x in y 구문을 쓰면 Series의 index에 대한 검사가 수행되기 때문에, Series로 변환할 때 values대신 index로 넘겨줘야한다.

## 나가며

**"돌아가는 코드를 짜는 것"**과 **"효율적인 코드를 짜는 것"**은 다른 차원의 문제이다. 돌아가는 코드를 먼저 짤 줄 알아야겠지만, 그 다음으로 지향해야하는 곳은 효율적인 코드를 짜는 것이다. 결국 시간과 전기세 모두 자원이기 때문에 ㅠ

심지어 오늘 본 예시처럼 극단적으로 효율성이 차이나는 경우가 드물지 않다. 돌아가게만 만들어놓고 **"왜 오래걸릴까?"**라는 질문을 해보지 않는다면 3류에 남을 수 밖에 없다고 생각한다.

**Linear search**와 **Hash**의 차이는 컴공 학부 2학년 정도에 모두 배우는 개념이다. 배운 내용들을 내 작업에 연결시킬 수 있느냐? 하는 부분은 개인의 역량일 것이다.

