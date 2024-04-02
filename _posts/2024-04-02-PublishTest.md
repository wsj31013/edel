---
layout: post
title: Karpenter Upgrade 0.32 to 0.34
color: brown
tags: [EKS, Karpenter, NodeBudget, Kubernetes]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

### Karpenter Version Upgrade
```text
karpenter 0.34 버전에서부터 nodeBudget을 사용할 수 있게 되었는데 이걸 EKS에서 사용하려면
EKS 1.29 버전 이상이 필요하다. 아래 호환성 매트릭스에서도 확인 할 수 있다.

```

#### Compatibility Matrix
![my images]({{"/assets/img/thumbnails/karpenter-compatibility/compatibility-matrix.png" | absolute_url}})
_Karpenter Compatibility URL : [Karpenter](https://karpenter.sh/preview/upgrading/compatibility/)_

### Karpenter CRD Helm Migration
```text
아직까지 딱히 단점은 못찾겠는데 K8s에서 잘 알려진 PDB와 유사한 개념으로 Karpenter의 노드 레벨에서
Budget 설정을 하여 노드가 한꺼번에 중단 되는 사태를 방지 할 수 있게 된다.
기존 Karpenter는 Terraform, Helm으로 배포 하던 상태 였고 업그레이드를 하려고 보니
karpenter-crd Helm 차트도 존재 한다는것을 확인 했다.

어차피 Karpenter를 Helm 차트로 배포 하게 되면 처음에 CRD(nodeclass, nodepool, nodeclaim)를 생성 하는데
업그레이드 할때는 초기에 생성한 CRD의 라이프사이클을 자동으로 관리 해주지 않는다.
결국 사용자가 kubectl과 같은 클라이언트로 CRD를 업데이트 해야 하는 문제가 있다.

Karpenter 측에서도 Karpenter 컨트롤러가 배포 될때 같은 버전의 CRD를 배포 하도록 하기 위해 개선이 된거 같다.
정확히 언제부터 릴리즈 된건지 찾아 봤는데 아 귀찮다.
```

```terraform
```

### 1일 1포스팅 하자