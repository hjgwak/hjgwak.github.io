---
title: "[Bioinfo] 우리는 지금까지 Precision, Recall에 속고있었다"
author: Johin
date: 2024-10-11 13:00:00 +0900
categories: [Bioinfomatics, Statistics]
tags: [classifier, statistics]
mermaid: true
---

## 들어가기 전에

AI model이 여러 분야에서 좋은 성적을 거두면서, bioinformatics 에도 AI를 활용한 예측모델 (predictive model)연구가 활발하다. 우리가 연구하는 이유는 자연에 대한 단순한 호기심을 넘어, 우리에게 이로운 어떤 것을 발견/개발해내기 위함이기 때문에 굉장히 중요한 연구들이라고 할 수 있다.

어떤 질병의 발생 가능성을 예측하거나 MRI 사진등을 근거로 질병이 있는지 없는지를 구분해내는 모델을 잘 만들어낸다면, 임상의료에서도 큰 혁신을 가져올 수 있을 것이다.

여기서 **잘 만들어 낸다면**이 중요하다. 질병이 있는지 없는지 50:50으로 찍어 맞추는 모델이 있다면 전혀 쓸모가 없을 것이기 때문에, 개발한 모델이 **얼마나 정확한지**를 측정하는 것이 굉장히 중요하다.

이 때 전통적으로 가장 널리 쓰이는 3가지 측정지표가 있다.

1. **Accuracy (정확도)**
2. **Recall (재현율)**
3. **Precision (정밀도)**

해당 지표에 대한 자세한 내용은 여기를 참고하자.

여기에서 이번에 가장 눈여겨 볼 지표는 **Precision (정밀도)**인데, 정의상 모델이 "참이라고 예측"한 예측들 중 "실제로 참"인 예측들의 비율을 말한다. 즉, 암이 있는지 없는지 판별하는 모델이 "암이 있다!"라고 말했을 때 얼마나 믿을만 한지를 나타내는 지표라고 생각하면 되겠다.

자! 이제 3가지 예측 모델을 제시해보겠다. 만일 당신이 의사라면, 어떤 모델을 가지고 진료할 지 한 번 생각해보자.

|                | Accuracy | Recall | Precision |
|:---------|---------:|-------:|---------:|
| Model A |        90% |    90% |    90.0% |
| Model B |        90% |    90% |     97.8% |
| Model C |        90% |    90% |     64.5% |

나는 마술사가 아니지만, 당신이 어떤 모델을 선택했을지 맞출 수 있을 것 같다.

## 정밀도는 고정된 값이 아니다!

제목을 살짝 자극적으로 적었다. **실제로 정밀도는 고정된 값이다.** 하지만, 측정하는 테스트셋을 어떻게 구성하느냐에 따라 **잘못측정될 수 있다**.

과연 어떻게 달라질 수 있을지 간단한 실험 코드를 이용해서 살펴보자.

먼저 간단한 예측 모델을 만들어보자!

```python
import numpy as np

class Model:
	# acc_pos: positive data에 대한 정확도 (0 ~ 1)
	# acc_neg: negative data에 대한 정확도 (0 ~ 1)
	def __init__(self, acc_pos, acc_neg):
		self.acc = [acc_neg, acc_pos]
        
        # label = [0, 1]
        def pred(self, label):
        	p = np.random().random()
        	if p < self.acc[label]:
        		return label
        	else:
        		return 1 - label
```

이 모델은 아주 간단하다. 입력된 데이터에 따라 예측을 진행해줄텐데, 사전에 입력된 정확도에 기반해서 가끔 노이즈를 섞어줄 것이다.

예를 들어, `acc_pos`를 0.9로 세팅했다면, `참`인 데이터에 대해서 90% 확률로 `참`이라고 대답하고 10% 확률로 `거짓`이라고 대답할 것이다. 어떤 문제는 `참`인 데이터는 너무 명확한데, `거짓`인 데이터를 구별하기 어려운 문제일 경우 `(acc_pos, acc_neg)`는 `(0.95, 0.7)`이 될 수도 있다. 반대로, `거짓`을 구별해내기 굉장히 쉽지만 `참`을 구별해 내기 어려운 문제일 경우 `(acc_pos, acc_neg)`가 `(0.5, 0.99)`가 될 수도 있다.

이제 해당 모델을 평가할 함수를 코딩하자.

```python
from sklearn.metrics import recall_score, precision_score, accuracy_score

def evaluate(model, N_POS, N_NEG):
	y_true = [1] * N_POS + [0] * N_NEG
	y_pred = [model.pred(label) for label in y_true]
	
	accuracy = accuracy_score(y_true, y_pred)
	recall = recall_score(y_true, y_pred)
	precision = precision_score(y_true, y_pred)
	
	return (
		accuracy,
		recall,
		precision
	)
```

큰 수의 법칙이 적용될 수 있게, positive sample과 negative sample을 각각 10만개씩 사용해서 모델의 성능을 측정해보자.

```python
model = Model(0.9, 0.9)
evaluate(model, 100000, 100000)
"""
Accuracy: 0.901
Recall: 0.902
Precision: 0.900
"""
```

성능이 잘 측정되는 것 같은데 뭐가 문제일까??


## 데이터 비율의 함정

잠시 앞에서 다뤘던 **Recall**과 **Precision**의 정확한 정의를 되짚어보고 가자

![PrecisionRecall](/assets/img/20241011/precisionrecall.jpg)

한가지 특징이 눈에 띄는가? 바로 **True Negative (TN)**가 계산에 반영되지 않는다. 다른 말로 하자면, Negative dataset이 영향을 주는 곳은 오로지 **False Positive (FP)**뿐이라는 사실이다.

아주 작은 실험을 한 번 더 해보자, 앞의 실험과 똑같은 조건에서 negative dataset만 2배로 늘려보는 것이다.

```python
model = Model(0.9, 0.9)
evaluate(model, 100000, 200000)
"""
Accuracy: 0.899
Recall: 0.899
Precision: 0.816
"""
```

**Accuracy**와 **Recall**은 오차범위 내에서 여전히 잘 나오는 것 같은데, **Precision**은 어떻게 된 일일까?

Negative dataset이 2배로 늘면서 FP의 개수도 2배로 늘었기 때문이다. Positive dataset의 크기는 동일하기 때문에 TP의 개수는 늘지 않았다. 만약 반대로, Negative dataset보다 Positive dataset의 개수가 더 많다면? FP의 개수는 일정한데 TP의 개수가 늘 것이기 때문에 **Precision**이 높게 측정될 수 있다.

위 실험을 Positive dataset과 Negative dataset의 비율을 바꿔보면서 반복실험 해보면 다음과 같은 결과를 얻을 수 있다.

| Positives:Negatives | Accuracy | Recall | Precision |
|:-------------------|---------:|-------:|---------:|
| 1:1                            |       90% |    90% |    90.0% |
| 2:1                           |       90% |    90% |     94.7% |
| 3:1                           |       90% |    90% |     96.5% |
| 4:1                           |       90% |    90% |     97.3% |
| 5:1                           |       90% |    90% |     97.8% |
| 1:2                           |       90% |    90% |     81.6% |
| 1:3                           |       90% |    90% |     75.0% |
| 1:4                           |       90% |    90% |     69.2% |
| 1:5                           |       90% |    90% |     64.5% |

결과적으로 **Precision은 테스트 셋 내부의 Positive set과 Negative set의 비율에 영향을 받는다!!** 

여기에서 아마 눈치 챘겠지만, 시작할 때 언급했던 3가지 모델은 사실 같은 모델이다. 그저 테스트셋을 다르게 해서 성능을 각각 측정했을 뿐이다. 아마 첫 질문에서 모두들 **Model B**를 선택했겠지만, 사실 어떤 모델을 선택하든지 같은 모델을 선택한 것이었다. 중요한 점은 테스트셋의 비율을 조절하는 것으로 내 모델의 성능이 좋게보이게 만들 수 있다는 점이다.

사실 **Accuracy** 역시 동일하게 영향을 받는다. 모델의 Positive set에 대한 성능, Negative set에 대한 성능을 다른 값으로 주고 `model(0.8, 0.95)` 위 실험을 다시 반복해보자.

| Positives:Negatives | Accuracy | Recall | Precision |
|:-------------------|---------:|-------:|---------:|
| 1:1                            |    87.5% |    80% |     94.1% |
| 2:1                           |    85.0% |    80% |    97.0% |
| 3:1                           |    83.7% |    80% |    98.0% |
| 4:1                           |    82.9% |    80% |    98.4% |
| 5:1                           |     82.5% |    80% |    98.8% |
| 1:2                           |    90.0% |    80% |    88.9% |
| 1:3                           |     91.2% |    80% |    83.9% |
| 1:4                           |    92.0% |    80% |    80.0% |
| 1:5                           |     92.5% |    80% |     76.1% |

## 조작할 수 없는 Metrics

우리가 테스트셋을 잘 만들 수 있다면 positive set과 negative set의 비율을 1:1로 유지하는 것으로 문제를 우선 해결할 수 있다. 하지만 이게 쉽지 않은 경우도 있다. Positive set 또는 Negative set 중 하나가 너무 적은 상황 이라거나, binary classification이 아닌 multi-lable/multi-class classification의 경우 이 비율을 맞추는 것은 훨씬 어렵다.

여기에 대안으로 제시할 수 있는 Metric 들이 있다.

1. **True Positive Rate (TPR)**: TP / (TP + FN)
2. **True Negative Rate (TNR)**: TN / (FP + TN)
3. **Balanced Accuracy**: (TPR + TNR) / 2

**TPR**은 Positive set 내에서의 성능에만 영향을 받는다, **TNR**은 Negative set 내에서의 성능에만 영향을 받는다. 즉, Positive:Negative 의 영향을 받지 않는다는 말이다. 마지막으로 한 실험에 **TPR**, **TNR** column을 추가해보자

```python
from sklearn.metrics import recall_score, precision_score, accuracy_score
from sklearn.metrics import confusion_matrix

def evaluate(model, N_POS, N_NEG):
	y_true = [1] * N_POS + [0] * N_NEG
	y_pred = [model.pred(label) for label in y_true]
	
	accuracy = accuracy_score(y_true, y_pred)
	recall = recall_score(y_true, y_pred)
	precision = precision_score(y_true, y_pred)
	
	cf_mat = confusion_matrix(y_true, y_pred)
	tnr = cf_mat[0, 0] / (cf_mat[0, 0] + cf_mat[0, 1])
	tpr = cf_mat[1, 1] / (cf_mat[1, 0] + cf_mat[1, 1])
	
	return (
		accuracy,
		recall,
		precision,
		tpr,
		tnr,
		(tpr + tnr) / 2
	)
```

| Positives:Negatives | Accuracy | Recall | Precision | TPR | TNR | B_ACC |
|:-------------------|---------:|-------:|---------:|-----:|-----:|------:|
| 1:1                            |    87.5% |    80% |     94.1% | 80% | 95% | 87.5% |
| 2:1                           |    85.0% |    80% |    97.0% | 80% | 95% | 87.5% |
| 3:1                           |    83.7% |    80% |    98.0% | 80% | 95% | 87.5% |
| 4:1                           |    82.9% |    80% |    98.4% | 80% | 95% | 87.5% |
| 5:1                           |     82.5% |    80% |    98.8% | 80% | 95% | 87.5% |
| 1:2                           |    90.0% |    80% |    88.9% | 80% | 95% | 87.5% |
| 1:3                           |     91.2% |    80% |    83.9% | 80% | 95% | 87.5% |
| 1:4                           |    92.0% |    80% |    80.0% | 80% | 95% | 87.5% |
| 1:5                           |     92.5% |    80% |     76.1% | 80% | 95% | 87.5% |

예상했던대로 Positive set과 Negative set의 비율에 영향을 받지 않고 모델의 성능이 측정되었다!! *(야호)*

사실 이게 많은 머신러닝 모델에서 **ROC curve**를 이용해서 성능을 측정하는 이유일 것이다. **ROC curve**는 classification boundary에 따라서 **TPR:FPR**을 측정하는 것인데, **FPR**은 **1-TNR**로 정의된다. 즉, 우리가 위에서 측정한 두 값을 이용해서 모델의 성능을 평가한다는 말이다.

## 나가며

간단히 정리하자면 이렇다.

1. Accuracy 와 Precision 은 테스트 데이터 내의 label 의 비율에 영향을 받는다.
2. 두 값을 비교적 정확히 측정하려면 label의 비율이 동일해야한다.
3. TPR, TNR을 활용할 경우 label의 비율을 신경쓰지 않아도 된다.


