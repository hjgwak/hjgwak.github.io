---
title: "[Python] Pandas에서 자주하는 실수 25가지 (1/5)"
author: Johin
date: 2023-02-13 14:00:00 +0900
categories: [Programming, Python]
tags: [python, basic, pandas]
---

파이썬 프로그래밍을 할 때 거의 묻지도 따지지도 않고 **import**하는 패키지가 2개정도 있다.

* pandas
* numpy

pandas는 데이터를 table 형태로 저장하고 처리하는데 특화되어있는 패키지인데, 대부분의 Database가 relational database, 즉 table 형태로 데이터를 표현하기 때문에 반드시 알아두어야 할 패키지라고 할 수 있다.
나도 예전에 table을 처리하는 함수들을 직접 만들어두고 쓰다가 pandas의 존재를 안 이후 모두 용도폐기시켜버렸다 ㅋㅋㅋ

pandas의 데이터 구조는 object의 list인 `Series`와 Series로 구성되어있는 `DataFrame`으로 구성된다. R에서 `DataFrame`과 거의 동일하다. (할 수 있는 일 까지도)

`DataFrame` class는 굉장히 많은 편의성을 제공하는데, 같은 동작도 여러가지 문법, 함수를 통해 수행할 수 있다. 개인적인 습관이 반영되는 부분이긴 하지만, 이번 포스팅에서는 같은 동작도 조금 더 효율적으로 처리할 수 있는 방법에 대해 알아보고자한다.

> 데이터가 적을 땐 큰 상관이 없지만, 데이터의 크기가 커질수록 이 차이는 엄청난 수행시간의 차이를 가져온다.
{: .prompt-tip}

## 1. CSV로 저장할 때 쓸모없는 column(index) 저장

```python
import pandas as pd
df = pd.read_csv('dummy.csv')
df.to_csv('output.csv')

df = pd.read_csv('output.csv')
df.head()
```
**DataFrame**을 손쉽게 저장할 수 있는 함수 중 하나는 `to_csv()`함수이다. 만약 `dummy.csv` 파일의 내용이 아래처럼 생겼다고 하고 위 코드를 실행시키면 어떤 결과가 나올까?

| Year  | Time  | Athlete  | Place  |
|-----:|------:|:--------|:-------|
| 1972 | 10.07 | Valeriy Borzov (URS) | Munich |
| 1973 | 10.15 | Steve Williams (USA) | Dakar |
| ... | ... | ... | ... |

바로 아래와 같은 결과가 나온다

| Unnamed:0  | Year  | Time  | Athlete  | Place  |
|-------------:|-----:|------:|:--------|:-------|
| 0 | 1972 | 10.07 | Valeriy Borzov (URS) | Munich |
| 1 | 1973 | 10.15 | Steve Williams (USA) | Dakar |
| ... | ... | ... | ... | ... |

<span style="color: red">**맨 앞에 이상한 열이 추가됐다!**</span>

`to_csv`함수에는 **DataFrame**의 index와 header를 저장할지 여부를 전달하는 `index`, `header`옵션이 있는데, 이 두 parameter의 기본값이 `True`이다. 따라서 아무 의미없는 index도 함께 저장되는 것이다. 이렇게 의미없는 index를 저장하지 않기 위해서는 다음과같이 코드를 수정해야한다.

```python
df.to_csv('output.csv', index=False)
``` 
다음과 같이 index를 같이 저장하고 읽어들일 때, 첫번째 열이 index임을 알려주는 방법도 있지만, csv파일의 용량이 커져서 추천하진 않는다

```python
df.to_csv('output.csv')
df = pd.read_csv('output.csv', index_col=[0])
```

## 2. column name에 공백 (space) 넣기

DataFrame의 각 열이 어떤 값을 가지고있는지 알려주기위해서 header (column names)를 지정해준다. 그리고 이 때 공백을 넣어서 지정할 수도 있는데, 사람 눈으로 보기엔 좋지만, pandas에서 제공하는 몇몇 편의기능을 사용하지 못하게하는 단점이 있다. 아래 예시를 보자

```python
import pandas as pd
df = pd.read_csv('dummy.csv')
df['First Initial'] = df['Athlete'].str[:1]
df.head()
```

| Year  | Time  | Athlete  | Place  | First Initial  |
|-----:|------:|:--------|:-------|------------:|
| 1972 | 10.07 | Valeriy Borzov (URS) | Munich | V |
| 1973 | 10.15 | Steve Williams (USA) | Dakar | S |
| ... | ... | ... | ... | ... |

이름의 첫 글자를 저장하는 `First Initial` column이 추가되었다. 이렇게 column name에 공백이 들어있을때 발생할 수 있는 문제는 대표적으로 다음 두가지가 있다.

### 1) dot syntax를 사용할 수 없음

**DataFrame**은 특정 column에 접근할 때 dot(.)을 이용해 접근할 수 있는 문법을 제공한다. 예를들면 다음과 같다

```python
df.Year
df.Time
``` 
하지만, column name에 공백이 있으면 이렇게 사용할 수 없다. (Interpreter가 구분할 수 없기 때문에)

### 2) .query() method를 사용할 때 불편함

또, **DataFrame**은 `.apply()` method와 비슷하게 `.query()` method도 제공한다. 이 때도 공백이 있으면 제대로 작동하지 않는다.

때문에 추천하는 방법은 공백 대신에 under bar(_)를 삽입하는 것이다. 그렇게 했을 때 위 두 단점을 극복할 수 있다. (보기에 따라 언더바가 거슬릴 수는 있다.)

```python
df = pd.read_csv('dummy.csv')
df['First_Initial'] = df['Athlete'].str[:1]

df.First_Initial
df.query('First_Initial == "S"')
```

> 물론 위 두 상황은 다른 문법으로 대체하는 것이 가능하다. 예를들어  
> df.First\_Initial 대신 df['First Initial']  
> df.query('First\_Initial == "S"') 대신 df.loc[df['First Initial'] == "S"]  
> 이렇게 하면 공백을 column name에 넣어도 전혀 상관이 없다.  
> 하지만 .query method를 활용하는 방법도 많고,  
> dot syntax나 .query method를 사용한 패키지가 있다면 오류가 날 수 있다.
{: .prompt-tip}

## 3. query() method를 활용 안 함

나도 이번에 처음 안 사실인데, **DataFrame**은 `.query()`라는 강력하고 편한 filtering method를 지원함. pandas를 처음 배울 때, 내 **DataFrame**에서 특정 데이터만 가져오기위해 다음과 같은 문법을 배울 것이다. (구글링의 많은 결과도 다음과 같은 문법을 알려준다. <span style="color: purple">ChatGPT</span> 역시)

```python
import pandas as pd
df = pd.read_csv('dummy.csv')
df.loc[(df['Year'] < 1980) & (df['Time'] > 10)]
```

나도 대부분의 코드에서 위와같은 필터링 방식을 사용해왔다. 위 동작이 어떻게 이루어지는지 꼼꼼하게 나누어 보자면 다음과 같다.

1. `df['Year'] < 1980, df['Time'] > 10` 으로 **Boolean Series**를 얻음
2. `(...) & (...)` 으로 **Boolean Series**를 얻음
3. `df.loc[]` method로 **True**에 해당하는 row를 가져옴

위 방법은 아무런 문제가 없다. 저렇게 쓴다고 예상치 않은 결과가 나오거나 속도가 눈에띄게 느려진다거나 하진 않는다. 그러나 필터링할 값이 많아지면 조건문을 쓰는게 굉장히 까다롭고 복잡해진다. ㅠㅠ

비교문마다 소괄호를 씌워줘야 하고, & 오퍼레이션으로 엮어줘야하고, **DataFrame**의 이름이 df보다 더 길 경우 저 문장은 기약없이 길어지게 된다. (column 하나를 언급할 때마다 **DataFrame** 이름을 한 번씩 써줘야 한다...!!)

하지만 `.query()` method를 활용하면 다음처럼 굉장히 간소화 할 수 있다.

```python
df.query('Year < 1980 and Time > 10')
```

## 4. query method를 사용할 때 string operation 사용

3번에서 살펴본 것 처럼 `.query()` method는 입력으로 string이 들어간다. 위 예시처럼 target column과 값이 고정된 경우는 상관 없겠지만, 많은 경우 column 또는 값을 유저의 입력을 받아서 필터링하게 된다. 이런 상황에서 직관적으로 떠올리는 해법은 무엇일까?

string operation이나 format string을 활용하는 방법일 것이다.

```python
import pandas as pd
df = pd.read_csv('dummy.csv')
min_year = 1980
min_time = 10

df = df.query('Year < ' + str(min_year) + ' and Time > ' + str(min_time))
df = df.query(f'Year < {min_year} and Time > {min_time}')
```

이것도 별다른 문제가 될 것 없는 코드이긴하다. 하지만 이역시 훨씬 간단하게 쓸 수 있다!!!

```python
df = df.query('Year < @min_year and Time > @min_time')
```

`.query()` method는 외부의 변수를 `@`을 이용하여 참조할 수 있는 기능을 제공한다! 코드가 훨씬 깔꼼하다 :)

## 5. inplace=True 사용

생각보다 많은 오류를 발생시키는 부분이기도하다. 코드를 읽을 때 헷갈리기도 하고, 처음엔 편리해보이지만 쓰다보면 저절로 기피하게되는 옵션이기도 하다. 

```python
import pandas as pd
df = pd.read_csv('dummy.csv')
df.fillna(0, inplace=True)
df.reset_index(inplace=True)
```
**DataFrame**의 변경내용을 원본에 바로 반영시켜버리기 때문에 수 많은 <span style="color:red">warning</span> 메세지의 원인이 되기도 하고 오류를 일으키기도 한다. 실제로 개발자들은 이 옵션을 없앨 계획까지 가지고 있다고도 한다.

```python
df = pd.read_csv('dummy.csv')
df = df.fillna(0)
df = df.reset_index()
```
`inplace=True`옵션 보다는 위처럼 명시적으로 덮어쓰기를 호출해주는게 읽기에도 좋고 훨씬 오류를 감소시켜준다.


이 포스트는 아래 유튜브를 참고하여 작성되었습니다.

{% include embed/youtube.html id='_gaAoJBMJ_Q' %}