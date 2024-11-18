---
title: "[Python] Pickle vs. HDF5"
author: Johin
date: 2024-11-18 13:00:00 +0900
categories: [Programming, Python]
tags: [python, pickle, h5py, dataset]
mermaid: true
---

## 들어가기 전에

이 글은 [이전글](/posts/saving-data-efficiently/)과 주제는 비슷하나, 약간 다른 데이터를 대상으로 실험을 진행해볼 것이다. 이전 글에선 각 열(column)별로 특정 데이터가 들어있는 tabular format의 데이터에 초점이 맞춰졌었다면, 이번에는 임베딩 벡터(embedded vectors)와 같은 고차원 숫자데이터에 초점을 맞춰보려고 한다.

개인적으로, 특정 단백질 서열 데이터베이스에 대해 **ESM-2** 또는 **ProtT5**와 같은 pLM 모델들을 미리 돌려서 추출된 protein embedding vector를 어떻게 효율적으로 저장하고 로드할 수 있을까? 라는 질문에서 시작되었다.

## Fake Dataset

먼저 이번에도 실험에 사용될 가짜 데이터셋을 만들고 시작하겠다. **ESM-2** 모델 중 **650M**사이즈의 모델을 기준으로 한 데이터당 1,280 dimension을 가지는 데이터를 N개 만들 것이다.

```python
import numpy as np

def create_random_data(n_data, dim=1280):
	embeds = np.random.random((n_data, dim))
	seqids = [f"seq{i:010d}" for i in range(n_data)]
	
	dataset = {sid:embed for sid, embed in zip(seqids, embeds)}
	
	return dataset
	
dataset = create_random_data(10000, dim=1280)
```

나는 주로 데이터셋을 접근하기 편하게 `sequence id`가 `key`가 되고 `embedded vector`가 `value`가 되는 **dictionary**로 관리하기 때문에, 같은 포맷으로 맞춰주었다.

이렇게 만든 1만개 데이터의 크기는 대략 **103.32MB**정도였다.


## Measuring running time

실험에 들어가기 전에, 공평한 시간측정을 위해서 수행시간을 측정하는 함수는 동일한 다음과 같은 함수를 만들어 사용했다.

```python
def measure_time(data, func, n_repeats=10):
	rtimes = []
	for i in range(n_repeats):
		stime = time.time()
		func(data)
		etime = time.time()
		rtimes.append(etime - stime)
		
	return np.mean(rtimes)
```

Context Switch등의 간섭 요소의 영향을 줄여보고자 같은 함수에 대해 10번의 평균을 측정했다.

## Pickle

Pickle package를 이용해 데이터를 쓰고 읽는 코드는 다음과같이 사용했다.

```python
import pickle as pkl

def write_pkl(data):
	with open("tmp.pkl", 'wb') as ostream:
		pkl.dump(data, ostream)
		
def read_pkl(file_name):
	with open(file_name, 'rb') as fstream:
		tmp = pkl.load(fstream)
		
Pw = measure_time(dataset, write_pkl)
Pr = measure_time("tmp.pkl", read_pkl)

print(f"pickle (write): {Pw:.2f} secs")
print(f"pickle (read): {Pr:.2f} secs")
```

미리 만들어둔 1만개의 데이터를 쓰고 읽는데 걸린 시간은 각각 **0.47초** 와 **0.15초**였다. *진짜 빠르다*

저장된 파일 (tmp.pkl)의 용량은 **99 MB**였다. 내 생각에 pickle을 통해서 용량이 압축된 것 같지는 않고, **MB**와 **MiB**의 차이인 것 같다.

## HDF5

HDF5 (Hierarchical Data Format version 5) 포맷은 h5py package를 통해서 손쉽게 사용할 수 있다. 원래는 모델의 weights등을 저장하는 패키지였는데, 어차피 숫자의 벡터인거 dataset도 저장할 수 있어서 h5py.Datasets 이 추가되었다. (사실 맞는 말인지 모른다, 그냥 내가 최근에 알게되었다.)

h5py package를 활용해서 데이터를 쓰고 읽는 코드는 다음과 같다. 우선 처음 써보는 패키지이기 때문에 [ProtTrans](https://github.com/agemagician/ProtTrans/blob/master/Embedding/prott5_embedder.py) 에 있는 코드를 참고해서 작성해보았다.

```python
import h5py

def write_hdf5(data):
	with h5py.File("tmp.h5", 'w') as hf:
		for k, v in data.items():
			hf.create_dataset(k, data=v)
		
def write_hdf5(file_name):
	with h5py.File(file_name, 'r') as hf:
		tmp = {k:hf[k][()] for k in hf.keys()}
		
Hw = measure_time(dataset, write_hdf5)
Hr = measure_time("tmp.h5", read_hdf5)

print(f"HDF5 (write): {Hw:.2f} secs")
print(f"HDF5 (read): {Hr:.2f} secs")
```

미리 만들어둔 1만개의 데이터를 쓰고 읽는데 걸린 시간은 각각 **2.84초** 와 **3.14초**였다. *좀 너무 느리다*

저장된 파일 (tmp.h5)의 용량은 **102 MB**였다. 

이건 너무 이상하다. HDF5를 쓸 필요가 없어보인다. 쓰기는 4배이상 읽기는 15배이상 느리고 그렇다고 저장공간을 획기적으로 줄이지도 않는다. (오히려 더 많이 쓴다) 뭔가 패키지나 데이터포맷 자체의 문제가 아니라, 내가 코드를 비효율적으로 쓴 느낌이 굉장히 많이 났기 때문에 (write_hdf5 함수는 누가봐도 비효율적이어보인다) 새로운 방법을 Document를 읽어가며 찾아봤다.

## HDF5 (fast)

앞서 썼던 코드는 각 벡터마다 create_dataset 함수를 호출하는걸 봤을 때 굉장히 비효율적이었던 것같다. 그래서 서열의 ID와 벡터를 각각 저장하는 형식으로 코드를 수정해봤다.

```python
def write_hdf5_v2(data):
	keys = [k.encode('utf8') for k in list(data.keys())]
	vals = np.array(list(data.values))

	with h5py.File("tmp.h5", 'w') as hf:
		hf.create_dataset("seqids", data=keys)
		hf.create_dataset("embeds", data=vals)
		
def write_hdf5_v2(file_name):
	d = {}
	with h5py.File(file_name, 'r') as hf:
		seqids = hf['seqids'][()]
		embeds = hf['embeds'][()]
		
		d = {k:v for k, v in zip(seqids, embeds)}
		
Hw = measure_time(dataset, write_hdf5_v2)
Hr = measure_time("tmp.h5", read_hdf5_v2)

print(f"HDF5 (write): {Hw:.2f} secs")
print(f"HDF5 (read): {Hr:.2f} secs")
```

미리 만들어둔 1만개의 데이터를 쓰고 읽는데 걸린 시간은 각각 **0.36초** 와 **0.07초**였다. *이게 맞지*

저장된 파일 (tmp.h5)의 용량은 **98 MB**였다. 

기껏 dictionary로 포장해둔 두 array를 다시 array로 바꿔주고, string 데이터의 경우 utf-8로 바꿔주는 귀찮은 과정을 거쳐야하지만, 어쨌든 pickle 과 비슷한 수준의 성능을 관찰했다!

## Various size of datasets

1만개 (100 MB)는 어쩌면 너무 적을 수가 있어서, 조금 더 많은 수의 데이터에 대해서 같은 측정을 반복해보았다. *(개인적으로 Markdown Table 마음에 안 듦 ㅠ)*

|    | # of data | 10,000 | 50,000 | 100,000 | 500,000 |
|--:|---------:|-------:|--------:|---------:|----------:|
|    | memory  | 103.32 MB | 517.72 MB | 1.04 GB | 5.17 GB |
| read | pickle | 0.15 | 0.55 | 1.01 | 5.33 |
|  | HDF5 (each) | 3.14 | 15.93 | 32.82 | 166.08 |
|  | HDF5 (bulk) | 0.07 | 0.34 | 0.62 | 3.14 |
| write | pickle | 0.47 | 4.28 | 7.14 | 36.64 |
|  | HDF5 (each) | 2.84 | 15.09 | 30.83 | 156.10 |
|  | HDF5 (bulk) | 0.36 | 3.25 | 6.14 | 33.53 |
| storage | pickle | 99 MB | 493 MB | 985 MB | 4.9 GB |
|  | HDF5 (each) | 102 MB | 508 MB | 1015 MB | 5.0 GB |
|  | HDF5 (bulk) | 98 MB | 489 MB | 978 MB | 4.8 GB |

사이즈가 커지면 HDF5에서 보여주는 차이가 커질까 싶었지만, 엄청 드라마틱한 차이까지는 없는 것 같다. 살짝 빠르고 살짝 용량 덜 쓰는 정도? 굳이 반드시 HDF5를 써야만 하는 이유는 없는 것 같다.

## HDF5 with compression

포스트를 마치려던 중 **h5py.Dataset** 에서 compression 옵션을 제공한다는 것을 알게되었다!!! 혹시 이게 저장공간을 획기적으로 줄여주지 않을까? 그렇다면 같은 속도에 적은 용량으로 HDF5를 쓸 이유를 찾을 수 있지 않을까해서 바로 실험해보았다.

```python
def write_hdf5_comp(data):
	keys = [k.encode('utf8') for k in list(data.keys())]
	vals = np.array(list(data.values))

	with h5py.File("tmp.h5", 'w') as hf:
		hf.create_dataset("seqids", data=keys, compression='gzip')
		hf.create_dataset("embeds", data=vals, compression='gzip')
		
Hw = measure_time(dataset, write_hdf5_comp)

print(f"HDF5 (write): {Hw:.2f} secs")
```

100 MB 데이터셋과 500 MB 데이터셋을 이용해 측정해본 실험과 저장용량은 다음과 같다.

|    | # of data | 10,000 | 50,000 |
|--:|---------:|-------:|--------:|
|    | memory | 103.32 MB | 517.72 MB |
| read | HDF5 (raw) | 0.06 | 0.34 |
|  | HDF5 (comp) | 0.7 | 3.45 |
| write | HDF5 (raw) | 0.35 | 3.11 |
|  | HDF5 (comp) | 4.14 | 26.36 |
| storage | HDF5 (raw) | 98 MB | 489 MB |
|  | HDF5 (comp) | 93 MB | 461 MB |

음... 랜덤 데이터셋이라 압축될 부분이 많이 없어서 그런지는 몰라도 compression여부가 용량에 큰 영향을 주지는 않았다 ㅠㅠ 심지어 수행시간은 거의 10배가량 늘어난... 딱히 쓸 이유를 찾지 못한 옵션 ㅇㅅㅇ

(0 이 많거나, 중복되는 값이 많은 데이터의 경우에는 큰 효과를 볼 수도 있으니, 추가적인 실험이 조금 필요해보인다.)

## 나가며

Pickle 과 HDF5를 비교해봤을 때, 굳이 어느 한 패키지가 월등히 좋다고 평가할 수는 없을 것 같다. 각자가 편한 방법이나 이미 구현해둔 코드가 있다면 그걸 활용하면 될 것 같다.

다만 엄청 큰 데이터셋의 경우 (내 데이터의 경우 5 GB 짜리) HDF5에서 chunk 옵션을 지원하기 때문에 in-memory 성능이 조금 더 좋을 수 있을 것 같다. 엄청 큰 컴퓨터에서 작업하면서 메모리를 생각하지 않고 써도 된다거나 하는 상황이 아니라면 일일히 쪼개서 저장해야하는 pickle 보다는 장점이 될 것 같기도 하다!


