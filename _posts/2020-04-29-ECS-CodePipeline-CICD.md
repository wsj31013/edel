---
layout: post
title: CodePipeline ECR ECS CI/CD 구현 
color: brown
tags: [CodePipeline, CodeBuild, ECS, CI/CD, AWS]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

### DevOps와 Agile 그리고 CI/CD
```xml
CI/CD에 대해 얘기 하기 전에 DevOps에 대해 먼저 얘기를 하는게 좋을 것 같아서 짧게 언급하려고 한다.
DevOps 라는 단어가 이제는 많이 알려져 있고 실제로 이미 팀과 조직에 도입하고 있는 곳들이 많아 지는 추세다.

DevOps는 그 뿌리가 Agile에 있어서 Agile의 확장판 정도로 볼 수 있겠다. 즉 Agile이라는 큰 우산 아래에
많은 소프트웨어 개발 방법론들이 있고 DevOps도 그 중에 하나라고 보면 된다. 
많은 책들과 팟캐스트, 유튜브 동영상들에 자주 나오는 개념이자 문장 들이다.

여기에 내 생각을 조금 더하자면 Dev와 Ops 사이에 있는 사일로를 깨고 빠른 개발과 빠른 릴리즈를 통해 
비즈니스 기회를 더 확보하고자 하는 것이 DevOps의 철학과 목적에 가장 가깝지 않나 싶다. 
그렇다면 좀 더 빠른 개발과 릴리즈는 어떻게 할건데? 여기에서 그 DevOps를 달성하기 위한 문화적인 부분, 기술적인 부분이 나오는 것이다.

그래서 이 DevOps라는 녀석은 애자일에 근본을 두고 있고 개발 방법론이므로 문화적인 것들과 기술적인 것들이 나눠질 수 있으며
문화적인 것들은 애자일 그 자체다. 애자일을 빼곤 데브옵스를 설명할 수가 없다. 또한 내 경험으로 문화적인 것과 기술적인 것 둘 중에
조직에 적용하기 어렵고 오래 걸리는 것은 기술이 아니라 문화다. 기술은 시간과 돈만 있으면 얼마든지 도입, 적용 가능하지만 문화적인 측면은
돈과 시간으로 해결할 수 없다. 

그리고 요즘 많은 기업에서 DevOps, SRE를 도입하고 그에 맞는 엔지니어를 채용하고 있는데 좀 정확히 알고 채용했으면 좋겠다.
DevOps는 개발과 운영 둘다 할수 있는 그런 잡부(좋게 얘기해서 풀스택)를 의미 한다거나 인프라 엔지니어가 아니다. 

Dev와 Ops에 존재하는 사일로를 깨고 좀 더 빠른 개발과 좀 더 빠른 릴리즈에 기여할 수 있는 애자일 철학과 사상이 몸에 베여 있는 소프트웨어 엔지니어를 의미한다고 생각한다.
마이크로서비스를 개발하기 위해 어떤 프레임워크를 써야 하는데? 컨테이너를 써야 해? 테라폼을 써야 해? 앤서블을 써야 해? 클라우드는 어딜 써야 해?
쿠버네티스를 써야 해? 깃헙 액션을 써야 해? 이런 게 중요한게 아니라는 것이다.

물론 기업들이 위에 언급한 것들을 데브옵스, SRE엔지니어들에게 요구 하는 기술스택들 인 것은 맞으나 저게 먼저가 아니라는 말을 하고 싶었다.
애자일이 뭔지 데브옵스의 철학이 뭔지 사일로를 도대체 왜 깨야 하는지 왜 하필 하고 많은 단어 중에 Dev + Ops 2개의 단어를 결합한건지
DevQA는? DevSecurity는? 아틀라시안 솔루션들을 사용하면 데브옵스 스러워 지는건가?
이런 것들을 고민하는게 훨씬 더 중요하다. 어떤 프레임워크를 쓸지 어떤 툴을 쓸지가 중요한게 아니라는 것이다.

뭐 대충 기술보다 문화가 더 적용하기 어렵다는 그런 소리를 좀 길게 썼다.


기술적인 부분중에 가장 많이 언급되는 게 MSA, IaC, CI/CD, Communication Tools 정도 인데 오늘 다룰 녀석이 CI/CD 이고
CI/CD를 AWS의 코드빌드와 코드파이프라인을 이용해서 ECS에 배포해 볼 것이다.

일반적인(?) CI/CD 라면 각 조직에 맞는 일련의 Git Flow가 있을 것이고 저장소 브랜치에 Merge가 되면
소스를 빌드하고 특정 타겟 인스턴스로 배포 전략(롤링,블루그린,카나리..등)에 맞게 배포 하게 된다.

아래에서 살펴 볼 것은 위에서 얘기한 파이프라인과 거의 동일 하긴 한데 특정 컨테이너 이미지가 ECR(Elastic Container Registry)에
푸시가 되면 트리거 되어 타겟 ECS(Elastic Container Service)로 배포 되는 예제를 살펴 볼 것이다.
이 과정에선 코드커밋,깃헙과 같은 레포지토리가 Source가 되는 것이 아니라 ECR이 소스가 된다.

```

### CodeBuild, CodePipeline
```xml
CodeBuild는 Jenkins, Bamboo와 비슷한 역할을 하는 빌드 도구로서 소스를 빌드하고 그 결과물을 저장하는 역할을 한다.
CodePipeline은 빌드 이후에 Tasks를 진행하게 된다.

ECR에 컨테이너 이미지가 Push -> CodeBuild가 빌드 -> 나머지 ECS에 배포는 CodePipeline
한다고 보면 될 것 같다. CI/CD 파이프라인을 작성 해본 사람들은 여기에서 의문점을 갖게 될텐데
아니 이미 컨테이너 이미지가 빌드가 되었잖아? 근데 왜 빌드를 해? 이게 머선 소리야?? 라고 할수 있는데..

맞다. 정확하게 얘기하면 소스코드 빌드는 아니다.
Source(저장소) -> Build(코드빌드) -> Deploy(코드파이프라인) 이 플로우에서 
Source(저장소) 이게 깃헙이나 코드커밋이 아닌 ECR 이기 때문에 Build Stage를 1개 두어야 ECS에 디플로이를 할수 있다.
아래 buildspec.yml 코드를 보면서 자세히 설명 하겠다.

```


### CodePipeline 파이프라인 생성
#### 1. 서비스 역할 및 아티팩트 S3 설정
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/02.pipeline.png" | absolute_url}})
```xml
파이프라인 이름을 지정하면서 새 서비스 역할을 생성하도록 한다. 
S3는 파이프라인의 각 스테이지에서 생긴 아티팩트들을 저장하는 버켓이므로 기본으로 해도 되고 본인이 원하는 지정위치를 해도 된다.
그냥 기본 위치 설정 하고 넘어가면 된다.
```
#### 2. 소스 스테이지 추가
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/03.source.png" | absolute_url}})
```xml
위에서도 얘기 했지만 
소스 -> 빌드 -> 배포 이 3가지 단계중 소스 단계이며, ECR의 푸시 되어 있는 이미지를 선택한다.
```
#### 3. 빌드 스테이지 추가(1)
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/04.build.png" | absolute_url}})
```xml
빌드는 CodeBuild를 선택하고 프로젝트 생성을 누르면 CodeBuild 프로젝트 생성 팝업창이 뜬다.
```

#### 3. 빌드 스테이지 추가(2)
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/05.build.png" | absolute_url}})
```xml
빌드 프로젝트의 이름을 작성하고
```
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/06.build.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/07.build.png" | absolute_url}})
```xml
코드빌드를 실행할 이미지와 런타임을 지정할 수 있는데 나는 위 사진 처럼 지정했고 이것은 도큐먼트에 더 자세하게 나와 있다.
즉 아래에서 buildspec.yml에 작성된 코드들이 실행될 서버 이미지와 런타임을 설정한다고 보면 된다.
```
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/08.build.png" | absolute_url}})
```xml
buildspec.yml 전체 코드
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.7
  build:
    commands:
      - YOUR_REPOSITORY1_URI=$(cat imageDetail.json | python -c "import sys, json; print(json.load(sys.stdin)['ImageURI'].split('@')[0])")
      - YOUR_IMAGE1_TAG=$(cat imageDetail.json | python -c "import sys, json; print(json.load(sys.stdin)['ImageTags'][0])")
      - echo $YOUR_REPOSITORY1_URI:$YOUR_IMAGE1_TAG
      - cd $CODEBUILD_SRC_DIR_SourceArtifactOcr
      - YOUR_REPOSITORY2_URI=$(cat imageDetail.json | python -c "import sys, json; print(json.load(sys.stdin)['ImageURI'].split('@')[0])")
      - YOUR_IMAGE2_TAG=$(cat imageDetail.json | python -c "import sys, json; print(json.load(sys.stdin)['ImageTags'][0])")
      - echo $YOUR_REPOSITORY2_URI:$YOUR_IMAGE2_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Writing image definitions file...
      - printf '[{"name":"your_container_name1","imageUri":"%s"},{"name":"your_container_name2","imageUri":"%s"}]' $YOUR_REPOSITORY1_URI:$YOUR_IMAGE1_TAG $YOUR_REPOSITORY2_URI:$YOUR_IMAGE2_TAG > $CODEBUILD_SRC_DIR/imagedefinitions.json
artifacts:
    files: imagedefinitions.json
```

```xml
자 이제 위 코드를 설명 하겠다.
이미 컨테이너 이미지가 빌드 된 상태인데 빌드 스테이지를 두고 위의 buildspec.yml을 작성한 이유는 
소스가 깃헙이나 코드커밋이 아닌 ECR 이기 때문이다. 소스가 ECR 일 경우 소스 아티팩트가 imageDetail.json으로 생성되는데
배포를 ECS 표준 배포를 사용한다면 imagedefinitions.json을 배포의 입력으로 사용해야 한다.

즉 빌드 스테이지 에서는 imageDetail.json 아티팩트를 배포에 필요한 imagedefinitions.json 으로 변환하는 작업을 해주는 것이다.
그리고 위의 코드는 2개의 컨테이너를 1개의 ECS 인스턴스에 같이 배포 하고자 위처럼 2개의 이미지가 정의 된다.
1개의 컨테이너 이미지만 배포하는 경우라면 1개만 작성하면 된다. 
```

![my images]({{"/assets/img/thumbnails/codepipeline-ecs/09.build.png" | absolute_url}})
```xml
CodeBuild에서 프로젝트 생성을 마치고 나면 다시 CodePipeline으로 돌아와 다음으로 넘어간다.
```

![my images]({{"/assets/img/thumbnails/codepipeline-ecs/10.deploy-skip.png" | absolute_url}})
```xml
Deploy 스테이지 작성은 잠시 미루고 건너뛰기를 클릭한다.
여기까지 다 되었다면 소스 스테이지에선 1개의 컨테이너 이미지만 소스아티팩트가 생겼고, 빌드에선 2개의 컨테이너 이미지를 buildspec.yml에 작성하였으므로
빌드가 실패하게 된다. 의도한 것이므로 수정을 해주면 된다!
만약 1개의 소스에서 배포 파이프라인을 작성하는것이라면 1개씩만 하면 된다.
```

#### 4. 소스 스테이지 작업 추가
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/14.edit-source02.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/15.edit-source03.png" | absolute_url}})

```xml
2개의 컨테이너 이미지에서 소스 아티팩트가 생성되야 하므로 작업을 추가 한다.
작업 이름과 아티팩트 이름을 다르게 설정 해야 한다.
```

#### 5. 빌드 스테이지 수정
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/17.edit-build02.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/19.release.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/20.build-complete01.png" | absolute_url}})

```xml
소스 스테이지에서 생성된 2개의 아티팩트를 지정하고 변경사항을 릴리즈 해보면 정상으로 빌드까지 성공할 것이다.
세부 정보를 클릭해서 빌드 실행(buildspec.yml) 로그를 확인해보고 아티팩트가 저장되는 S3에서 다운로드 받아
배포에 필요한 imagedefinitions.json 이 정상적으로 문법에 맞게 생성되었는지 확인 해 봐야 한다.
```

#### 6. 배포 스테이지 생성
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/23.deploy-add01.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/24.deploy-add02.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/25.deploy-add03.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/26.deploy-add04.png" | absolute_url}})
```xml
입력 아티팩트를 빌드스테이지에서 생성되는 빌드 아티팩트를 지정하고 배포 타겟 ECS를 지정한다.
이미지 정의 파일을 빌드 아티팩트에서 생성된 imagedefinitions.json 으로 설정 한다.
배포 스테이지까지 생성하고 나서 변경 사항을 릴리즈 하면 ECS로 배포 되는데 아래처럼 자동으로 Tasks Definition이 +1 된다.
```

#### 6. ECS 배포 확인
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/27.ecs-tasks-definition.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/codepipeline-ecs/28.ecs-service-deploy.png" | absolute_url}})




#### 참고한 문서
`codepipeline` : [AWS Document](https://docs.aws.amazon.com/ko_kr/codepipeline/latest/userguide/file-reference.html#pipelines-create-image-definitions)



