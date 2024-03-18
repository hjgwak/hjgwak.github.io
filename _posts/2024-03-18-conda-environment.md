---
title: "[Python] conda environment export/import"
author: Johin
date: 2024-03-18 10:30:00 +0900
categories: [Programming, Python]
tags: [python, conda]
---

정말 간단하지만 자주 사용하진 않아서 맨날 헷갈리고 찾아보는 콘다 환경 관리!  

가끔 패키지 충돌 등의 이슈로 `latest`를 설치하지 않고 기존에 설치된 것들을 그대로 써야하는 상황이 생긴다.  
패키지 한 두개라면 그냥 손으로 명시해주겠지만, 생각보다 패키지가 많을 때도 있고 나도 모르는 디펜던시가 있을 때가 있다.

conda에서는 지금 사용중인 환경을 공유할 수 있게 출력하는 기능과 공유받은 파일을 기반으로 환경을 구축할 수 있는 기능을 제공한다.

## How to export conda environment
```bash
conda activate my_env
conda env export > my_env.yaml
```

`conda env export` 명령어로 현재 활성화된 환경을 export 할 수 있다.  
아래는 환경을 export 했을 때 나오는 파일의 구성요소이다.

![yaml](/assets/img/20240318/yaml.png)

* **name**: export 된 환경의 이름.
* **channels**: 패키지들을 다운로드 받을 때 사용된 채널들. 우선순위대로 배치되어 있다.
* **dependencies**: 환경에 설치되어있던 패키지들의 리스트. (pip를 통해 설치된 피캐지는 별도로 표시되어있다.)
* **prefix**: 현재 머신에서 해당 환경이 저장되어있는 위치를 나타낸다.

## How to import conda environment

환경을 export 했다면, 당연히 해당 파일을 이용해서 환경을 구축할 수도 있어야 한다.  
이 방법역시 아주 간단하다.

```bash
conda create -f my_env
conda activate my_env
```

이렇게 환경을 세팅할 때, 위 `yaml` 포맷에서 말한 것 중 **name**에 해당하는 이름 그대로 환경이 생성된다.  

> 만일 다른 이름으로 환경을 생성하고 싶다면  
>  yaml 파일에서 **name**에 해당하는 부분을 수정해주면 된다.
>  물론 **prefix**에 해당하는 부분도 함께 수정해주어야 한다.
{: .prompt-tip }

## How to clone conda environment

가상환경을 다른 머신으로 넘겨야 할 상황도 있지만, 테스트를 위해 같은 머신에서 환경을 복제해야할 상황도 있다.  
위의 export / import 를 순차적으로 수행해도 되지만, 이걸 간단하게 할 수 있는 명령어도 존재한다!

```bash
conda create -n new_env --clone my_env
```

위 명령어를 수행하면 `my_env`의 내용을 그대로 복사하여 `new_env`라는 이름으로 환경을 생성해준다.