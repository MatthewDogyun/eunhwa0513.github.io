---
layout: post
title: 설명가능한 부스팅 모델 - Explainable Boosting Machine
subheading: 설명가능한 부스팅 모델 - Explainable Boosting Machine
author: 전지호
# 백상현 | Data Analyst
categories: 머신러닝
banner:
  image: "/assets/images/post/machine.png"
tags: mlops aws kubernetes
sidebar: []
---
# 설명가능한 부스팅 모델 Explainable Boosting Machine

> 설명하기 어려운 XGBoost, LightGBM, RandomForest와 대등한 성능을 가지면서, 설명 가능한 Explainable Boosting Machine의 원리, API 사용 방법, 활용 사례에 대해 알아보자.



## accuracy vs interpretability trade-off

----------


![img](https://miro.medium.com/max/978/1*SI3wAOvfTQrLl5NXQHwuxA.png)

머신러닝에서 모델의 예측 정확도와 설명 가능성이 trade-off 관계를 보인다는 것은 정설처럼 받아들여졌다.

부스팅, 딥러닝 모델처럼 높은 예측 정확도를 보이는 모델들은 각 feature가 의사결정에 어떻게 관여하는지 파악하기 어렵고

선형회귀, 의사결정나무 같이 각 feature의 의사결정 기여를 설명하기 쉬운 모델들은 보편적으로 낮은 예측 정확도를 보인다.

즉, 우수한 예측 성능과 설명 가능성을 모두 확보하는 것은 불가능하다고 여겨져왔다.



## Explainable Boosting Machine (EBM)

----------


그러나, 마이크로소프트의 interpretML 패키지에 포함된 Explainable Boosting Machine(EBM)은 우수한 예측 성능과 설명 가능성 두마리 토끼를 모두 잡은 것처럼 보인다.

![ebm_performance](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/ebm_performance.png)

ebm_performance

5개의 테스트 데이터셋에 대해 EBM, LightGBM, 로지스틱 회귀, 랜덤 포리스트, XGBoost 모델을 학습시킨 결과이다.

EBM은 높은 설명 가능성을 가지고 있음에도 불구하고, LightGBM, 랜덤 포리스트, XGboost같은 모델들과 동등한 수준의 성능을 보인다.

이는 EBM을 활용한다면 우리는 더이상 높은 예측 정확도를 확보하기 위해 설명 가능성을 포기할 필요가 없다는 것을 의미한다!


**그렇다면 설명 가능성을 포기하지 않는게 왜 큰 장점인가?**

-   모델 디버깅

> 설명 가능성이 높은 모델을 사용한다면,  
> 단순히 모델의 잘못된 예측을 관찰하는것을 넘어서, 잘못된 예측의 의사결정 과정을 확인할 수 있다  
> 이를 기반으로 overfitting의 징후를 찾고, 문제를 일으키는 데이터 전처리 과정을 찾는 등 모델 디버깅을 수월하게 진행할 수 있다.

-   공정성 이슈 확인

> 모델에 공정하지 못한 bias가 존재하는지 확인할 수 있다.

-   모델 신뢰성

> 의사결정 과정을 확인할 수 있는 모델은 사람이 더 신뢰할 수 있고, 머신러닝 도메인 지식이 없는 사람들에게 의사결정 과정을 설명하기 용이하다.

-   하이 리스크 의사결정

> 의료, 금융같은 산업 분야에서 머신러닝 모델을 사용한다면, 잘못된 예측에 대한 리스크가 매우 크다.  
> 설명 가능성이 높은 모델을 사용한다면 모델의 예측을 무작정 믿지 않고, 모델의 의사결정 과정을 검토하며 sanity check을 진행할 수 있다.



## EBM 모델 구조

----------


EBM은 Generalized Additive Model (GAM)의 발전된 형태인 GA2M 모델에 속하는 알고리즘이다. 조금 더 자세히 설명하자면, FAST알고리즘으로 pairwise feature interaction term을 선택하는 GA2M 알고리즘이다.


**Generalized Additive Model (GAM)**

아래의 형태로 구성된 예측 모델을 Generalized Additive Model (GAM)이라 부른다.

> g(E(y))=β0+Σfi(Xi)g(E(y))=β0+Σfi(Xi)

gg를 link function으라 지칭하고, 1개의 feature를 입력으로 받는  fifi를 shape function이라 지칭한다.

regression task에서  gg는 identity function이고:  E(y)=β0+Σfi(Xi)E(y)=β0+Σfi(Xi)

classification task에서  gg는 logistic function이다:  logp1−p=β0+Σfi(Xi)log⁡p1−p=β0+Σfi(Xi)

전통적으로 backfitting 알고리즘, regression spline이 GAM 학습에 사용되어왔다.



GAM은 Generalized Linear Model과 형태가 흡사하다.

multiple linear regression:  E(y)=β0+ΣβiXiE(y)=β0+ΣβiXi

logistic regression:  logp1−p=β0+ΣβiXilog⁡p1−p=β0+ΣβiXi



GAM의 형태는 Generalized Linear Model처럼 각 feature에 대한 연산값의 합을 의사결정에 활용하기 때문에 각 feature가 의사결정에 어떤 영향을 미치는지 파악하기 쉬우면서도, Generalized Linear Model에 비해 feature와 target값의 비선형적인 관계를 더 잘 파악할 수 있어, linear model에 비해 예측 성능이 훨씬 좋다.

_Note: Gradient Boosting같은 Full Complexity Model은  y=f(x1,…,xn)y=f(x1,…,xn)  형태로 예측을하기 때문에 각 feature가 의사결정에 어떻게 기여하는지 쉽게 파악할 수 없다._

하지만, GAM은 feature  xi,xkxi,xk의 pairwise interaction을 모델에 포함할 수 없다는 한계점 있다.



**Generalized Additive Model plus Interactions (GA2M)**

GAM의 한계점을 극복하기 2 feature간의 pairwise interaction을 담을 수 있는 pairwise interaction term이 추가된 알고리즘이 GA2M이다.

아래의 형태로 구성된 예측 모델을 Generalized Additive Model plus Interactions (GA2M)이라 부른다.

> g(E(y))=β0+Σfi(Xi)+Σfi,j(xi,xj)g(E(y))=β0+Σfi(Xi)+Σfi,j(xi,xj)

pairwise interaction term  fi,j(xi,xj)fi,j(xi,xj)은  xi,xjxi,xj평면에 heatmap 형태도 시각화할 수 있고.

xi,xkxi,xk의 pairwise 값이 모델의 의사결정에 어떤 영향을 미치는지 비교적 쉽게 파악할 수 있다.



feature의 수가 많아지면 pairwise interaction term의 수도 폭발적으로 증가한다.

GA2M 모델을 생성할 때 모델 싸이즈가 과도하게 커지는 것을 방지하지 위해 통계적으로 유의한 수준의 pairwise interaction을 가지는  xi,xjxi,xj에 해당하는  fi,j(xi,xj)fi,j(xi,xj)을 선택할 필요가 있다.

전통적으로 ANOVA , Partial Dependence Function, GUIDE, Grove등이 pairwise interaction 검증에 사용되었다.



**Explainable Boosting Machine**

Explainable Boosting Machine (EBM)은 현대적인 머신러닝 기법(gradient boosting 등)을 활용해 feature function  ff를 학습하고, FAST 알고리즘으로 pairwise interaction term을 선택하는 GA2M 모델이다.



## EBM 학습 과정

----------



Explainable Boosting Machine은 2 단계의 과정을 거치며 학습을 진행한다.

1.  gradient boosting으로 pairwise interaction을 고려하지 않은  fifi  학습
2.  FAST 알고리즘으로 모델에 포함할  fi,jfi,j  선택 후 gradient boosting으로 학습


**1. gradient boosting으로 pairwise interaction을 고려하지 않은 GAM 모델 학습**

![image-20210622021726012](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/boosting_gam.png)

EBM은 위 알고리즘을 사용해 pairwise interaction을 고려하지 않은 GAM 모델을 학습한다.

보편적인 부스팅 기법과 유사하게 데이터에 작은 트리를 피팅하는 것으로 학습이 시작하고, 다음 트리는 이전 트리의 residual을 예측하도록 학습된다. 조금 다른 점은 트리를 학습할 때 1개의 feature (feature1)만 사용할 수 있도록 제한한다는 것이다.

![image-20210622024256361](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/gam_p1_1.png)  
마찬가지로, 두번째 트리도 1개의 feature (feature2)만 사용하여 첫번째 트리의 residual을 예측하도록 트레이닝합니다.

![image-20210622024707598](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/gam_p1_2.png)  
이런 형식으로 모든 feature를 순차적으로 사용해서 base estimator를 학습 시키면 한 iteration이 완료됩니다.

모든 트리를 학습 시킬 때 아주 작은 learning rate를 적용하여 feature에 트리를 적용 시키는 순서가 무의미하도록 유도합니다.

![image-20210622024900319](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/gam_p1_3.png)  
이 과정을 M iteration 반복합니다.

![image-20210622031525624](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/gam_p1_4.png)

M iteration이 완료되면 각 feature 만을 독립변수(x)로 받는 트리가 M개의 생성됩니다. 각 feature에 해당하는 트리 예측값의 합을 lookup table (graph)로 변환하고 트리를 삭제합니다.

이후  x1x1  가 주어지면 해당 그래프로  f1f1값을 계산합니다.

![image-20210622025947705](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/gam_p1_5.png)

**2. FAST 알고리즘으로 모델에 포함할  fi,jfi,j  선택 후 gradient boosting으로 학습**

앞서 설명했듯이 feature의 수가 많을때, 존재하는 feature pair 조합의 수는 매우 크다. 그러므로, 수많은 feature interaction term 중 어떤 interaction term을 모델에 포함할지 결정하는 과정이 필요하다.

EBM은 FAST 알고리즘을 이용하여 어떤 interaction term을 모델에 포함할지 결정하게된다.

FAST 알고리즘의 철학은 다음과 같다:  
(xi,xj)(xi,xj)feature pair의 interaction이 강하다면,  fi,jfi,j도 낮은 RSS를 보일 것이다.

하지만, 모든  fi,jfi,j를 부스팅 방식으로 학습 시키는 것은 컴퓨팅 비용이 크기 때문에 적절하지 않다.

FAST는  xi,xjxi,xj  평면에 2개의 cut  ci,cjci,cj을 생성하고 각 quadrant에 속하는 데이터의 평균값을 예측값으로 사용하는 간단한 모델을 만들어 부스팅 모델 학습을 대체한다. 이 모델을  Ti,jTi,j(interaction predictor)라고 지칭한다.

![image-20210622033255318](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/fast.png)

Ti,jTi,j  학습은 모든  (ci,cj)(ci,cj)조합을 모두 탐색하여  RSS=Σ(yk−Tij(xk))2RSS=Σ(yk−Tij(xk))2가 가장 낮은 최적  (ci,cj)(ci,cj)를 선택하는 방식으로 진행된다.

가장 낮은  RSSRSS값을 가지는  Ti,jTi,j에 해당하는  (xi,xj)(xi,xj)를 가장 interaction이 높은 feature pair로 간주한다.

이후 interaction이 높은 상위 K개 feature pair를 선택하고. 이전 방식과 동일한 부스팅 방식으로 모델 잔차에 2개 feature를 사용할 수 있는 tree를 학습 시킨다.

_현재 EBM 학습 과정 1,2가 1 iteration을 구성하는지. 1에 대한 iteration이 완료된 후 2에 대한 iteration이 진행되는지 명확하게 확인하지 못했습니다.추후에 추가적으로 확인해보고 위에서 설명한 알고리즘과 다르다는 것이 확인된다면 업데이트하겠습니다._



## 코드 예제

----------


interpretML은 scikit-learn 스타일의 API를 제공하기 때문에, 익숙한 .fit() .predict() method로 간편하게 모델 학습 및 예측을 진행할 수 있습니다.

아래 코드 예제는 interpretML EBM documentation에서 가져왔습니다.

나이, 학력 등 14개 feature로 연소득이 $50,000 이상인지 아닌지를 분류하는 이진 분류 문제입니다.

```javascript
from interpret import set_visualize_provider
from interpret.provider import InlineProvider
set_visualize_provider(InlineProvider())

```

```python
import pandas as pd
from sklearn.model_selection import train_test_split

from interpret.glassbox import ExplainableBoostingClassifier
from interpret import show

df = pd.read_csv(
    "https://archive.ics.uci.edu/ml/machine-learning-databases/adult/adult.data",
    header=None)
df.columns = [
    "Age", "WorkClass", "fnlwgt", "Education", "EducationNum",
    "MaritalStatus", "Occupation", "Relationship", "Race", "Gender",
    "CapitalGain", "CapitalLoss", "HoursPerWeek", "NativeCountry", "Income"
]
df = df.sample(frac=0.05)
train_cols = df.columns[0:-1]
label = df.columns[-1]
X = df[train_cols]
y = df[label]

seed = 1
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.20, random_state=seed)

ebm = ExplainableBoostingClassifier(random_state=seed)
ebm.fit(X_train, y_train)

```

모델 트레이닝이 완료되면 .explain_global() 메소드로 각 feature값이 EBM의 예측값에 어떻게 기여하는지 확인할 수 있습니다.

```makefile
ebm_global = ebm.explain_global()
show(ebm_global)

```

다른 트리 기반 앙상블 모델과 마찬가지로, feature importance를 확인할 수 있습니다.  ![image-20210622035103270](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/ebm_fi.png)


모든 feature에 해당하는  fifi  shape function을 시각적으로 확인할 수 있습니다.  ![image-20210622035128181](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/ebm_f1.png)  
모델에 포함된 interaction term  fi,jfi,j도 시각적으로 확인할 수 있습니다.  ![image-20210622035153494](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/ebm_f2.png)



EBM의 .explain_local() 메소드로 instance의 예측 결과에 대한 설명을 볼 수 있습니다.

```makefile
ebm_local = ebm.explain_local(X_test[:5], y_test[:5])
show(ebm_local)

```

![image-20210622035351644](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/local_ex.png)  
4번째 index 값을 EBM은 0으로 예측했고, 그 예측을 어떻게 했는지 확인할 수 있습니다.

위 그림의 각 막대는 각 shape function에 feature 값을 대입한 연산 결과를 나타냅니다.

EBM regressor에서 모델의 예측값은 위 그림에서 시각화된 막대 값의 합이고.  
EBM Classifier에서 모델의 예측 positive class 확률은 위 그림에서 시각화된 막대 값의 합의 inverse-logit 값입니다.

다른 부스팅 모델과 달리, EBM을 사용하면 모델이 특정 예측 결과에 어떻게 도달했는지 매우 명확하게 확인할 수 있습니다.



## EBM을 활용한 모델 디버깅 사례

----------


![image-123](http://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/sanghyunbaek/debug.png)  간략하게 EBM을 디버깅한 사례를 소개하고 글을 마무리하겠습니다.

위 그래프는 폐렴 사망 여부 예측 EBM 모델링 과정에서 발견되었습니다.

폐렴 사망 여부 예측에 사용된 feature 중 흡입산소분율(FiO2)동맥혈산소분압(PaO2)비율이 모델의 예측 결과에 어떤 영향을 미치는지 확인하고자 해당 shape function 그래프를 확인해보았는데 이상한 점이 발견되었습니다.

보편적으로 폐렴 환자들은 흡입산소분율(FiO2)동맥혈산소분압(PaO2)비율이 낮을 수록 위급한 상태입니다. 하지만, 이상하게도 FiO2/PaO2비율이 300일 때 EBM은 사망 확률이 낮아진다고 예측하다는 점을 발견했습니다.

데이터 전처리 과정을 검토해보니, 데이터셋 FiO2/PaO2비율 feature의 평균값이 300이었고. 데이터의 결측치를 이 평균값으로 채웠다는 사실을 발견했습니다.

즉, 폐렴의 정도가 심하지 않아 병원에서 FiO2/PaO2비율을 굳이 측정하지 않은 환자들의 FiO2/PaO2비율이 300으로 EBM 모델에 주어져서 FiO2/PaO2비율이 300정도 일때 EBM 모델은 사망 확률이 낮아진다고 예측하는 현상이 발생한다는 것을 확인할 수 있었습니다.

이 사례에서 볼 수 있듯이 EBM은 높은 예측 성능과 설명 가능성을 모두 가지고 있는 훌륭한 모델입니다.



## 참고 자료



-   [InterpretML documentation](https://interpret.ml/docs/ebm.html?fbclid=IwAR0P7TwfiWMBpbY1EU7YhDrtjM4wPbpiV-qh211mMoaTK9O4q3bxTSk8VXI#id7)
-   [Yin Lou, Rich Caruana, Johannes Gehrke, and Giles Hooker. Accurate intelligible models with pairwise interactions. In  _Proceedings of the 19th ACM SIGKDD international conference on Knowledge discovery and data mining_, 623–631. 2013.](https://www.cs.cornell.edu/~yinlou/papers/lou-kdd13.pdf)
-   [Y. Lou, R. Caruana, and J. Gehrke. Intelligible models for classification and regression. In KDD, 2012.](https://www.cs.cornell.edu/~yinlou/papers/lou-kdd12.pdf)
-   [InterpretML: A Unified Framework for Machine Learning Interpretability](https://arxiv.org/pdf/1909.09223.pdf)
-   [The Science Behind InterpretML: Explainable Boosting Machine](https://www.youtube.com/watch?v=MREiHgHgl0k)
-   [An introduction to Explainable AI (XAI) and Explainable Boosting Machines (EBM)](https://www.kdnuggets.com/2021/06/explainable-ai-xai-explainable-boosting-machines-ebm.html)
