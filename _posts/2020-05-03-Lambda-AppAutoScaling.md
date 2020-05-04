---
layout: post
title: Lambda Function을 활용한 AWS Application AutoScaling 자동화
color: brown
tags: [Lambda, ECS, ApplicationAutoScaling]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

### 1.Application Auto Scaling
```xml
AWS의 오토스케일링 하면 EC2 기반의 오토스케일링을 떠올리게 되는데 이번 포스팅에서는 Application AutoScaling을 
람다 함수를 이용해 자동화한 것에 대해 포스팅 하려고 한다.

요구 사항은 Working Time에만 구동하도록 ECS 클러스터의 EC2 인스턴스를 특정 시간에 스케일링 동작이 필요 했다. 
ECS에서는 'Scheduled Tasks'를 제공 해줘서 이걸 이용하려고 했다가 테스트 해보니 Scheduled Tasks는
Tasks개수 조건 >= 1 이라서 0으로 줄일수가 없어서 AWS API를 찾아 보았고 람다 함수를 작성 하게 되었다.

이 포스팅에 사용된 코드를 사용하면 Capacity Provider가 적용된 ECS Cluster의 인스턴스를
ECS Service 레벨에서 특정 시간에 스케일링 동작을 컨트롤 할 수 있게 된다.

이게 무슨 말이냐면 Capacity Provider가 적용 된 ECS의 경우 미리 프로비저닝 된 인스턴스가 없더라도
Application AutoScaling API를 통해 EC2 인스턴스의 스케일 인/아웃이 가능 하다.
그러므로 EC2 AutoScaling API가 아닌 Application AutoScaling API를 이용해 람다 함수를 작성 하는 것이다.
```

### 2.Lambda가 사용할 IAM Policy, Roles 생성
```xml
IAM Policy를 생성하고 Roles에 Policy를 연결 한다.
```
![my images]({{"/assets/img/thumbnails/application-autoscaling/iam/02.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/application-autoscaling/iam/03.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/application-autoscaling/iam/04.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/application-autoscaling/iam/05.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/application-autoscaling/iam/06.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/application-autoscaling/iam/07.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/application-autoscaling/iam/08.add_passrole.png" | absolute_url}})
```xml
CloudWatchLogs Roles는 람다 실행 로그를 Watch Logs로 스트리밍 하기 위해 같이 넣어준다.
PassRole은 나중에 추가해 주었는데 저걸 추가 해주지 않으면 람다 함수 실행시 PassRole 에러로 실행이 되지 않으니 같이 추가해 준다.
```
### 3.Lambda 함수 작성
```xml
런타임은 python 최신 버전인 3.8을 선택하고 미리 만들어둔 IAM Roles로 설정 한다.
```
![my images]({{"/assets/img/thumbnails/application-autoscaling/lambda_function/01.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/application-autoscaling/lambda_function/02.png" | absolute_url}})

### 4.Lambda 코드
```python
'''
Lambda ENV
SCALABLE_DIMENSION = 'ecs:service:DesiredCount'
SERVICE_NAMESPACE = 'ecs'
ROLEARN = 'arn:aws:iam::youraccount:role/aws-service-role/ecs.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_ECSService'
위 3가지 변수는 람다 환경변수로 지정 하였고 Python에서 람다 환경변수를 사용하기 위해선 os 모듈이 필요하다.
샘플 코드 이므로 본인의 환경에 맞게 수정이 필요할 수도 있다.
'''
import boto3
import os

client = boto3.client('application-autoscaling')
MILEAGE_RESOURCE_IDS = [
    'your ecs resource ids of list'
]
RESOURCE_IDS = [
    'your ecs resource ids of string'
]

def mileage_get_current_min_scalable_target():
    response = client.describe_scalable_targets(
        ServiceNamespace = os.environ['SERVICE_NAMESPACE'],
        ResourceIds = MILEAGE_RESOURCE_IDS,
        ScalableDimension = os.environ['SCALABLE_DIMENSION']
    )
    return response["ScalableTargets"][0]["MinCapacity"]

def mileage_start_scalable_target():
    MinCapacity = mileage_get_current_min_scalable_target()
    if MinCapacity == 0:
        return True
    else:
        return False

def lambda_handler(event, context):
    if mileage_start_scalable_target():
        response = client.register_scalable_target(
            ServiceNamespace = os.environ['SERVICE_NAMESPACE'],
            ResourceId =  RESOURCE_IDS[0],
            ScalableDimension = os.environ['SCALABLE_DIMENSION'],
            RoleARN = os.environ['ROLEARN'],
            MinCapacity = 1,
            MaxCapacity = 3,
        )
        print(response)
    else:
        response = client.register_scalable_target(
            ServiceNamespace = os.environ['SERVICE_NAMESPACE'],
            ResourceId = RESOURCE_IDS[0],
            ScalableDimension = os.environ['SCALABLE_DIMENSION'],
            RoleARN = os.environ['ROLEARN'],
            MaxCapacity = 0, 
            MinCapacity = 0,
        )
        print(response)
```



### 5.CloudWatch Event Trigger
![my images]({{"/assets/img/thumbnails/application-autoscaling/lambda_function/03.watch_event_down.png" | absolute_url}})
```xml
Cloud Watch Event의 경우 크론탭 형식으로 람다 함수를 트리거 할 수 있게 지원하므로 위 스크린샷을 참고하여 
UTC KST 시간 차이를 고려하여 Down(Scale in) / Up(Scale out) 2개의 Event Rule을 작성 해 주면
Working Time에만 동작하는 람다 함수가 완성 된다.
```

#### 참고한 문서
`API ApplicationAutoScaling` : [AWS API Documents](https://docs.aws.amazon.com/autoscaling/application/APIReference/API_RegisterScalableTarget.html)

`Boto3 ApplicationAutoScaling` : [Boto3 API Documents](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/application-autoscaling.html#ApplicationAutoScaling.Client.register_scalable_target)

`PassRole` : [Add PassRole Documents](https://docs.aws.amazon.com/ko_kr/iot/latest/developerguide/pass-role.html)