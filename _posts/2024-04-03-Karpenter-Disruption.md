---
layout: post
title: Karpenter Disruption 정리
color: brown
tags: [EKS, Karpenter, Disruption, Kubernetes]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

### Karpenter Disruption
```text
Karpenter를 프로덕션 레벨에서 사용하면서 가장 불안한 부분은 노드가 중단 되는 사태인데
공식 문서 보면서 공부 한 내용을 정리 한다.
틀린 부분이 있을 수 있기 때문에 이 내용을 백퍼센트 믿기 보단 참고해서 사용하길 바란다.
```


#### 1. Pods 수준에서 disruption을 방지
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        karpenter.sh/do-not-disrupt: "true"
```

#### 2. nodepool 수준에서 disruption을 방지
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      annotations:
        karpenter.sh/do-not-disrupt: "true"
```

#### 3. 특정 노드 수준에서 disruption을 방지 
```yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
    karpenter.sh/do-not-disrupt: "true"
```


### Karpenter NodePool Budgets
```text
NodePool의 spec.disruption.budgets. 
정의되지 않은 경우 Karpenter는 기본적으로 nodes: 10%. 
예산은 어떤 이유로든 적극적으로 삭제되는 노드를 고려하며 
Karpenter가 만료, 드리프트, 비어 있음 및 통합을 통해 
자발적으로 노드를 방해하는 것을 차단합니다.

번역기 돌린거라 좀 이상한데 뜻은 10%로 설정 되어 있을 경우(디폴트 10%)
병합, 삭제 대상의 노드 비율이 10%로 지정 한다는 뜻임
```

#### 다수의 Budgets이 설정된 NodePool 예제
```yaml
# NodePool에 여러 예산이 있는 경우 Karpenter는 각 예산의 최소값(가장 제한적)을 사용
# 예를 들어 예산이 3개인 다음 NodePool은 다음 요구 사항을 정의합니다.
# 첫 번째 예산에서는 해당 NodePool이 소유한 노드의 20%만 중단되도록 허용합니다. 
# 예를 들어 NodePool이 소유한 노드가 19개라면 4번의 중단이 허용됩니다 19 * .2 = 3.8.
# 두 번째 예산은 이전 예산의 상한선 역할을 하며, 노드가 25개가 넘는 경우 중단을 5번만 허용합니다.
# 마지막 예산은 하루 중 처음 10분 동안의 중단만 차단하며 중단은 0회 허용됩니다.
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h # 30 * 24h = 720h
    budgets:
    - nodes: "20%"
    - nodes: "5"
    - nodes: "0"
      schedule: "@daily"
      duration: 10m
```

### 공식 : allowed_disruptions = roundup(total * percentage) - total_deleting - total_notready
#### 문제1. 만약 전체 노드 10개, percent 10%, total_deleting 1개, total_notready 1개, 이 조건일때 allowed_disruptions를 구하면?
```text
- 전체 노드 수(total): 10개
- 백분율(percentage): 10%
- 현재 삭제 중인 노드 수(total_deleting): 1개
- NotReady 상태의 노드 수(total_notready): 1개

allowed_disruptions = roundup(10 * 0.10) - 1 - 1
여기서 10 * 0.10 = 1이므로, 반올림(roundup)을 적용할 필요가 없으며, 계산된 값이 이미 정수 입니다. 
따라서 allowed_disruptions = 1 - 1 - 1이 되어, allowed_disruptions의 값은 -1
하지만 실제 allowed_disruptions 의 값이 음수가 되는 것은 불가능 합니다. 
실제로는 Karpenter가 노드를 자발적으로 중단할 수 있는 여유가 없음을 의미합니다. 
즉, 현재 삭제 중인 노드와 NotReady 상태의 노드를 고려할 때, 추가적인 자발적 중단이 허용되지 않습니다.
```

#### 문제2. 만약 전체 노드 10개, percent 10%, total_deleting 0개, total_notready 0개, 위 조건일때 allowed_disruptions를 구하면?
```text
- 전체 노드 수(total): 10개
- 백분율(percentage): 10%
- 현재 삭제 중인 노드 수(total_deleting): 0개
- NotReady 상태의 노드 수(total_notready): 0개

allowed_disruptions = roundup(10 * 0.10) - 0 - 0
여기서 10 * 0.10 = 1입니다. 이 경우, 반올림(roundup)을 적용할 필요가 없으며, 
계산된 값이 이미 정수입니다. 따라서 allowed_disruptions = 1 - 0 - 0 이 되어, allowed_disruptions의 값은 1이 됩니다.

즉, 이 조건에서는 Karpenter가 NodePool에서 최대 1개의 노드를 자발적으로 중단할 수 있음을 의미합니다.
```


#### Ref.
_Karpenter Official Document : [Karpenter disruption guide](https://karpenter.sh/docs/concepts/disruption/)_