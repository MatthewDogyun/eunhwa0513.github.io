---
layout: post
title: AWS Aurora DB, Mysql DB에서 스케줄러 사용하는 방법
subtitle: AWS에서 사용하는 RDB 데이터베이스인 AuroraDB에서 스케줄러를 사용하는 방법을 설명합니다.
author: 전지호
categories: 카테고리2
tags: [ws,sql,mysql]
---

# AWS AuroraDB, Mysql DB 에서 스케줄러 사용하는 방법 🕘

  

> AWS AuroraDB에서 데이터 사용을 용이하게 하기 위해 스케줄러를 사용해봅시다.

  

AWS에서 사용하는 RDB 데이터베이스인 AuroraDB에서 스케줄러를 사용하는 방법을 설명합니다.

굉장히 많은 데이터가 일반 RDB에 쌓일 경우, 불가피하게 속도에 영향을 끼칠 수 있습니다.

한 예로, 1 번의 쿼리로 데이터를 삭제하기 너무 많은 시간이 필요할 수 있습니다.

스케줄러를 적절히 사용하면 데이터 관리를 용이하게 할 수 있을 것입니다.

  

## 이벤트 스케줄을 생성하는 기본 문법 ✏️

----------

먼저 기본 문법을 알아봅시다. workbench 등을 통해 접근하여 다음과 같은 문법으로 이벤트를 생성할 수 있습니다.

```vbnet
CREATE EVENT IF NOT EXISTS [이벤트 명]
    ON SCHEDULE
        [수행, 반복 할 시간]
    ON COMPLETION NOT PRESERVE
    ENABLE
    COMMENT [코멘트]
    DO 
    [수행할 명령]

```

  

## 실제 쿼리를 작성해봅시다. ✏️

----------

일단, 매 시간마다 데이터를 삭제하는 예제 쿼리를 작성해보겠습니다.

```sql
CREATE EVENT IF NOT EXISTS EVENT_DELETE_ROW_BEFORE_180DAY
    ON SCHEDULE
        # EVERY 1 MINUTE 1분 마다
        EVERY 1 HOUR
    ON COMPLETION NOT PRESERVE
    ENABLE
    COMMENT '현재일로 부터 180일 전 데이터는 삭제'
    DO 

    # 수행할 이벤트
    DELETE FROM [테이블 명] WHERE date < date_sub(NOW(), INTERVAL 180 DAY) LIMIT 100

```

  

## 스케줄러 On/Off

----------

쿼리 작성을 완료했다면, 해당 데이터베이스가 스케줄러가 작동가능한 지 확인해봅시다.

```sql
SHOW VARIABLES
WHERE VARIABLE_NAME = 'event_scheduler'

```

![이벤트 확인](https://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/batteryho/%EC%8A%A4%EC%BC%80%EC%A4%84%EB%9F%AC.JPG)

이벤트 확인

작동이 가능하지 않다면 아래 명령어를 통해 스케줄러 기능을 사용가능하게 바꿔주세요.

```vbnet
SET GLOBAL event_scheduler = ON;

```

AWS에서는 AuroraDB를 처음 만들었을 때, 파라미터 그룹(Parameter groups)이라는 설정값에 의해 스케줄러가 on/off를 할 수 없는 상태로 생성됬을 수도 있습니다.

AWS console에서 확인해 봅시다.

데이터 베이스의 파라미터 그룹을 확인해봅시다.

![데이터 베이스 확인](https://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/batteryho/%ED%8C%8C%EB%9D%BC%EB%AF%B8%ED%84%B0%EA%B7%B8%EB%A3%B9.JPG)

데이터 베이스 확인

default.aurora5.6라는 기본 파라미터 그룹으로 생성된 것을 알 수 있습니다.

![파라미터 그룹 확인](https://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/batteryho/%ED%8C%8C%EB%9D%BC%EB%AF%B8%ED%84%B0%EA%B7%B8%EB%A3%B91.JPG)

파라미터 그룹 확인

이벤트 스케줄러가 변경이 가능하도록 되어있으나, 기본 파라미터 그룹은 값 변경이 되지 않습니다. 그렇기 때문에, 새 파라미터 그룹을 생성하고 데이터베이스 파라미터 그룹을 변경해야 합니다.

![파라미터 변경](https://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/batteryho/%ED%8C%8C%EB%9D%BC%EB%AF%B8%ED%84%B0%20%EB%B3%80%EA%B2%BD.JPG)

파라미터 변경

1.  새로운 파라미터 그룹을 생성했다면 파라미터를 변경 후, 데이터베이스 파라미터 그룹을 생성한 파라미터 그룹으로 변경한다.  
    
2.  변경이 가능한 파라미터 그룹이라면 파라미터만 변경한다

적용 후, 데이터베이스 재부팅을 통해 파라미터가 변경이 적용되게 하세요.

![재부팅](https://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/batteryho/%EC%9E%AC%EB%B6%80%ED%8C%85.JPG)

재부팅

재부팅이 완료된 후, 다시 명령어로 확인해봅니다.

```sql
SHOW VARIABLES
WHERE VARIABLE_NAME = 'event_scheduler'

```

  

![파라미터 변경 완료](https://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/batteryho/%ED%8C%8C%EB%9D%BC%EB%AF%B8%ED%84%B0%20%EB%B3%80%EA%B2%BD%20%EC%99%84%EB%A3%8C.JPG)

파라미터 변경 완료

이제 이벤트 스케줄러를 등록할 수 있습니다.

  

## 이벤트 스케줄러 등록 📅

----------

예제 쿼리를 실행합니다.

```sql
CREATE EVENT IF NOT EXISTS EVENT_DELETE_ROW_BEFORE_180DAY
    ON SCHEDULE
        # EVERY 1 MINUTE 1분 마다
        EVERY 1 HOUR
    ON COMPLETION NOT PRESERVE
    ENABLE
    COMMENT '현재일로 부터 180일 전 데이터는 삭제'
    DO 

    # 수행할 이벤트
    DELETE FROM [테이블 명] WHERE date < date_sub(NOW(), INTERVAL 180 DAY) LIMIT 100

```

등록된 이벤트는 다음 명령어로 확인할 수 있습니다.

```sql
SHOW EVENTS;

```

![등록된 이벤트 보기](https://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/batteryho/%EB%93%B1%EB%A1%9D%EB%90%9C%20%EC%9D%B4%EB%B2%A4%ED%8A%B8%20%EC%8A%A4%EC%BC%80%EC%A4%84%EB%9F%AC.JPG)

등록된 이벤트 보기

등록된 이벤트를 삭제하려면 다음 명령어로 삭제할 수 있습니다.

```csharp
DROP event [이벤트 명];

```

  

그럼 20,000 👋