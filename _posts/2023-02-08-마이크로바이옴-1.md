---
title: "[Bioinfo] 마이크로바이옴 다양성 (Diversity) 분석"
author: Johin
date: 2023-02-08 12:00:00 +0900
categories: [Bioinfomatics, Microbiome]
tags: [python, scikit-bio, microbiome, diversity]
---

마이크로바이옴 데이터를 볼 때 항상 빠지지 않고 보는 분석이 바로 `다양성 (Diversity) 분석`이다. 그런데 할 때마다 예전에 코드 어떻게 짰었는지를 까먹고, 예전코드를 뒤적이는게 귀찮아서 이참에 블로그에 정리해본다.

마이크로바이옴 조성 (Microbiome profile)은 주로 **16S rRNA**를 사용해 추정하지만, **Shotgun WGS**데이터로부터 추정하기도 한다. 어쨌든 이번 포스트에서는 추정된 조성테이블을 분석하는 방법들만 정리해보기로 한다.

입력으로 사용할 파일은 아래와 같은 형식의 테이블이다.

| taxa | sample1  | sample2  | ...  | sampleN  |
|:----|---------:|----------:|---:|---------:|
| taxon1 | 0.012 | 0.23         |      | 0.005     |
| taxon2 | 0.37  | 0.31         |       | 0.32       |
| taxon3 | 0.05 | 0.01         |       | 0.12        |
| ...    |    |    |    |    |
| **Total** | 1.0  | 1.0  |  1.0  | 1.0  |

> Relative abundance의 표현은 0.0 - 1.0 범위나 0.0 - 100.0 범위중 어떤 것을 써도 상관 없다.
{: .prompt-tip}

각 샘플에 대한 메타데이터 (meta data)를 담은 파일도 필요하다.  

| sample name | antibiotics | sequencing platform | ... |
|:-------------|:-----------|:--------------------|:---|
| sample1  | pre  | Shotgun |  |
| sample2  | pre | Shotgun |  |
| ...  |  |  |  |
| sampleN | post | Shotgun |  |

## Alpha diversity

가장 먼저 분석하는 것은 `alpha diversity`이다. 간략하게 설명하자면, 얼마나 많은 종류의 미생물이 존재하는지를 측정하는 것이다. 예를들어, 항생제 (antibiotics) 처방 전/후를 비교할 때 미생물의 종류가 많이 줄어들었는지 아닌지를 보는 것이다.

`skbio.diversity` 모듈 안에 `alpha_diversity` 함수를 이용해 쉽게 구할 수 있다.

```python
# base_table: microbiome profile table
# meta: sample metadata table

from skbio.diversity import alpha_diversity

alpha = pd.DataFrame({
    "chao1": alpha_diversity("chao1", base_table.T, base_table.columns),
    "shannon": alpha_diversity("shannon", base_table.T, base_table.columns),
    "simpson": alpha_diversity("simpson", base_table.T, base_table.columns)
})
```

**Chao1, Shannon index, Simpson index**는 alpha diversity를 측정하는데 가장 많이 사용되는 계산식들이다. **Chao1**의 경우 단순히 등장하는 미생물의 개수를 측정하고, **Shannon index**는 정보이론에서 엔트로피와 같은 계산식을 사용한다. 자세한 것들이 궁금하다면 위키피디아를 찾아보도록하자.

어쨌거나 위 코드를 실행하면 다음과 같이 샘플별로 alpha diversity 값이 구해지게된다.

| sample  | chao1  | shannon  | simpson  |
|:--------|-------:|---------:|---------:|
| sample1 | 33      | 3.298      | 0.858      |
| sample2 | 36     | 3.628      | 0.879      |
| ...            |          |                |                 |
| sampleN | 18     | 0.978      | 0.282      |

> **Chao1**의 경우 Relative abundance를 반영하지 않기 때문에 스킵하는 경우도 많다.
{: .prompt-tip}

결과로 나온 테이블을 boxplot으로 표현해보면 다음과 같이 볼 수 있다. 
*visualization은 다음 포스팅에서 다루도록 하자*

![alpha](/assets/img/20230208/Alpha.genus.pdf)

위쪽이 항생제 처리하기 전 샘플들이고 아래쪽이 항생제를 처리한 후 샘플들이다.  
항생제를 처리하기 전보다 후가 다양성의 분포가 더 큰게 한 눈에 보인다.  
(항생제처리 후가 일괄적으로 다양성이 낮으면 제일 좋겠지만, 언제나 데이터는 우리가 원하는대로만 나와주지는 않는다 ㅠㅠ)

## Beta diversity

`alpha diversity`가 얼마나 많은 미생물이 존재하는지를 측정했다면, `beta diversity`는 미생물의 **분포**가 얼마나 비슷한지를 측정하는 것이다. 

한가지 예시를 떠올려보자, 간단하게 A, B 두 종류의 박테리아가 존재한다고 하고. 항생제 처리 전에는 두 박테리아가 1:9 비율로, 처리 후에는 9:1 비율로 존재한다고 생각해보자.

|   | pre  | post  |
|:--|----:|-----:|
| **A**  | 1  | 9  |
| **B**  | 9  | 1  |  

alpha diversity를 측정한다면, Chao1을 사용하든 Shannon index를 사용하든 같은 값이 나올 것이다. 하지만 딱 봐도 항생제 처리 전/후 미생물 조성에는 분명한 차이가 있다. (아마도 A 박테리아는 항생제에 내성이 있고, B 박테리아는 항생제에 민감하다고 생각할 수 있다.)

`beta diversity`는 두 조성이 얼마나 비슷한지를 측정하기 때문에, 위와 같은 경우에 유의미한 차이를 보일 수 있다. 여기서 **얼마나 비슷한지**를 어떻게 측정할 것이냐도 중요한데, 주로 `Bray-Curtis dissimilarity`를 사용한다.

`skbio.diversity` 모듈 안에 `beta_diversity` 함수를 이용해 쉽게 구할 수 있다.

```python
# base_table: microbiome profile table
# meta: sample metadata table

from skbio.diversity import beta_diversity
from skbio.stats.ordination import pcoa

beta = beta_diversity("braycurtis", base_table.T, base_table.columns)

pcoa_res = pcoa(beta)
embedded = pcoa_res.samples[['PC1', 'PC2']]
embedded = pd.merge(left = embedded, right = meta, left_index=True, right_index=True)
```

`beta_diveristy`함수를 수행하면 모든 샘플간의 거리를 계산한 distance matrix가 반환된다. 총 N개의 샘플이 있다고 할 때, **N x N matrix**가 반환된다. distance matrix를 heatmap등을 이용해서 볼 수도 있지만, **PCoA 분석**을 주로 동반하게 된다. 

> Bray-Curtis dissimilarity를 계산하지 않고, base_table에 대해 PCA 분석을 하기도 한다.  
> PCA 분석을 하면 정확히 어떤 박테리아가 얼마만큼 기여를 하는지 수치로 측정할 수 있다는 장점이 있다.  
> 하지만 나는 대세를 따라 PCoA 분석을 이용하는 편이다. 정답은 없다.
{: .prompt-tip}

어쨌거나 beta diversity 분석 결과를 그림으로 표현하면 아래와 같이 볼 수 있다.

![beta](/assets/img/20230208/Beta.genus.pdf)

솔직히 그림만 봐서는 유의미한지 아닌지 판단이 잘 서지는 않는다. 개인적으로 코에걸면 코걸이 귀에걸면 귀걸이 식의 해석이 가능한 영역인 것 같다. 그래도, 확실하게 구분되는 편은 아니어도 어느정도 차이는 보이는 것 같다.

## 마치며

이번 포스팅에서는 다양성분석에 가장 자주 사용되는 두가지 방법을 정리해봤다. 가장 마지막에 말한 것 처럼 그림만 봐서는 정확한 해석을 할 수 없고, 과학적이지도 못하다. 두 그림을 제시하며 필연적으로 통계분석이 이루어져야 한다. 다음 포스팅에서는 내가 주로 사용하는 통계분석을 정리해보려고 한다.