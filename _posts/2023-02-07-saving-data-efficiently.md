---
title: "[Python] 효율적으로 데이터 저장하기"
author: Johin
date: 2023-02-07 20:00:00 +0900
categories: [Programming, Python]
tags: [python, basic, pickle, parquet, feather]
---

파이썬으로 데이터분석을 진행할 때, 생각보다 많은 시간을 잡아먹는 구간이 있다.
바로 `데이터 입출력` 이다. 대부분 별로 중요하지 않게 생각하는데 (내가 주로 그렇다) 실질적인 러닝타임에 영향을 주기도 하고, storage의 압박에 시달리기 시작한다면 한 번쯤 써볼만한 내용을 정리해보려고 한다.

앞으로 서술할 내용을 사용하기위해 필요한 패키지는 다음과 같다.

* pandas
* numpy
* pyarrow

> fastaparquet 패키지를 설치하면 parquet은 사용할 수 있지만, feather를 사용하지 못한다.  
> 또, 내 실험 환경에서는 fastparquet 패키지보다 pyarrow 패키지의 수행속도가 훨씬 빨랐다.  
> 그러나 pyarrow 패키지의 경우 float16 (halffloat)을 저장하지 못하는 단점이 있다.
{: .prompt-warning }

실험에 사용한 컴퓨터 스펙은 다음과 같다.

* iMac (M1, 2021)
* Chip: Apple M1
* Memory: 8 GB
* macOS: Ventura 13.1

## Fake Dataset
```python
import pandas as pd
import numpy as np

def get_dataset(size):
    # Create Fake Dataset
    df = pd.DataFrame()
    df['size'] = np.random.choice(['big', 'medium', 'small'], size)
    df['age'] = np.random.randint(1, 50, size)
    df['team'] = np.random.choice(['red', 'blue', 'yellow', 'green'], size)
    df['win'] = np.random.choice(['yes', 'no'], size)
    dates = pd.date_range('2020-01-01', '2020-12-31')
    df['date'] = np.random.choice(dates, size)
    df['prob'] = np.random.uniform(0, 1, size)
    
    return df

def set_dtypes(df):
    df['size'] = df['size'].astype('category')
    df['team'] = df['team'].astype('category')
    df['age'] = df['age'].astype('int16')
    df['win'] = df['win'].map({'yes': True, 'no': False})
    df['prob'] = df['prob'].astype('float32')
    
    return df
```

우선 간단한 실험을 위해 사용할 fake dataset을 생성하는 함수들이다. 각 열별로 특별한 의미는 생각하지 말자.

> DataFrame 에서 type을 설정해주는 것이 생각보다 효율면에서 도움이 된다.  
> raw_data = get_dataset(1_000_000)   // 45.8 MB  
> typed_data = set_dtypes(raw_data)   // 22.9 MB
{: .prompt-tip }

## CSV vs. Pickle vs. Parquet vs. Feather

파이썬에서, 특히 Pandas 에서, 데이터를 쉽게 저장하고 불러올 수 있는 방법 중 `csv, pickle, parquet, feather` 4가지를 살펴보려고 한다.

결론부터 간단히 말하자면, 속도적인 측면에서 **pickle**이 가장 빠르고 공간적인 측면에서 **parquet**가 가장 적게 차지한다. **feather**는 가장 밸런스가 좋은 포지션에 있다.

어떤 포맷을 선택할지는 완전히 취향의 문제인 듯 하다.

### dtype을 설정 안 했을 때

| format  | write   | load   | storage   |
| :-------|-------:|------:|---------:|
| csv       | 6.63s ± 101ms | 442ms ± 18.1ms | 46 MB |
| pickle   | 662ms ± 19ms | 273ms ± 11.8ms | 43 MB |
| parquet | 223ms ± 9.07ms | 79.8ms ± 3.24ms | 10 MB |
| feather | 107ms ± 4.82ms | 67.4ms ± 2.01ms | 29 MB | 

### dtype을 설정 했을 때

| format  | write   | load   | storage   |
| :-------|-------:|------:|---------:|
| csv       | 6.63s ± 101ms | 442ms ± 18.1ms | 46 MB |
| pickle   | 31.5ms ± 1ms | 48ms ± 1ms | 24 MB |
| parquet | 80.8ms ± 0.1ms | 13.9ms ± 0.2ms | 4 MB |
| feather | 43.9ms ± 0.3ms | 15.4ms ± 0.2ms | 8.9 MB | 

`pickle`의 경우는 메모리에 올라온 용량과 거의 같은 용량의 파일로 저장하는 것을 볼 수 있다. `parquet`와 `feather`의 경우 조금 더 압축된 형태로 데이터를 저장하고 parquet 포맷이 가장 적은 용량을 차지한다.

dtype을 지정한 경우 `pickle`이 가장 빠른 속도를 보여주지만, 그렇지 않은 경우 `feather`가 가장 빠른 속도를 보여준다. dtype을 지정하지 않았을 때 물리적으로 읽어들일 용량이 많기 때문에 당연한 결과로 보여진다.  
_2배정도 용량이 차이나는데 쓰는시간이 20배정도 차이가 나는 것이 좀 신기하긴 하다_  
`csv`로 저장했을 때 dtype 정보가 전혀 저장되지 않는다.

## Sequence dataset의 경우

내가 실제 프로젝트에서 사용하는 데이터의 경우 어떻게 달라지는지 확인해보고자 한다.  
예시로 가져온 데이터셋은 DeepLoc-2.0 에서 Train/Validation 용으로 쓰인 데이터셋이다.

* 28,303 seqs
* accession number, kingdom, 10 locations, sequence

accession number, sequence 열의 경우 *object* 타입으로  
kingdom, partition 열의 경우 *category* 타입으로  
나머지 location의 열들의 경우 *int16* 타입으로 지정했다.

| format  | write   | load   | storage   |
| :-------|-------:|------:|---------:|
| csv       | 295ms ± 9.73ms | 167ms ± 0.9ms | 16 MB |
| pickle   | 10.1ms ± 0.1ms | 12.2ms ± 0.2ms | 16 MB |
| parquet | 31.3ms ± 1.82ms | 22.9ms ± 2.93ms | 15 MB |
| feather | 20.2ms ± 0.7ms | 16.4ms ± 0.1ms | 14 MB | 

> Sequence 데이터셋의 경우 label 등 metadata 보다  
> 서열 자체가 차지하는 용량이 크기 때문에 포맷별로 큰 용량차이가 나지 않는 것 같다  
> 결국 속도 빠른 pickle이 짱짱맨
{: .prompt-tip }

이 포스트는 아래 유튜브를 참고하여 작성되었습니다.

{% include embed/youtube.html id='u4rsA5ZiTls' %}