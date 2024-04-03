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

0.34 이상부터는 기존 CRD(Provisioner, NodeTemplate) 이 지원되지 않기 때문에
정확히는 0.32가 기존 CRD와 신규 CRD(NodeClass, NodePool, NodeClaim)를 지원하는 마지막 버전이다.
그만큼 변경되는 부분이 많기 때문에 업그레이드를 하려면 신중하게 테스트 해야 한다.

```

```sh
karpenter crd를 helm 차트 관리 방식으로 변경 하려면 crd helm chart 레이블과 어노테이션을 한번 추가 해줘야 한다.
이 한번의 작업은 crd를 helm으로 migration 하기전 단 한번만 하면 되는 작업이고 karpenter 공식문서에도 관련 내용이 언급되어 있다.

# kubectl label crd ec2nodeclasses.karpenter.k8s.aws nodepools.karpenter.sh nodeclaims.karpenter.sh app.kubernetes.io/managed-by=Helm --overwrite
# kubectl annotate crd ec2nodeclasses.karpenter.k8s.aws nodepools.karpenter.sh nodeclaims.karpenter.sh meta.helm.sh/release-name=karpenter-crd --overwrite
# kubectl annotate crd ec2nodeclasses.karpenter.k8s.aws nodepools.karpenter.sh nodeclaims.karpenter.sh meta.helm.sh/release-namespace=karpenter --overwrite
```

#### terraform 으로 karpenter-crd를 배포 하는 예제 코드
```terraform
resource "helm_release" "karpenter_crd" {
  namespace                 = "karpenter"
  create_namespace     = true

  name         = "karpenter-crd"
  repository  = "oci://public.ecr.aws/karpenter"
  repository_username = data.aws_ecrpublic_authorization_token.token.user_name
  repository_password = data.aws_ecrpublic_authorization_token.token.password
  chart         = "karpenter-crd"
  version      = var.stable_karpenter_helm
}

```

#### Ref.
_Karpenter Official Document : [Karpenter upgrade guide](https://karpenter.sh/preview/upgrading/upgrade-guide/)_

