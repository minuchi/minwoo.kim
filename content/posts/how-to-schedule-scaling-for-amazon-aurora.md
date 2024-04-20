---
author: minwoo.kim
categories:
  - AWS
  - Aurora
  - RDS
date: 2024-04-20T07:59:15.424Z
tags:
  - AWS
  - RDS
title: 'Amazon Aurora Auto Scaling을 이용하여 원하는 시간에 스케줄링하기'
cover:
  image: '/assets/img/aws-smile.jpg'
  alt: 'AWS Smile'
  relative: false
showToc: true
---

Amazon RDS Aurora MySQL, PostgreSQL은 Auto Scaling을 현재 지원하고 있다.  
현재 해당 글을 쓰는 시점으로 CPU 사용률 또는 Connection 수로 Scaling을 할 수 있다.

하지만 원하는 시간에 DB를 Scaling 하려면 어떻게 해야 할까?  
Amazon RDS 콘솔에서는 보이지 않지만, `aws application-autoscaling` CLI를 이용하면 설정할 수 있다.

## 사전 조건

우선 Amazon RDS Aurora MySQL 또는 PostgreSQL을 사용하고 있어야 한다.  
Auto Scaling이 설정되어 있다면 상관없지만 걸려있지 않다면 아래 명령어를 이용하여 MinCapacity, MaxCapcity를 설정해 준다.  
만약 Auto Scaling이 설정되어 있다면 해당 명령어는 실패한다.

- `DB_CLUSTER_NAME`: Scaling 이 필요한 DB cluster 이름
- `MIN_CAPACITY`: 최소 Scaling 수 (기본적으로 0 을 사용하는 경우가 많을 것이다.)
- `MAX_CAPACITY`: 최대 Scaling 수
- `AWS_REGION`: 해당 DB cluster가 존재하는 AWS Region

```bash
aws application-autoscaling register-scalable-target \
    --service-namespace rds \
    --scalable-dimension rds:cluster:ReadReplicaCount \
    --resource-id cluster:$DB_CLUSTER_NAME \
    --min-capacity $MIN_CAPACITY \
    --max-capacity $MAX_CAPACITY \
    --region $AWS_REGION
```

성공 시 아래와 같이 응답이 온다.


```json
{
    "ScalableTargetARN": "arn:aws:application-autoscaling:ap-northeast-2:<AWS_ACCOUNT_ID>:scalable-target/<SCALABLE_TARGET_ID>"
}
```

위와 같이 응답이 오면 정상적으로 등록이 된 것이다.


### Scalable Target 목록 확인

성공적으로 Scalable Target이 등록된지 확인할 방법으로는 아래 명령어를 수행하는 방법이 있다.

- `DB_CLUSTER_NAME`: Scaling 이 필요한 DB cluster 이름
- `AWS_REGION`: 해당 DB cluster가 존재하는 AWS Region

```bash
aws application-autoscaling describe-scalable-targets \
    --service-namespace rds \
    --resource-ids cluster:$DB_CLUSTER_NAME \
    --region $AWS_REGION
```

위 명령어로 아래와 같이 존재하는 Scalable Target을 확인할 수 있다.

```json
{
    "ScalableTargets": [
        {
            "ServiceNamespace": "rds",
            "ResourceId": "cluster:<DB_CLUSTER_NAME>",
            "ScalableDimension": "rds:cluster:ReadReplicaCount",
            "MinCapacity": 0,
            "MaxCapacity": 4,
            "RoleARN": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/aws-service-role/rds.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_RDSCluster",
            "CreationTime": "2024-03-04T18:23:48.048000+09:00",
            "SuspendedState": {
                "DynamicScalingInSuspended": false,
                "DynamicScalingOutSuspended": false,
                "ScheduledScalingSuspended": false
            },
            "ScalableTargetARN": "arn:aws:application-autoscaling:ap-northeast-2:<ACCOUNT_ID>:scalable-target/<SCALABLE_TARGET_ID>"
        }
    ]
}
```

## Scaling 설정

### Scale-Out 설정

원하는 시간에 Scaling 설정을 할 때는 [AWS Cron Expression](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cron-expressions.html)을 사용하여 설정할 수 있다.  
설정하는 명령어는 아래와 같다.

- `SCHEDULE_ACTION_NAME`: 해당 Action의 Uniq한 이름으로 겹치면 안 된다.
- `DB_CLUSTER_NAME`: Scaling 이 필요한 DB cluster 이름
- `MIN_CAPACITY`: 최소 Scaling 수 (1 이상으로 해야 Scale Out이 된다.)
- `MAX_CAPACITY`: 최대 Scaling 수
- `AWS_REGION`: 해당 DB cluster가 존재하는 AWS Region
- `CRON_EXPRESSION`: [AWS Cron Expression](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cron-expressions.html)를 이용하여 시간을 설정할 수 있다.
- `TIMEZONE`: Timezone 값. Default는 `UTC` 이다.


```bash
aws application-autoscaling put-scheduled-action \
  --service-namespace rds \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --scheduled-action-name $SCHEDULE_ACTION_NAME \
  --resource-id cluster:$DB_CLUSTER_NAME \
  --schedule "$CRON_EXPRESSION" \
  --scalable-target-action MinCapacity=$MIN_CAPACITY,MaxCapacity=$MAX_CAPACITY \
  --timezone "$TIMEZONE" \
  --region $AWS_REGION
```

예제는 아래와 같다.

```bash
# ap-northeast-2에 존재하는 database-1 cluster를
# 매일 한국 시간 기준 오전 8시 30분에 최소 2개 이상 Scaling 되도록 하는 설정이다.
# 다른 Auto Scaling 조건 (CPU, Connection) 에 의해 더 조절되면 4개까지 Scale out 될 수 있다.
aws application-autoscaling put-scheduled-action \
  --service-namespace rds \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --scheduled-action-name ScaleUpAurora \
  --resource-id cluster:database-1 \
  --schedule "cron(30 8 ? * * *)" \
  --scalable-target-action MinCapacity=2,MaxCapacity=4 \
  --timezone "Asia/Seoul" \
  --region ap-northeast-2
```


### Scale-In 설정

Scale-Out 설정과 동일하게 진행하면 된다.

예제는 아래와 같다.

```bash
# ap-northeast-2에 존재하는 database-1 cluster를
# 매일 한국 시간 기준 오후 6시 30분에 최소 Scaling 개수를 0개로 만드는 설정이다.
# 이렇게 되면 최소 개수를 0으로 변경하여 Scale-In이 되도록 설정할 수 있다.
# 다른 Auto Scaling 조건 (CPU, Connection) 에 의해 더 조절되면 4개까지 Scale out 될 수 있다.
aws application-autoscaling put-scheduled-action \
  --service-namespace rds \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --scheduled-action-name ScaleDownAurora \
  --resource-id cluster:database-1 \
  --schedule "cron(30 18 ? * * *)" \
  --scalable-target-action MinCapacity=2,MaxCapacity=4 \
  --timezone "Asia/Seoul" \
  --region ap-northeast-2
```

## 스케일링을 설정한 Action 목록 조회

설정된 목록을 보고싶다면 아래의 명령어를 수행한다.

- `DB_CLUSTER_NAME`: Scheduled Action이 존재하는 DB cluster 이름
- `AWS_REGION`: 해당 DB cluster가 존재하는 AWS Region

```bash
aws application-autoscaling describe-scheduled-actions \
    --service-namespace rds \
    --resource-id cluster:$DB_CLUSTER_NAME \
    --region AWS_REGION
```

성공 응답 예시는 아래와 같다.

```json
{
    "ScheduledActions": [
        {
            "ScheduledActionName": "ScaleDownAurora",
            "ScheduledActionARN": "arn:aws:autoscaling:ap-northeast-2<ACCOUNT_ID>:scheduledAction:<ID>resource/rds/cluster:<DB_CLUSTER_NAME>:scheduledActionName/ScaleDownAurora",
            "ServiceNamespace": "rds",
            "Schedule": "cron(30 18 ? * * *)",
            "Timezone": "Asia/Seoul",
            "ResourceId": "cluster:<DB_CLUSTER_NAME>",
            "ScalableDimension": "rds:cluster:ReadReplicaCount",
            "ScalableTargetAction": {
                "MinCapacity": 0,
                "MaxCapacity": 4
            },
            "CreationTime": "2024-04-20T08:53:59.134000+09:00"
        },
        {
            "ScheduledActionName": "ScaleUpAurora",
            "ScheduledActionARN": "arn:aws:autoscaling:ap-northeast-2<ACCOUNT_ID>:scheduledAction:<ID>resource/rds/cluster:<DB_CLUSTER_NAME>:scheduledActionName/ScaleUpAurora",
            "ServiceNamespace": "rds",
            "Schedule": "cron(30 8 ? * * *)",
            "Timezone": "Asia/Seoul",
            "ResourceId": "cluster:<DB_CLUSTER_NAME>",
            "ScalableDimension": "rds:cluster:ReadReplicaCount",
            "ScalableTargetAction": {
                "MinCapacity": 1,
                "MaxCapacity": 4
            },
            "CreationTime": "2024-04-20T16:43:06.950000+09:00"
        }
    ]
}
```

## 스케일링을 설정한 Action 삭제

- `DB_CLUSTER_NAME`: 제거할 Scheduled Action이 존재하는 DB cluster 이름
- `SCHEDULE_ACTION_NAME`: 제거할 Scheduled Action 의 이름
- `AWS_REGION`: 해당 DB cluster가 존재하는 AWS Region

```bash
aws application-autoscaling delete-scheduled-action \
    --service-namespace rds \
    --scalable-dimension rds:cluster:ReadReplicaCount \
    --resource-id cluster:$DB_CLUSTER_NAME \
    --scheduled-action-name $SCHEDULE_ACTION_NAME \
    --region $AWS_REGION
```

## 마무리

이렇게 Scaling 추가, 목록 조회 및 삭제에 대해 알아봤다.  
UI에 없어서 해당 기능에 모르고 있을 수 있지만, AWS Blog 등을 확인해 보면 알 수 있다!  
ECS 스케줄링 또한 이렇게 되어있는데, 평소에 AWS Blog를 유심히 살펴보자.

## 참조

- {{< newtabref href="https://aws.amazon.com/blogs/database/schedule-scaling-for-amazon-aurora-replicas-using-aws-application-auto-scaling/" title="Schedule scaling for Amazon Aurora replicas using AWS Application Auto Scaling" >}}
- {{< newtabref href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cron-expressions.html" title="Cron expressions reference (English)" >}}
- {{< newtabref href="https://docs.aws.amazon.com/ko_kr/eventbridge/latest/userguide/eb-cron-expressions.html" title="cron 표현식 참조 (한국어)" >}}
