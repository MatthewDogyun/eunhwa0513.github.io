---
layout: post
title: 데이터 분석가를 위한 소소한 파이썬 tip - 판다스 part 1
subheading: 데이터 분석가를 위한 소소한 파이썬 tip - 판다스 part 1
author: 전지호
categories: 머신러닝
banner:
image: "/assets/images/post/machine.png"
tags: mlops aws kubernetes
sidebar: []
---


# Part 1 | Pandas dataframe 메모리 감소시키기

> 판다스(Pandas)를 좀 더 효율적으로 사용하기 위한 방법들! 이번에는 데이터프레임 메모리를 좀 더 효율적으로 사용하기 위한 방법에 대한 내용입니다 🙂

</br>

판다스(Pandas)는 데이터 분석에서 가장 많이 사용되는 라이브러리입니다.

part 1에서 소개할 방법은 판다스 dataframe 자체 메모리를 감소시키는 것입니다. 굉장히 간단한 방법이지만, 전처리에서 자주 놓치는 부분이기도 합니다.



## 1. Numeric dtype 메모리 감소

----------

</br>

판다스는 기본적으로 numerical data에 64bit를 사용해 dataframe을 생성합니다. (예를 들어, 정수라면 int64, 실수라면 float64)

하지만, 모든 데이터가 64bit를 필요로하지는 않습니다. 데이터 범위가 작을 경우 더 낮은 bit로도 충분히 저장이 가능합니다.

</br>

아래 표는 numerical한 데이터 타입에 대한 설명입니다.

</br>

Data type

Description

Value/Range

bool_

Boolean stored as a byte.

True or False

int8

Byte.

-128 to 127

int16

Integer.

-32768 to 32767

int32

Integer.

-2147483648 to 2147483647

int64

Integer.

-9223372036854775808 to 9223372036854775807

uint8

Unsigned integer.

0 to 255

uint16

Unsigned integer.

0 to 65535

uint32

Unsigned integer.

0 to 4294967295

uint64

Unsigned integer

0 to 18446744073709551615

float16

Half precision float.

sign bit, 5 bits exponent, 10 bits mantissa

float32

Single precision float.

sign bit, 8 bits exponent, 23 bits mantissa

float64

Double precision float.

sign bit, 11 bits exponent, 52 bits mantissa

즉, 데이터 범위에 따라서 다양한 데이터 타입을 사용할 수 있습니다.

사용하고 있는 데이터 범위가 16bit로 충분히 가능한데 64bit를 사용하는 것은 굉장히 비효율적인 메모리 사용이 되겠습니다.

</br>

예시를 보면 더 파악이 빠릅니다.

우선, 임의의 데이터 프레임을 생성해주겠습니다.

```kotlin
num = 10000000

data = pd.DataFrame(
                    {'col1': [1.0  ] * num,
                     'col2': [1    ] * num,
                     'col3': ['AAA'] * num
                    })

```

col1의 경우 실수를, col2는 정수, col3에는 문자열을 넣어주었습니다.

이렇게 생성된 데이터 프레임의 정보는 다음과 같습니다.

```vbnet
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 10000000 entries, 0 to 9999999
Data columns (total 3 columns):
 #   Column  Dtype  
---  ------  -----  
 0   col1    float64
 1   col2    int64  
 2   col3    object 
dtypes: float64(1), int64(1), object(1)
memory usage: 228.9+ MB

```

col1과 col2에 대해 자동으로 float64, int64를 사용하고 있습니다. 총 데이터는 약 230MB 정도입니다.

</br>

이를 각각 16bit와 8bit로 바꾼 후 다시 메모리를 체크해봅시다.

```kotlin
data['col1'] = data['col1'].astype(np.float16)
data['col2'] = data['col2'].astype(np.int8)

```

```vbnet
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 10000000 entries, 0 to 9999999
Data columns (total 3 columns):
 #   Column  Dtype  
---  ------  -----  
 0   col1    float16
 1   col2    int8   
 2   col3    object 
dtypes: float16(1), int8(1), object(1)
memory usage: 104.9+ MB

```

메모리가 105MB정도로 2배 이상 줄어든 것을 확인할 수 있습니다.

이 부분에서 주의하셔야 할 점은  **bit의 범위를 바꿔줄 때 데이터에 따라 정보를 잃을 위험이 있다는 것**입니다. 따라서, 데이터 손실을 원하지 않는다면 각 데이터에 대한 range를 확인한 후 type을 바꿔줘야 합니다.

</br>

또는, downcast를 이용해 range 확인 절차를 자동으로 진행하는 것도 가능합니다.

```kotlin
data['col1'] = pd.to_numeric(data['col1'], downcast='float')
data['col2'] = pd.to_numeric(data['col2'], downcast='integer')

```

단, downcast는 아래와 같은 smallest dtype을 지원합니다.

-   ‘integer’ or ‘signed’: smallest signed int dtype (min.: np.int8)

-   ‘unsigned’: smallest unsigned int dtype (min.: np.uint8)

-   ‘float’: smallest float dtype (min.: np.float32)


</br>

따라서, 다음과 같이 데이터 메모리가 줄어들었습니다.

```vbnet
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 10000000 entries, 0 to 9999999
Data columns (total 3 columns):
 #   Column  Dtype  
---  ------  -----  
 0   col1    float32
 1   col2    int8   
 2   col3    object 
dtypes: float32(1), int8(1), object(1)
memory usage: 124.0+ MB

```

col1의 float type이 32bit까지만 줄어들었기 때문에 float16 dtype보다는 메모리 용량이 크지만, 처음 230MB 보다는 많이 줄어들었습니다.

</br>

## 2. Objective dtype 메모리 감소

----------

</br>

만약, objective type 데이터가 일정한 값이 반복되는 범주형이라면?

**category type으로의 변경을 통해 메모리 효율을 높일 수 있습니다.👍**

</br>

category type은 문자열을 직접 저장하지 않습니다. 문자열의 유일값으로  _Index - 문자열_  을 만들어 놓고 실제 저장소에 index만을 저장하는 형태입니다.

아래와 같이 category 형태로 바꿔봅시다.

```kotlin
data['col3'] = data['col3'].astype('category')

```

바꾼 후의 데이터는 다음과 같습니다.

```vbnet
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 10000000 entries, 0 to 9999999
Data columns (total 3 columns):
 #   Column  Dtype   
---  ------  -----   
 0   col1    float16 
 1   col2    int8    
 2   col3    category
dtypes: category(1), float16(1), int8(1)
memory usage: 38.1 MB

```

col1, col2를 downcast한 후와 비교해봐도 2배 이상의 메모리가 줄어든 것을 알 수 있습니다.

**처음 데이터가 230MB 였다는 것을 생각해보면 거의 20%의 메모리로 동일한 데이터를 처리할 수 있게 되었습니다.**

</br>

----------

</br>

아주 간단한 방법이지만, 빅데이터를 다루는 분석가 입장에서는 놓칠 수 없는 부분이라고 생각합니다. spark와 같은 병렬처리 방법이 있어도 이를 지원하지 않는 라이브러리가 아직 많기 때문입니다.

part 2에서는 판다스에서 데이터 처리 속도를 증가시키는 법에 대해 다뤄보도록 하겠습니다👋
