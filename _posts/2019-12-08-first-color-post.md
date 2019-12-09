---
layout: post
title: My first Color Post
tags: [Test, AWS, reinvent, LasVegas]
color: brown
author: cobain
excerpt_separator: <!--more-->
---

<!--more-->

## 깃헙 블로그 이사 후 첫번째 글이..리인벤트 후기다.
라스베가스에서 첫 게시글을 쓰게 될 줄 몰랐다. 😉

공항에서 기다리는 시간이 꽤 길어져 첫 게시글을 작성 해 본다.

```yml
올해 처음 참가한 리인벤트 행사에서 처음 출발할때 생각 했던 것 보다 더 많은 인사이트를 얻게 되었다.
리인벤트 행사 전후로 쏟아진다는 표현이 딱 맞을 정도로 많은 기능들이 출시 되었는데
인상적이었던 것은 행사가 시작되기 전날 밤에 진행되는 미드나잇 매드니스에서 딥컴포저가 출시됨을 알렸다고 한다.
사실 나는 라스베가스 기준 월요일 오후에 도착하여 조금 늦게 등록 해서 미드나잇은 유튜브를 통해 보았다.

이번 리인벤트를 AI 관련 서비스로 시작 했다는 것은 개인적으로 나름 의미가 있는 것이라고 생각한다.
나도 Eks on Fargate가 출시되어 New Launch 세션을 들으러 부랴부랴 등록하고 호텔로 이동했는데
이것보다 CodeGuru(코드리뷰 AI 서비스)가 훨씬 더 흥미로운 기능 이었다.. 코드리뷰를 이젠 AWS가 해주겠다고 한다.

데이터사이언스에 대한 학습은 아마 내년부터 많이 하게 될 것 같다.
대학원도 고려 하고 있고..

아무튼 Elasticsearch 관련 기능이나 Managed Cassandra, Kendra, Outpost등 많은 기능들의 출시를 알렸고
AWS 답게 플렉스 했다.

중간 중간에 세션 시간이 빌때 미리 리저브하지 않은 세션들을 들어갔는데 뜻밖의 좋은 세션을 들었다.
최근에 CloudFormation과 AWSCLI를 사용하여 ECS Fargate를 개발쪽에 배포하는 작업을 했었는데
세션 내용은 CloudFormation으로 코드를 작성하고 TaskCat으로 유닛테스트, CodePipeline을 태우는 내용이었다.
또한 Formation StackSet 크로스계정/크로스리전 유즈 케이스들에 대한 내용들도 있었고.

기존에 내가 했던 방식은 CloudFormation 템플릿을 작성하고 바로 CLI로 배포하였는데 이 방식을 개선해서
일련의 GitFlow를 적용하는 방식이다.
굳이 어플리케이션도 아니고, CloudFormation Templates을 유닛테스트에 코드파이프라인 까지 라는 생각을 할 수 있는데
이젠 AWS Resources들이 많은 IaC 코드에 의해 구성, 배포 될 수 있으므로 
인프라스트럭쳐 또한 Git 등을 이용한 버전관리를 하는게 자연스러운 형태라고 생각 한다.
물론 단기간 내에 테라폼이나 앤서블, AWS CDK, CloudFormation 등을 적용하긴 쉽지 않겠지..조직이란게 내맘같지 않아서....

러닝커브는 그다지 높지 않은 편이니 코드로서 모든 것을 관리하자는 취지를 이해 한다면 금방 배울 수 있을 것이다.


```



#### AWS 2019 리인벤트에 대해 더 자세히 보고 싶다면 아래를 참고 
`출시 정보` : [aws 한국 블로그](https://aws.amazon.com/ko/blogs/korea/tag/aws-reinvent-daily/?fbclid=IwAR29vwB6KXsc8h0m0MzaHaJP4NMSGprcCZuzpP2yHTcG6NSMU2fUk4PneO8)




