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
여러 멀티 라인의 


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