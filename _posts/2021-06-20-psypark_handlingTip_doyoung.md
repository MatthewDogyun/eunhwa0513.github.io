---
layout: post
title: 실전 Pyspark 중급편 - 분석가를 위한 핸들링 팁
subtitle: 실전 Pyspark 중급편 - 분석가를 위한 핸들링 팁
author: 전지호
# 김도영 | Data analyst
categories: 머신러닝
tags: [docker,kubernetes,jupyter,aws]
---
# 스파크 데이터프레임 핸들링, 이럴땐 어떻게?

> 단순한 pyspark function은 쉽게 학습하여 사용할 수 있지만, 그것만으로는 부족할 때가 있다.  
> 그럴 때마다 참새처럼 스택오버플로우를 들락날락하게 되는데, 그 과정에서 얻은 소소한 팁을 정리해두려 한다.




## 다수의 column에 반복작업하기

----------

다수의 column에 같은 function을 적용할 때에는 아래와 같은 방법으로 해결할 수 있다.  
개인적으로는 특정 method를 반복할 때에는 for loop이, agg 안에서 같은 함수를 반복할 때에는 list comprehension이 편했다.

([이 medium post](https://mrpowers.medium.com/performing-operations-on-multiple-columns-in-a-pyspark-dataframe-36e97896c378)의 내용을 옮김)


### reduce

정석적인 방법이지만, 파이써닉하지 못하다.  
첫번째 파라미터로는 함수가 들어가게 된다. (아래처럼 람다일 수도 있고, 정의된 함수일수도 있음)  
두번째 파라미터는 iterable한 형태의 데이터, 세번째 파라미터는 초기값이다. 초기값을 원본 df로 설정해준다.

```lua
actual_df = (reduce(
    lambda memo_df, col_name: memo_df.withColumn(col_name, lower(F.col(col_name))),
    source_df.columns,
    source_df
))

```




### for loop

코드는 못생겼어도, 스파크는 똑똑해서 reduce와 같은 plan을 생성해낸다고 한다.

```lua
actual_df = source_df

for col_name in actual_df.columns:
    actual_df = actual_df.withColumn(col_name, lower(col(col_name)))

```




### list comprehension

파이써닉하여 널리 쓰이고 있는 방법이다.

```csharp
actual_df = source_df.select(
    *[lower(F.col(col_name)).alias(col_name) for col_name in source_df.columns]
)

```






## timestamp 연산

----------

데이터를 가공하다보면 timestamp 타입을 다뤄야 하는 경우가 많았다. 편리한 2가지 function을 소개한다.




### timestamp 연산 - unixtime

timstamp를 unix timestamp로 변환하여 정수로 만든 뒤 계산하면 편리하다.  
unix timestamp는 1970년 1월 1일부터의 경과시간을 초 단위로 표현한 것으로, 당연히 밀리초 단위는 생략된다. 굳이 필요하다면 원본 timestamp에서 밀리초 단위를 가져온 후, 소수점 아래의 값으로 붙이면 표기 가능하다.

```bash
df_1 = (df
        .withColumn('unix_timestamp_1',  F.unix_timestamp('timestamp_1'))
        .withColumn('unix_timestamp_2',  F.unix_timestamp('timestamp_2'))
        # 초 단위를 일 단위로 변환
        .withColumn('diff_daynum', (F.col('unix_timestamp_2') - F.col('unix_timestamp_1'))/86400)

```




### timestamp 연산 - expr, INTERVAL

PySpark expr()는 SQL function으로, SQL-like expressions을 실행하여 데이터프레임 열의 값을 변환하는 방법이다. 굳이 타입변환을 하지 않아도 간단한 연산이 가능하다는 장점이 있다.

```ini
df = df.withColumn('testing_time', df.testing_time + F.expr('INTERVAL 2 HOURS')) 
# INTERVAL은 SQL에서 제공하는 기능이다.

```






## 가장 빈번하게 등장하는 값 찾기

----------

특정 column에서 가장 빈번하게 등장하는 값(최빈값)을 찾는 udf 함수를 정의하여 사용해보자.  
mode 함수 자체는 collections 모듈의 Counter 함수와 most_common 메소드를 사용하여 쉽게 만들 수 있다.  
collect_list 함수는 특정 id를 기준으로 여러 개의 값들을 배열 형식으로 묶어주는 함수로, 해당 배열에 mode함수를 적용하는 것으로 이해할 수 있다.

```python
import pyspark.sql.functions as F
@F.udf
def mode(x):
    from collections import Counter
    return Counter(x).most_common(1)[0][0]

cols = ['A1', 'A2', 'A3']
agg_expr = [mode(F.collect_list(col)).alias(col) for col in cols]
df_1 = df.groupBy('id').agg(*agg_expr)

```






## Window 함수로 누적 연산, 랭킹, 간격 구하기

----------

Window 함수를 사용하면 (그룹 간 비교가 아닌) 그룹 내 비교를 통한 연산을 쉽게 수행할 수 있다.  
먼저 window를 정의하고, 해당 window 내에서 수행될 함수를 적용해보자.

([데이터브릭스 팀에서 만든 이 문서](https://databricks-prod-cloudfront.cloud.databricks.com/public/4027ec902e239c93eaaa8714f173bcfc/968100988546031/157591980591166/8836542754149149/latest.html)의 내용을 참고함)




### 누적 연산

누적 연산은 특정 함수를 지정한 window 내에 적용하는 식으로 계산할 수 있다. orderBy로 지정한 순서대로, 특정 순서까지의 모든 값에 대한 연산이 리턴된다.  
누적합, 누적평균, 누적 최대(최소)값 등을 구할 때 유용하다.

```sql
from pyspark.sql import Window as W

cumulative_window_1 = W.partitionBy('partition').orderBy('cumulative')

df_cumulative_1 = (df
                   .select('partition', 'cumulative')
                   .withColumn('cumulative_sum', F.sum('cumulative').over(cumulative_window_1))
                  )

```




### 랭킹

누적 연산과 같은 방식으로 window를 정의하여 다양한 형태의 rank를 계산할 수 있다.

```sql
ranking_window = W.partitionBy('partition').orderBy('ranking')

df_ranks = df.select(
  'partition', 'ranking'
).withColumn(
  # note that fn.row_number() does not take any arguments
  'ranking_row_number', F.row_number().over(ranking_window) 
).withColumn(
  # rank will leave spaces in ranking to account for preceding rows receiving equal ranks
  'ranking_rank', F.rank().over(ranking_window)
).withColumn(
  # dense rank does not account for previous equal rankings
  'ranking_dense_rank', F.dense_rank().over(ranking_window)
).withColumn(
  # percent rank ranges between 0-1 not 0-100
  'ranking_percent_rank', F.percent_rank().over(ranking_window)
).withColumn(
  # fn.ntile takes a parameter for now many 'buckets' to divide rows into when ranking
  'ranking_ntile_rank', F.ntile(2).over(ranking_window)
)

```




### 간격

column 내 t기와 t+1기 사이의 간격(difference with previous row)을 계산할 수 있다.  
특정 window 안에서 한 칸씩 밀린 값(lag)을 생성하고, coalesce 함수로 lag으로 인해 발생한 공백을 제거하여 차이를 계산하는 방식이다.

```sql
diff_window = W.partitionBy('partition').orderBy("order_col")

df_1 = (df
        .withColumn("diff",
        (F.col('date').cast('int') - F.coalesce(F.lag(F.col("date").cast('int')).over(diff_window))) / 3600)
      )

```






## Lambda를 활용한 간단한 udf 생성

----------

특정 column에 pyspark function으로 정의되지 않은 간단한 변환을 적용하고 싶을 때에는 udf를 사용하는 방법이 있다.  
lambda를 활용하여 간단히 udf를 정의하고 사용해보자.  
아래 예시에서는 timestamp에서 시간 값만을 추출하는 간단한 udf를 만들어 적용해보았다.

```makefile
# 리턴할 값의 타입을 정확히 정의하는 것이 중요하다.
hour = F.udf(lambda x: x.hour, T.IntegerType())
# withColumn을 사용하여 변환된 column을 간단히 생성한다.
df_1 = (df
        .withColumn("hour_1", hour("timestamp_1"))
       )

```






## 원하는 값의 리스트 생성 - collect

----------

filter 함수의 조건, null값을 채우는 용도 등으로 특정 value를 확인해야 하는 경우가 있다.  
collect()를 사용하면 해당 데이터프레임은 리스트 형태로 변환되고, 리스트 안의 값들은 row로 인덱싱을 활용하여 원하는 값을 쉽게 확인할 수 있다.

```ini
# 내가 원하는 정보가 있는 컬럼
col_list = ['a', 'b', 'c']
col_name = 'a'

# unique value를 구하기 위해 distinct 사용, collect 함수로 리스트로 변환
list_raw = df.select(col_list).distinct().collect()
# 특정 컬럼의 값만 들어있는 리스트 생성
value_list = [list_raw[n]['a'] for n in range(len(list_raw))]

```






## 상위 n퍼센트 값 구하기 - expr, percentile_approx

중위값(median)을 구하거나 특이값을 제거할 기준치를 정할 때 유용한 방법이다.  
timestamp 연산에서와 마찬가지로 expr를 사용하여 원하는 연산을 적용한다.

```bash
# percentile_approx(연산대상이_되는_열, 하위_n퍼센트_값)
df_1 = (df
        .groupBy('a', 'b')
        .agg(F.expr('percentile_approx(c, 0.5)').alias('c_median'),
       )
```
