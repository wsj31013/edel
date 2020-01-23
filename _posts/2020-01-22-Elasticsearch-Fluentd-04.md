---
layout: post
title: AWS Elasticsearch, Fluentd를 활용한 로그 분석 플랫폼 구축 04
image: "assets/img/thumbnails/Fluentd_ES_Data_Flow.png"
color: brown
tags: [Elasticsearch, Fluentd, Data Analysis, log, AWS]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

##### 로그 수집, 분석 플랫폼 구축과 관련된 네번째 포스팅이며 이전 단계에서 자바 어플리케이션의 Fluentd Grok Pattern 파싱 처리와
##### Elasticsearch 인덱스와 매핑에 대해 간단히 살펴봤다. 이번 포스팅에는 Fluentd Regex 파싱 처리와 인덱스템플릿을 살펴보겠다.


```yml
Grok Pattern으로 데이터들을 정제하는 과정에 대한 설명은 이전 포스팅에서 진행했고 Fluentd의 버퍼쪽도 살펴봤다.
정규식으로 테스트할 때는 https://regex101.com 이 사이트에서 테스트하면서 짜면 된다. 이 사이트를 이용하는 방법이 간단해서 한 두번만 해보면
바로 가능할 것이다.
Fluentd 문서(https://docs.fluentd.org/parser/multiline) 에는 자바, 레일스의 로그들을 multiline으로 파싱하는 예제들이 있으니 참고 하면 됨.

바로 설정 파일들을 살펴 보자.

```


```yml
- td-agent.conf
<source>
  @type tail
  path /var/log/tomcat8/api/*.log
  pos_file /var/log/tomcat8/api/api.pos
  tag api-logback.Logname
  multiline_flush_interval 10s
  <parse>
    @type multiline
    format_firstline /\d{4}-\d{1,2}-\d{1,2}/
    format1 /^\[(?<time>\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2}.\d{1,3})\] *(?<level>[^\s]+) (?<pid>\d+) --- \[(?<thread>.*?)\] (?<class>\S+)\s*:(?<message>.*)/
  </parse>
</source>


<filter *.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

<match *.**>
  @type forward
  retry_limit 5
  <server>
    host Aggregator Fargate ELB
    port 24224
  </server>
</match>

```


```yml
설정 파일을 보면 Grok 패턴으로 처리한 것과 유사한 방식으로 진행했고, 이전 포스팅에서 설명한 것과 중복되는 것들이 많아서
다른 부분만 설명하겠다.

multiline을 parser로 사용할때는 format_firstline, formatN 지시자를 입력해야 한다.
여러 멀티 라인의 로그들을 파싱 해야 하므로 firstline에서 시작라인을 감지하고, formatN 으로 나머지 로그들을 처리하게 된다.
여기는 많이 설명할게 별로 없다.
아 그리고 정규표현식이 생각보다 컴퓨팅 리소스를 잡아먹을수 있으니 Client Fluentd(어플리케이션 노드)에서 파싱 처리 할지
Aggregator Fluentd에서 처리 할지는 성능 테스트를 해보고 진행 해야 한다.
Fluentd 로그 집계 때문에 어플리케이션에 지연이 발생하면 안되니까.
```



##### 다음은 Elasticsearch index templates에 대해 잠깐 정리하면
```yml
인덱스 템플릿의 경우 인덱스에서 사용하게 될 매핑을 미리 선언 해둔 템플릿이다.
로그스태쉬 포맷으로 일자별로 인덱스가 생성되므로 템플릿을 미리 작성해 두고 새로 인덱스가 생성될때마다 해당 템플릿의 매핑대로 만들어지게 된다.
데이터 타입의 매핑만 템플릿으로 만드는게 아니라 샤드 개수, 리플리카등 세팅 정보도 설정할수 있다.

로그성 데이터들이 엘라스틱서치에 들어오게 될때 가장 고민했던게 텍스트 타입을 "text" vs "keyword" 에서 고민을 많이 했고
이건 나만 고민하는 건 아닌거 같다.
성능이슈도 있고 keyword인 경우 로그가 길어져서 누락되는 경우도 있고..

일단 나는 2가지를 모두 테스트 해봤고, 이 필드로 검색과 집계를 어떻게 이용할지에 대해서는 사용자마다 조금씩 다를 것이다.
해당 필드를 어떻게 사용해서 인사이트를 얻을지에 대해 결정을 하고 둘다 테스트 해봐야 한다. 

쉽게 되는게 없어...

```


##### 다른 글을 읽고 싶다면..
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>



[참고한 링크 1 : Elasticsearch index tempaltes](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html)

[참고한 링크 2 : Date datatype ](https://www.elastic.co/guide/en/elasticsearch/reference/current/date.html)