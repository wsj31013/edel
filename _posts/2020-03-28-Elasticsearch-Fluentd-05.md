---
layout: post
title: AWS Elasticsearch, Fluentd를 활용한 로그 분석 플랫폼 구축 05
image: "assets/img/thumbnails/Fluentd_ES_Data_Flow.png"
color: brown
tags: [Elasticsearch, Fluentd, Data Analysis, log, AWS]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

#### 로그 수집, 분석 플랫폼 구축과 관련된 다섯번째 포스팅이고 이걸로 끝내려고 한다.
#### 이 포스팅에선 Fluent Docker 빌드, ECS Fargate 배포, Secret Manager를 통힌 환경변수 삽입 등에 대해 다룬다.


#### 1. Fluent Docker 
```xml
Fluentd 도커 이미지는 도커허브에 있는 이미지를 가져와 빌드 했고 1월 초에 빌드할 당시 faraday 버그가 있었는데 지금은 픽스 되었는지 모르겠다.
Fargate Fluent에서 Elasticsearch로 데이터를 넣을때 Ruby Rest API 라이브러리(faraday)를 이용하는데 그때 그게 버그가 있어서
최신버전을 사용하진 못했다. 그래서 Dockerfile을 보면 fluent-plugin-aws-elasticsearch-service의 버전을 2.2.0으로 사용했다. 
그리고 onbuild 이미지는 Deprecated 되는 것 같으니 꼭 도커허브 fluent 공식 페이지에서 확인하고 사용하길 바란다.

fluent.conf
<system>
  # prod workers 4
  # dev workers 2
  workers 4
</system>

<source>
  @type  forward
  @id    input1
  @label @mainstream
  port  24224
</source>

# Used for docker health check
<source>
  @type http
  port 8888
  bind 0.0.0.0
</source>

# records sent for health checking won't be forwarded anywhere
<match health**>
  @type null
</match>


<label @mainstream>
<match api-logback**>
  @type "aws-elasticsearch-service"
  logstash_format true
  logstash_prefix fluentd-logback
  type_name fluentd-logback
  include_tag_key true
  tag_key @log_name
  reload_connections false
  reconnect_on_error true
  reload_on_failure true
  <endpoint>
    url "#{ENV['ES_ENDPOINT']}" -> 이 값을 시크릿 매니저에 저장
    region ap-northeast-2
    access_key_id "#{ENV['ACCESS_KEY_ID']}" -> 이 값을 시크릿 매니저에 저장
    secret_access_key "#{ENV['SECRET_ACCESS_KEY']}" -> 이 값을 시크릿 매니저에 저장
  </endpoint>
  <buffer>
    @type file
    path /var/log/td-agent/buffer/logback/aggregator.buffer
    flush_interval 1s
    chunk_limit_size 1m
    flush_thread_interval 0.1
    flush_thread_burst_interval 0.01
    flush_thread_count 15
    total_limit_size 2GB
    overflow_action throw_exception
    flush_at_shutdown true
    retry_max_times 30
    retry_max_interval 1h
  </buffer>
</match>

<match api-sleuth**>
  @type "aws-elasticsearch-service"
  logstash_format true
  logstash_prefix fluentd-sleuth
  type_name fluentd-sleuth
  include_tag_key true
  tag_key @log_name
  time_key api-logging-time
  time_key_exclude_timestamp true
  reload_connections false
  reconnect_on_error true
  reload_on_failure true
  <endpoint>
    url "#{ENV['ES_ENDPOINT']}" -> 이 값을 시크릿 매니저에 저장 
    region ap-northeast-2
    access_key_id "#{ENV['ACCESS_KEY_ID']}" -> 이 값을 시크릿 매니저에 저장 
    secret_access_key "#{ENV['SECRET_ACCESS_KEY']}" -> 이 값을 시크릿 매니저에 저장
  </endpoint>
  <buffer>
    @type file
    path /var/log/td-agent/buffer/sleuth/aggregator.buffer
    flush_interval 1s
    chunk_limit_size 1m
    flush_thread_interval 0.1
    flush_thread_burst_interval 0.01
    flush_thread_count 15
    total_limit_size 2GB
    overflow_action throw_exception
    flush_at_shutdown true
    retry_max_times 30
    retry_max_interval 1h
  </buffer>
</match>
</label>


Dockerfile
FROM fluent/fluentd:v1.3-onbuild-1

USER root

RUN apk add --update --virtual .build-deps \
        sudo build-base ruby-dev \
 && sudo gem install \
        fluent-plugin-aws-elasticsearch-service -v 2.2.0 \
 && sudo gem install \
        fluent-plugin-grok-parser \
 && sudo gem sources --clear-all \
 && apk add curl \
 && apk del .build-deps \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem

빌드가 끝나고 나면 ECR로 푸쉬 해둔다.
```

#### 2. AWS Secret Manager, Fargate, Capacity Provider 설명
```yml
Fargate ECS는 서버리스 컨테이너 서비스다. 작년부터 프로덕션에 ECS로 서비스를 구축한 것들이 몇개 있었고 서버리스 형태로 구현해도 될 정도의 기술 노하우가
쌓였다는 판단하에 서버리스 컨테이너를 도입 해 봤다. 
지금 내가 근무하고 있는 회사에서 ECS나 서버리스 컨테이너를 처음 도입 한게 나다.
보통 처음 ECS나 EKS를 도입하려고 고민하는 사람들은 EC2 기반으로 해야 될지 서버리스(Fargate)로 해야 할지 무조건 고민을 하게 되어 있는데
내가 추천하는 것은 먼저 EC2기반의 ECS,EKS를 사용해보고 어느정도 기술, 운영 노하우가 쌓인 이후에 서버리스로 넘어가는 것을 추천한다.

그리고 작년 12월에 ECS capacity provider가 릴리즈 되서 이걸 이용해서 EC2 기반의 ECS를 다른 서비스에 구축 해 보았는데, ECS 클러스터 오토스케일링이 정말 빠르다. 
약 1~2분대에 스케일아웃이 완료 된다. 기존에 EC2나 Beanstalk 기반에서 스케일 아웃이 되려면 최소 5분은 잡아야 되는데
Capacity Provider가 적용되지 않은 ECS 클러스터의 경우 Service 레벨에서의 오토스케일과 EC2 오토스케일을 구성 해야 한다.
근데 이때 문제가 스케일아웃 트리거 되어도 프로비저닝된 EC2가 없을 경우 스케일아웃 되는 시간은 기존의 빈스톡이나 EC2 오토스케일과 다르지 않다.
이러면 컨테이너기반의 장점들이 무색해지기에 아마도 아마존에선 Capacity Provider라는 ECS 클러스터의 오토스케일 서비스를 런칭하지 않았을까 생각해 본다.
Capacity Provider는 따로 포스팅 하려고 한다.

먼저 데이터의 흐름이
Beanstalk Fleunt -> ECS Fargate Fluent -> Elasticsearch 이 과정에서
ECS,ES의 엔드포인트, ES의 액세스키,시크릿키 값들을 Secret Manager에 저장해놓고 그 값을 Beanstalk과 Fargate에서 가져 오게 구현 하였다.

값을 외부에 저장해놓고 안전하게 가져 오고 싶다면 AWS에서는 2개의 서비스를 제공한다.
시크릿 매니저와 파라미터 스토어 2개의 서비스가 있는데 시크릿 매니저를 먼저 테스트 했고, 잘 되서 파라미터 스토어는 테스트 하지 않았다.

Fluent는 설정 파일에 환경 변수를 지원하고, Docker도 env 파라미터를 지원하니 ECS에서도 당연히 되지 않을까? 라는 단순한 생각해 찾아봤고
역시 ECS에서도 환경변수 값을 주입 할수 있게 지원한다.

또한 시크릿 매니저를 통해 먼저 값을 저장하고 그 값을 빈스톡 내부에서 호출 해 오도록 파이썬 함수를 하나 만들었고
아마존 개발자들이 시크릿매니저 캐싱 라이브러리를 공개 해놔서 캐싱까지만 구현해 놨다. 
여길 참고 해라. 
https://github.com/aws/aws-secretsmanager-caching-python

시크릿 매니저를 사용하게 되면 각 런타임별로 예제코드 들이 제공되니 그걸 참고해서 본인이 필요한 코드를 짜길 바란다.

시크릿 매니저를 사용 할 IAM 권한의 경우 
Beanstalk에서 시크릿 매니저를 콜 하고 값을 가져올 때 -> aws-elasticbeanstalk-ec2-role 에 secret manager policy를 생성 후 연결 한다.
ECS에서 시크릿 매니저를 콜 하고 값을 가져올 때 -> ecsTaskExecutionRole 에 secret manager policy를 생성 후 연결 한다.

```

#### 3.  AWS Secret Manager Caching 예제 코드(Beanstalk ebextensions에 requirements.txt와 함께 넣는다.)
```python
import boto3
import base64
from aws_secretsmanager_caching import SecretCache, SecretCacheConfig
from botocore.exceptions import ClientError
import datetime

now = datetime.datetime.now()
nowDate = now.strftime('%Y%m%d-%H:%M:%S')

def get_secret():
    secret_name = "LOGS_FARGATE_ENDPOINT"
    region_name = "ap-northeast-2"
    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager', region_name=region_name)

    try:
        cache = SecretCache(SecretCacheConfig(), client)
        get_secret_value_response = cache.get_secret_string(secret_name)
        with open('/etc/sysconfig/td-agent', 'w') as env:
            env.write('export ' + 'FARGATE_ENDPOINT=' + get_secret_value_response)
    except ClientError as e:
        if e.response['Error']['Code'] == 'DecryptionFailureException':
            raise e
        elif e.response['Error']['Code'] == 'InternalServiceErrorException':
            raise e
        elif e.response['Error']['Code'] == 'InvalidParameterException':
            raise e
        elif e.response['Error']['Code'] == 'InvalidRequestException':
            raise e
        elif e.response['Error']['Code'] == 'ResourceNotFoundException':
            raise e
    else:
            secret = get_secret_value_response
            with open('/root/venv/secret-manager-result.log', 'a') as result:
                result.write(nowDate + " >> " + secret + '\n')


if __name__ == "__main__":
    get_secret()
```

#### 4. Fargate Deploy
```yml
Fargate를 배포하는 것은 CloudFormation과 AWSCLI 2개의 조합으로 사용했고 참고할 만한 매우 훌륭한 템플릿이 있다.
여길 참고 해라. https://github.com/aws-samples/aws-fargate-log-aggregator-with-fluentd
나도 위 내용을 보고 참고 했다.
Cloudformation을 개인적으로 좋아하진 않는데 AWS는 CloudFormation을 백엔드에 사용하는 서비스가 굉장히 많다...

1. Cluster 생성
aws ecs create-cluster --cluster-name fargate-fluentd --profile profile

2. 배포
aws cloudformation deploy --template-file prod-fargate-fluent.yml --stack-name aggregator-service \
--parameter-overrides EnvironmentName=fluentd-aggregator-service \
DockerImage=111111.dkr.ecr.ap-northeast-2.amazonaws.com/prod-custom-fluentd:v1.3 \
VPC=vpc-111111 Subnets=subnet-111111,subnet-111111 Cluster=fargate-fluentd ExecutionRoleArn=arn:aws:iam::111111:role/ecsTaskExecutionRole \
MinTasks=2 MaxTasks=4 --capabilities CAPABILITY_NAMED_IAM --profile profile

```

#### 5. Fargate Secret Manager
##### 배포가 끝난 후 Tasks Definition을 아래 처럼 업데이트 하면 된다.
![my images]({{"/assets/img/thumbnails/fargate-secret-manager/fargate-secret-manager.png" | absolute_url}})



#### 마지막 정리 글
```yml
작년 연말 즈음 부터 AWS Fargate, Fluent, Elasticsearch를 활용한 로그 데이터 수집과 관련된 포스팅을 끝내려고 한다.
데이터가 다 수집된 이후 키바나에서 시각화 하는 것과 Elasticsearch CBE 처리 등과 같은 것들도 더 업데이트 하고 싶었는데
연재 형식으로는 안하고 나중에 따로 포스팅을 하려고 하는데 과연..
Elasticseach Curator, Lambda Function, Cognito kibana 등 작업 한 것들이 꽤 많다.
로그 누락되는 케이스도 있어서 정제 처리 부분을 수정 했던 적도 있었고..

사용된 모든 코드들이나 작업하며 남긴 스크린샷들을 모두 공개 하고 싶지만 외부로 유출되면 안되는 정보가 포함된 것들이 꽤 있어서 
다 공개하진 못했다.

이미 AWS 기술 블로그에는 Fargate, Fluent, Kinesis로 구성된 데이터 파이프라인 구축과 관련된 포스팅이 있다.
나도 처음에 이 글을 읽고 인사이트를 얻어 도입하게 되었고 작업을 하다보니 Fluent가 굉장히 가볍고, 다양한 옵션들로 
굳이 키네시스나 카프카를 넣지 않고도 버퍼 큐의 역할을 충분히 할 수 있겠다 싶었다.

그리고 Beanstalk에 Client Fluent를 넣었는데 이게 Fluent-bit가 아님에도 굉장히 가벼워서
모니터링 해보니 평균 0.1~0.3 정도의 CPU를 소비 하는 것을 확인 할 수 있었다.

Fleunt가 K8s와 같은 CNCF 재단에 소속된 오픈소스라서 그런지는 모르겠지만 
K8s, Prometheus등과 같이 사용되는 케이스가 점점 많아 지는 것 같다.
K8s에서 데몬셋으로 Fluent를 띄워놓고 모든 로그 데이터들을 Elasticsearch로 보내도 되고.


굉장히 가볍고 강력한 성능을 갖고 있는 Fluent
서버리스 컨테이너 오케스트레이션 Fargate
데이터 분석 플랫폼 Elasticsearch, Kibana

이 3가지를 활용한 로그 플랫폼 구축 포스팅은 여기서 끝.

```


##### 다른 글을 읽고 싶다면..
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

