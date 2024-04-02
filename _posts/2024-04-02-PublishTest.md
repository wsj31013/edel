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
EKS 1.29 버전 이상이 필요하다. 아래 호환성 매트릭스에서도 확인 할 수 있는데
```

#### Compatibility Matrix
![my images]({{"/assets/img/thumbnails/karpenter-compatibility/compatibility-matrix.png" | absolute_url}})


### Karpenter CRD Helm Migration


### 1일 1포스팅 하자