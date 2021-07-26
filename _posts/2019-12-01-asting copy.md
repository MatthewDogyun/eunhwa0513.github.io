---
layout: post
title: Spark broadcasting을 이용해 기존 머신러닝 라이브러리 속도 개선하기
subheading: JupyterHub를 구축했다면 사용자 별 환경과 리소스를 구분하기 위한 설정방법을 알아봅시다.
author: 김연수 | Data Analyst
categories: 머신러닝
tags: python sklearn ml spark
sidebar: []
---

# Spark Broadcast

> 우리에게 필요한 것은 inference 효율성 😎
> 
> > 기존 python ml 라이브러리로 다량의 데이터를 빠르게 처리를 해야 할 때!  Spark broadcasting을 이용해 inference 속도를 개선할 수 있습니다.

  

## Spark의 공유 변수

----------

`아피치 스파크`는 SQL, 머신러닝 등 처리를 위한 기본 제공 모듈이 있는  `대규모 데이터 처리용 통합 분석 엔진`입니다.

  
일반적인 스파크 연산이 클러스터에서 실행될 때 함수에서 사용되는 변수는 각 클러스터에 복제되고 각 노드 연산에서는 독립적입니다. 클러스터 상에서 변수를 공유하는 것은 비효율적이기 때문인데, 일반적인 연산 패턴을 지원하기 위해 아래와 같은 공유 변수를 제공합니다 .

-   broadcast variables : 큰 데이터를 효율적으로 처리하는데 사용
-   Accumulators variables : 특정 컬렉션의 정보를 집계하는 데 사용

오늘은  `broadcast variables`에 대하여 알아보도록 하겠습니다.

  

## Spark broadcast 란?

----------

broadcast 변수는 읽기 전용 변수입니다. 모든 노드에 큰 input 데이터셋을 제공할 때 효과적인 방법이라고 할 수 있습니다. 이를 사용하면 효율적인 broadcast 알고리즘을 통해 계산 비용을 줄일 수도 있습니다.

  

스파크의 액션들은 기본적으로 여러 단계의 집합으로 실행되는데, 자동적으로 각 단계에 필요한 공유 변수들을 broadcast하는 것입니다.

Broadcast 변수는 RDD나 Dataframe과 동일한 방식으로 사용될 수 있습니다. (데이터는 serialized 캐싱되고 deserialized 되어 각 테스크로 넘어가게 됩니다.)

  
호출 방법은 다음과 같습니다.

```ini
sc = SparkContext()
var = Array(4,5,6) # 사용할 공유 변수 var

broadcastvar = sc.broadcast(var)

```

일단 broadcast variable이 만들어지면, 클러스터상의 모든 함수에 사용될 수 있습니다. (즉,  **var은 한 번 이상 노드에 올라가지 않게**  됩니다.)

참고로, 한 번 생성된 broadcastvar 객체는 이후 수정될 수 없습니다.

  
broadcast 된 값을 이용하기 위해서는  `value 메소드`를 사용하면 됩니다.

```markdown
# value 값에 액세스 하기 위함
broadcastvar.value

# output
> Array(4,5,6)

```

  

## sklearn에 spark broadcast를 적용해 inference 하기

----------

python에서 가장 많이 이용되는 mechine learning library는 sklearn일 것입니다. sklearn 에서는 보통 pandas dataframe 을 활용한 데이터를 fit하고 predict 하게 됩니다.

예를 들어, KNN (K-Nearest Neighbors)과 같은 알고리즘을 사용하게 된다면 우리는 각 key 값에 대한 각각의 k개의 neighbors를 모두 구해야합니다. 즉,  **key 값 하나하나에 모든 데이터를 연산**하게 됩니다.결국 inference 시간이 엄청나게 길어질 수 밖에 없는 구조가 됩니다.

이는  **sklearn model을 broadcast 하는 방식**을 활용해 효율적으로 처리될 수 있습니다.

  

예제 코드는 다음과 같습니다. ( knn model 사용 )

```python
def knn_regressor_broadcast(data_pd, neighbors=100, n_partitions=10, weights = "uniform"):

  # X_train, y_train, X_test, y_test 의 경우 pandas dataframe 형태
  

  # 모델 fit
  knn = KNeighborsRegressor(n_neighbors = neighbors, weights = weights)
  model = knn.fit(X_train, y_train)
  
  # 여러 배치로 돌리기 위한 함수
  def batch(value):
    yield list(value)
    
  rdd = sc.parallelize(X_test, n_partitions).zipWithIndex()
  b_model = sc.broadcast(model)
  result = rdd.mapPartitions(batch).map(lambda value: ([x[0] for x in value], [x[1] for x in value])).flatMap(lambda x: zip(x[1], b_model.value.predict(x[0])))
  
  result = result.collect()

  return result

```

위와 같이  **model을 broadcast + rdd test data**를 사용하자 Pandas dataframe을 이용해 predict를 하였을 때 Runtime errorrk가 났던 모델을 약 5분 내외로 처리할 수 있었습니다 👏👏

  

부가적인 코드 설명을 잠시 하자면

mapPartitions의 경우 나눠진 partition내에서 연산을 수행하게 해줍니다

```python
rdd = sc.parallelize([1, 2, 3, 4], 2)
def f(iterator): yield sum(iterator)
rdd.mapPartitions(f).collect()

# output
> [3, 7]

```

즉, broadcast된 모델에 test 데이터를 batch 형태로 분산 처리가 가능해진다고 할 수 있습니다.

  

## 참고 자료

----------

👉  [About advaced spark programming - spark broadcast](https://www.tutorialspoint.com/apache_spark/advanced_spark_programming.htm)

👉  [Spark Broadcast Varibales](https://sparkbyexamples.com/spark/spark-broadcast-variables/)