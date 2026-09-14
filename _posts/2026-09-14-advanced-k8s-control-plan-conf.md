---
layout: post
title: Amazon EKS 고급 컨트롤 플레인 구성
color: brown
tags: [EKS, Kubernetes, Control Plane, AWS]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

2026년 8월 12일 Amazon EKS가 [Advanced Kubernetes control plane configuration](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-configuration.html)을 출시했다. 그동안 EKS가 관리해 온 kube-scheduler, kube-controller-manager, kube-apiserver 파라미터 5가지를 클러스터 전역으로 직접 설정할 수 있다.

기존 `CreateCluster` / `UpdateClusterConfig`에 컴포넌트별 필드가 추가된 형태다. Kubernetes 1.31 이상 신규·기존 클러스터에서 지원하고, EKS가 있는 상용 리전에서 추가 요금은 없다. 파라미터를 명시하기 전까지 기존 클러스터 동작은 바뀌지 않는다.

---

### 설정 가능한 파라미터

| 컴포넌트 | 파라미터 | 허용 범위 | 기본값 | PCP 필요 |
| --- | --- | --- | --- | --- |
| kube-scheduler | `nodeResourcesFit.scoringStrategy` | `LeastAllocated`, `MostAllocated` | `LeastAllocated` (cpu:1, memory:1) | 아니요 |
| kube-controller-manager | `horizontalPodAutoscalerSyncPeriod` | 10s ~ 15s | 15s | 예 |
| kube-controller-manager | `terminatedPodGcThreshold` | 10000 ~ 12500 | 12500 | 예 |
| kube-apiserver | `eventTtl` | 10m ~ 60m | 60m | 아니요 |
| kube-apiserver | `serviceNodePortRange` | min/max 각 10260 ~ 32767 | 30000 ~ 32767 | 아니요 |

기본값·허용 범위는 버전마다 달라질 수 있다. IaC에 하드코딩하지 말고 `DescribeClusterVersions`의 `defaultValue` / `constraints`를 조회하는 편이 안전하다.

검증된 범위 밖 값은 거절되고, 변경은 CloudTrail에 기록된다. 네임스페이스·워크로드 단위 적용은 불가하다.

---

### kube-scheduler — scoringStrategy

스케줄러 filter 단계는 그대로며 바뀌는 것은 score 단계의 노드 선호 전략 뿐이다.

- **LeastAllocated** (기본): 할당량이 적은 노드를 선호해 워크로드를 분산
- **MostAllocated**: 이미 채워진 노드를 선호해 파드를 모은다. consolidation 노드 풀과 맞으면 빈 노드를 걷어 컴퓨트 비용을 줄일 수 있다

업스트림의 `RequestedToCapacityRatio`는 지원하지 않는다.

`resources`에는 cpu, memory와 가속기 3종(`nvidia.com/gpu`, `aws.amazon.com/neuron`, `aws.amazon.com/neuroncore`)을 가중치 1~100으로 넣을 수 있다. 배열을 지정하면 **나열한 리소스만** 점수에 들어간다. 영향만 줄이려면 낮은 weight로라도 남겨야 한다. 가속기 가중치는 해당 리소스를 `requests`로 선언한 파드에만 적용되고, DRA 리소스는 이 점수화와 무관하다.

전략 변경은 이후 스케줄링에만 영향을 준다. 이미 떠 있는 파드는 eviction/재시작이 필요하다. `MostAllocated`는 blast radius도 같이 모은다. EKS Auto Mode·Karpenter의 노드 프로비저닝/제거 방식 자체는 바꾸지 않는다.

---

### kube-controller-manager — HPA sync period / 종료 파드 GC

둘 다 [Provisioned Control Plane](https://docs.aws.amazon.com/eks/latest/userguide/eks-provisioned-control-plane.html)(PCP)이 필수다. Standard 모드에서 설정하면 실패한다. 컨트롤 플레인 자원 소비를 키우는 파라미터를 사전 할당 용량이 있는 클러스터로 제한한 것이다.

**HPA sync period**는 각 HorizontalPodAutoscaler를 다시 계산하는 주기. 15s → 10s로 줄이면 스케일링이 더 빨리 시작되지만, 같은 큐를 처리할 시간이 짧아져 지원 가능한 HPA 객체 수는 약 1/3 감소한다. EKS는 객체 수를 검증하지 않고, 한도를 넘겨도 알람·이벤트가 없다. 증상은 "오토스케일링이 기대보다 느려짐"이다. 조정 전 `kubectl get hpa --all-namespaces`로 개수를 확인한다.

**terminatedPodGcThreshold**는 `Succeeded`/`Failed` 파드가 이 개수를 넘으면 GC가 삭제를 시작한다. GC 주기는 20초 고정. 임계값을 낮추면 다음 주기부터 가장 오래된 종료 파드를 즉시 강제 삭제한다. Job/CronJob이 주 기여자다. 특정 Job만 제어하려면 Job의 `ttlSecondsAfterFinished`를 쓰는 편이 맞다.

PCP에서 Standard로 돌아가려면 두 파라미터를 먼저 기본값(15s, 12500)으로 되돌려야 한다. Standard etcd 한도는 8GB라, 사용량이 이를 넘어도 복귀가 막힌다.

---

### kube-apiserver — eventTtl / serviceNodePortRange

**eventTtl**은 이벤트 보존 기간. 기본 60분에서 줄이는 방향만 열려 있다. 배치·CI/CD·잦은 CronJob처럼 이벤트가 etcd를 잠식할 때 검토한다. 만료 시점은 생성 시점에 정해지므로 기존 이벤트는 원래 TTL대로 빠지고, 스토리지 감소는 점진적이다. 삭제된 이벤트는 복구할 수 없다.

**serviceNodePortRange**는 NodePort(기본 구성에선 LoadBalancer도 이 범위) 할당 포트. 하한 10260은 kubelet(10248)·kube-proxy(10256)를 피하고, 상한 32767은 Linux ephemeral port(보통 32768~)와의 충돌을 피하기 위한 값이다. 기존 서비스는 할당받은 포트를 유지한다. 삭제 후 재생성하면 범위 밖 포트는 다시 받을 수 없다.

---

### 운영 규칙

- 리셋 오퍼레이션은 없다. 필드를 생략하면 현재 값이 유지된다. 기본값으로 되돌리려면 `DescribeClusterVersions`로 조회한 뒤 명시적으로 설정한다.
- 업데이트는 merge. 지정한 필드만 바뀌고, `describe-cluster`가 실행 중인 구성 전체를 돌려준다.
- 변경은 즉시 반영되지 않는다. 컨트롤 플레인 롤링 업데이트(`ControlPlaneComponentConfigUpdate`)로 수 분에 걸쳐 적용된다. `describe-update`로 추적하고, 완료 후 클러스터는 `ACTIVE`로 돌아온다.

```bash
aws eks update-cluster-config --name "$CLUSTER" \
  --kube-scheduler-config '{"nodeResourcesFit":{"scoringStrategy":{"type":"MostAllocated"}}}'

aws eks wait cluster-active --name "$CLUSTER"

aws eks describe-cluster-versions --cluster-versions 1.35 \
  --query 'clusterVersions[0].controlPlaneComponentConfig'
```

출시 시점 지원: Console, eksctl, AWS CLI, EKS API, CloudFormation, CDK. ACK·Terraform은 준비 중이다.

---

### 언제 바꿀까

EKS가 넣어 둔 기본값은 대부분 워크로드에서 그대로 써도 되는 값이다. 파라미터를 바꾸면 비용·반응 속도는 나아질 수 있지만 blast radius, 디버깅 윈도우, PCP 과금 같은 트레이드오프가 따라오므로, 아래 상황에 해당하고 목적이 분명할 때만 조정한다.

| 상황 | 조정 | 같이 볼 것 |
| --- | --- | --- |
| 노드에 파드를 모아 컴퓨트 비용을 줄이고 싶다 | `MostAllocated` | consolidation 결합 검증, blast radius |
| 부하 증가에 더 빨리 스케일링하고 싶다 | syncPeriod 10s | PCP 과금, HPA 객체 수, 조용한 성능 저하 |
| Job/CronJob이 종료 파드를 대량으로 남긴다 | GC threshold 하향 | PCP 과금, 즉시 강제 삭제, logs 윈도우 축소 |
| 이벤트가 etcd를 잠식한다 | eventTtl 단축 | 점진적 감소, 외부 이벤트 수집 여부 |
| 기본 범위 밖 고정 NodePort가 필요하다 | serviceNodePortRange | SG/NACL, 포트 충돌, 서비스 재생성 시 재할당 불가 |

조정 전 `describe-cluster-versions`, 변경 후 `describe-cluster`로 확인하는 절차를 런북에 넣어 두면 된다.

**참고**

- [Amazon EKS 고급 컨트롤 플레인 구성하기 (AWS 기술 블로그)](https://aws.amazon.com/ko/blogs/tech/eks-advanced-control-plane/)
- [Advanced Kubernetes control plane configuration](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-configuration.html)
- [Configure advanced Kubernetes control plane parameters](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-configuration-getting-started.html)
- [Amazon EKS Provisioned Control Plane](https://docs.aws.amazon.com/eks/latest/userguide/eks-provisioned-control-plane.html)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
