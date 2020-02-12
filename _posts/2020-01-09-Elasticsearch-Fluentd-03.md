---
layout: post
title: AWS Elasticsearch, Fluentd를 활용한 로그 분석 플랫폼 구축 03
image: "assets/img/thumbnails/Fluentd_ES_Data_Flow.png"
color: brown
tags: [Elasticsearch, Fluentd, Data Analysis, log, AWS]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

##### 로그 수집, 분석 플랫폼 구축과 관련된 세번째 포스팅 이며, 이전 단계에서 ElasticSearch를 프로비저닝 하였고 
##### 이번 포스팅 에선 Index Templates, Mapping 그리고 이 아키텍처의 핵심인 Fluentd를 다룹니다.

```yml
인덱스는 RDBMS의 DB랑 비슷하고, Mapping은 Schema와 비슷하다. 똑같진 않지만 처음엔 이렇게 비교하는게 Elasticsearch 입문자에겐 편한 것 같다.
대략적인 순서는 이렇다.
1. Dynamic Mapping을 이용해 로그 데이터를 Elasticsearc에 보내본다.
2. Dynamic Mapping에 의해 생성된 Mapping을 확인한다.
3. 내가 원하는 Mapping을 설계하고 Index templates을 만든다.
4. Index Templates Mapping에 맞게 Fleuntd로 데이터를 쪼갠다. (3번과 4번은 바뀔수도 있겠다.)
5. Elasticsearch에 저장되는 데이터를 Kibana로 시각화 한다.

Elasticsearch는 Dynamic Mapping 이라는걸 지원하는데 사전에 잘 설계된 Mapping 없이 프로덕션 레벨에서 운영할 수는 없다.
데이터가 유실이 될 수도 있고, Fleuntd나 Logstash에 의해 정제된 Key Value 형태의 도큐먼트들을 Elasticsearch에 잘 쌓는 것은 더할 나위 없이 중요하다.

첫번째 포스팅에 있는 아키텍처대로 어플리케이션 노드(이하 Client Fluentd), 로그 수집기(이하 Aggregator Fluentd) 둘다 Fluentd의 패키지 형태인
td-agent를 이용했고, Client Fluentd의 경우 ebextensions(AWS Beanstalk) 소스 번들에 패키징 하였으며 
Aggregator Fluentd의 경우 AWS ECS Fargate로 구성 하였다.

꽤 많은 삽질을 하였는데 차례대로 정리하다 보니 별거 없다.
Client Fluentd, Aggregator Fluentd의 설정 파일들과 인덱스템플릿, 매핑을 보면서 설명 하겠다.
```

```yml
1. Client Fluentd
자바 어플리케이션의 로그를 수집하는 케이스이며 Fluentd 공식 홈과 Fluentd Plugin Github등을 참고 하였다.
또한 로그를 파싱할때 Fluentd는 아래의 3가지의 방법을 제공하는데 3가지 모두를 사용 하게 되었다.
- Grok Pattern
- Regex Expression
- Ruby

로그 형식을 보면 2가지로 나눌수 있었고 1가지는 Grok, 나머지 1개는 Regex로 파싱했다.
Ruby의 경우 Grok Pattern 파싱 이후에 타임존이 Null 값 이어서 그 부분만 Ruby 문법을 이용했다.
이 타임존 문제 때문에 나중에 Kibana에서 쿼리를 하면 데이터가 정상적으로 조회는 되는데 대쉬보드에서 시간차이가 발생하는 문제가 생기는데 이것 때문에 거의 하루 정도를 삽질 했다.
여담이지만 키바나 타임존 설정 바꿔도 안되니 그거 바꾸면 되는거 아니냐 라는 생각은 하지 말자.
Sleuth가 적용된 로그 패턴은 아래와 같았으며 Grok Pattern으로 파싱 하였고 Grok Debug 사이트(https://grokdebug.herokuapp.com/)를 이용하였다.
```

```yml
- 로그 패턴
[2019-11-25 18:21:43.457]  INFO [-,,,] 419 --- [http-nio-8080-exec-1] o.s.w.s.DispatcherServlet                : Initializing Servlet 'dispatcherServlet'

아래처럼 생각하면 된다.
[시간] 로그레벨 [Sleuth service,trace,span,exportable] PID --- [스레드] 클래스 : 로그 내용


- td-agent.conf
<source>
  @type tail
  path /var/log/tomcat8/api/*.log
  pos_file /var/log/tomcat8/api/api.pos
  tag api-sleuth.Logname
  <parse>
    @type multiline_grok
    grok_pattern \[%{TIMESTAMP_ISO8601:api-logging-time}\]\s+%{LOGLEVEL:level}\s+\[%{DATA:service},%{DATA:trace},%{DATA:span},%{DATA:exportable}\]\s+%{DATA:pid}\s+---\s+\[\s*%{DATA:thread}\]\s+%{DATA:class}\s+:\s+%{GREEDYDATA:message}
    multiline_start_regexp /\d{4}-\d{1,2}-\d{1,2}/
    time_key api-logging-time
  </parse>
</source>

<filter *.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<filter *.**>
  @type record_transformer
  enable_ruby true
  <record>
    api-logging-time ${time.strftime('%Y-%m-%dT%H:%M:%S.%3N%z')}
  </record>
</filter>

<match *.**>
  @type forward
  <buffer>
    @type file
    path /var/log/td-agent/buffer/sleuth/sleuth.buffer
    chunk_limit_size 1m
    total_limit_size 1GB
    flush_interval 5s
    retry_max_times 5
    retry_max_interval 10s
    flush_at_shutdown true
    overflow_action throw_exception
  </buffer>
  <server>
    host "#{ENV['FARGATE_ENDPOINT']}"
    port 24224
  </server>
</match>
```


```yml
Fluentd는 Input, Parse, Filter, Buffer, Output, Formatter 등으로 나눠지며
데이터를 입력받는 Input, 정제 역할의 Parse, Filter, 버퍼와 큐잉 역할을 하는 Buffer, 데이터를 저장소로 보내는 Output 정도가 되겠다.
Parse나 Filter는 필수는 아니고 개인의 필요에 따라 설정 하면 된다.

그렇다면 위의 설정 파일을 보면...
첫 번째 source 섹션에서 Grok Pattern으로 파싱 처리 한 부분이며 아래와 같이 최초 플러그인 설치가 필요하다
# td-agent-gem install fluent-plugin-grok-parser
pos_file은 Fluentd에서 로그를 읽어 들일때 어디까지 읽어 들였는지 기록하는 파일이 pos_file 이다. 
```

```yml
두번째 섹션의 filter는 Client Fluentd, 즉 로그를 보내는 서버들이 다수의 로드밸런서 안에서 분산처리 되므로 호스트네임 정도는 구분을 하기 위해 추가한 값이며

세번째 섹션의 filter는 첫번째 섹션에서 파싱처리를 할때 Timezone이 없어 Elasticsearch로 보내고 난 뒤 키바나 Discovery 탭에서 데이터 조회시
UTC, KST타임의 문제가 있어서 추가한 섹션이다.
밑에서 Aggregator Fluentd에서 설명을 하겠지만 어차피 logstash_format 값을 true로 설정하면 Elasticsearch에서 인덱스가 날마다 로테이션 되게 할수 있는데
이때 키바나에서는 @timestamp 메타 필드로 인덱스 패턴을 생성하는게 일반적이다.
근데 나같은 경우 첫번째 섹션에서 파싱처리 하고 난 뒤의 api-logging-time 이라는 값으로 인덱스 패턴을 생성할때 Time Filter field를 @timestamp가 아니라
api-logging-time으로 하고 싶었다. 아래처럼 말이다.

{
  "_index": "fluentd-sleuth-2020.01.08",
  "_type": "_doc",
  "_id": "AgPyg28BPl8w4aTOVyGj",
  "_version": 1,
  "_score": null,
  "_source": {
    "api-logging-time": "2020-01-08T15:58:14.785+0900",
    "level": "DEBUG",
    "service": "-",
    "trace": "8167933928ca13f1",
    "span": "8167933928ca13f1",
    "exportable": "false",
    "pid": "3841",
    "thread": "http-nio-8080-exec-2",
    "class": "c.e.a.l.v.h.s.HealthService",
    "message": "cobain"
    "hostname": "눈뜨고코베인",
    "@log_name": "api-sleuth.logs"
  },
  "fields": {
    "api-logging-time": [
      "2020-01-08T06:58:14.785Z"
    ]
  }
}

인덱스 패턴의 Time filter를 @timestamp로 만들게 되면 이렇게 된다. 
  "fields": {
    "@timestamp": [
      "2020-01-08T06:58:14.785Z"
    ]
  }

그렇다면 @timestamp는 이건 뭘까? @timestamp는 Logstash에 의해 메타 데이터 필드로 자동으로 생성된는 필드이며
이 시간은 Logstash에 의해 이벤트가 처리된 시간을 말한다. 
Elasticsearch 안에 쌓인 Document를 보면 api-logging-time이 실제 어플리케이션 로깅 시간이며 @timestamp는 
Logstash에 의해 이벤트가 처리된 시간임을 확인 할수 있으며
둘다 있어도 상관 없지만 api-logging-time 필드 1개를 갖고 인덱스 패턴을 만들고 싶었고 내가 정의한 매핑에 필드 하나가 더 들어오는게 싫었다.
물론 2가지 모두 테스트 해봤고 있거나 없거나 아직까진 문제가 되는 부분이 없다. 얘기가 길어졌는데 다음 섹션으로 넘어가자.
```

```yml
세번째 filter 섹션은 api-logging-time이라는 Key에 Timezone을 추가한 섹션이다. Grok 패턴으로 파싱한 이후에 stdout 지시자로
데이터 형태를 확인하고 Aggregator나 Elasticsearch로 전달 하는 과정을 거쳤었는데 그때 타임존이 없다는걸 처음엔 몰랐다.
키바나에서 데이터를 조회하다 보니 나중에 알게되서 추가한 섹션이다.
```

```yml
마지막 match 섹션이 output 부분인데 Aggregator Fluentd로 보내는 설정이다. 여기에서 버퍼 설정을 줄 수 있는데
내가 Fluentd를 사용하면서 정말 잘 개발된 아키텍처구나 라고 느꼈던게 이 버퍼 부분 이었다.

데이터가 누락없이 수집되도록 하기 위해 Fluentd는 정말 세심한 큐처리와 재송신 로직을 처리할 수 있는데 서버의 리소스가 무한정이 아닌이상
이 버퍼 섹션을 적절하게 시행착오를 겪으며 설정 하게 된다. 아마도 이건 절대 값 이라는 건 존재하지 않을 거다.

match 지시자 에서 외부 통신하게 되는 Output 플러그인은 BufferOutput(비동기)과 TimeSlicedOutPut(지정한 시간) 2가지가 있고
만약 네트워크 지연이나 장애가 발생하여 처리가 실패 했을때 Fluentd는 이를 버퍼에 담았다가 재시도 하게 된다.
버퍼는 메모리, 파일 타입이 있으며 리소스의 제한과 다루는 로그들의 크기등을 고려해서 설정하는게 좋다. 

스터디한 내용을 좀더 적자면
Chunk라고 하는 큐에 이벤트를 추가함
Chunk 크기 또는 시간 한계에 도달하게 되면 Chunk 버퍼를 플러쉬 하게 됨
만약 플러쉬가 실패할 경우 그 실패한 청크는 큐에 남고 retry_wait 지시자에 의해 그 시간만큼 기다렸다가 Chunk 플러쉬를 재시도 함
또 실패 할 경우 성공할 때까지 재시도 시간(retry_wait가 2배씩 늘어남)을 늘려가며 재시도 하게 됨
이 재시도 횟수는 retry_limit 만큼 시도하며 재시도 횟수 한계에 도달하면 그 청크는 폐기 된다. 이 재시도를 무한정 계속 하게 할수도 있다.(disable_retry_limit)

BufferOutput(비동기)는 Chunk를 플러쉬 하는 지정시간(flush_interval) 또는 제한(buffer_queue_limit) 2개의 상한선 중에 어느 쪽이든 먼저 그 설정 값에 도달 하는 경우 동작한다.

TimeSlicedOutPut(지정한 시간)은 flush_interval을 사용하지 않고 time_slice_format, time_slice_wait를 사용하게 되며
time_slice_format 으로 지정된 시간 단위로 Chunk 버퍼를 만들고 time_slice_wait로 지정한 10분 내에 지연되어 도착한 레코드는 이전 Chunk 버퍼에 추가 된다. 
time_slice_wait 지정한 시간이 지나게 되면 이전의 Chunk 버퍼를 파일등으로 출력한다.
```

##### 쓰다보니 너무 길어져 나머지 Regex 파싱 처리와 Aggregator Fluentd, Index templates 등은 다음 포스팅으로 넘깁니다.


##### 다른 글을 읽고 싶다면..
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>



[참고한 서적 링크 : 데이터 분석 플랫폼 구축과 활용](http://www.kyobobook.co.kr/product/detailViewKor.laf?ejkGb=KOR&mallGb=KOR&barcode=9791161340302&orderClick=LAG&Kc=)