---
layout: post
title: K8s GatewayAPI, Envoy Gateway
color: brown
tags: [EKS, Kubernetes, GatewayAPI, Envoy]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->


#### GatewayAPI 역할 분리와 트래픽 흐름
![my images]({{"/assets/img/thumbnails/envoy-gateway/gatewayapi.png" | absolute_url}})
![my images]({{"/assets/img/thumbnails/envoy-gateway/flow.png" | absolute_url}})

#### Envoy Gateway Architecture
![my images]({{"/assets/img/thumbnails/envoy-gateway/envoy-gateway-architecture.png" | absolute_url}})  

<table>
  <thead>
    <tr>
      <th>Field</th>
      <th>Type</th>
      <th>Required</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>envoyProxy</code></td>
      <td><code>EnvoyProxySpec</code></td>
      <td>false</td>
      <td>
        <p>EnvoyProxy defines the default EnvoyProxy configuration that applies to all managed Envoy Proxy fleet. This is an optional field and when provided, the settings from this EnvoyProxySpec serve as the base defaults for all Envoy Proxy instances.</p>
        <p>The hierarchy for EnvoyProxy configuration is (highest to lowest priority):</p>
        <ol>
          <li>Gateway-level EnvoyProxy (referenced via <code>Gateway.spec.infrastructure.parametersRef</code>)</li>
          <li>GatewayClass-level EnvoyProxy (referenced via <code>GatewayClass.spec.parametersRef</code>)</li>
          <li>This EnvoyProxy default spec</li>
        </ol>
        <p>Currently, the most specific EnvoyProxy configuration wins completely (replace semantics). A future release will introduce merge semantics to allow combining configurations across multiple levels.</p>
      </td>
    </tr>
  </tbody>
</table>

gatewayClass가 envoyProxy의 설정을 참조 하기 때문에
gatewayClass를 상속 받아 gateway에서 배포하는 envoyProxy Deployments는 당연하게도 gatewayClass, envoyProxy의 설정을 모두 상속 받게 된다.
즉, 1개의 gatewayClass를 상속받아 배포 되는 gateway는 모두 동일한 설정을 갖게 된다는 의미이고
다른 설정이 필요하다면 gatewayClass, envoyProxy를 모두 분리 하는 방법을 쓰거나
Gateway의 infrastructure.parametersRef로 Gateway마다 다른 EnvoyProxy를 줄 수 있기 때문에 GatewayClass를 분리 하지 않고도 배포 할 수도 있다.
ELB(NLB) 분리의 기준은 Gateway Custom Resources 이다.
Gateway의 교차 네임스페이스를 지원 하려면 referenceGrant Custom Resources를 배포 해야 한다.


#### 설계 고려 사항

##### 1. Gateway CR을 만들 때 고려사항

- 1개의 `Gateway`(NLB)를 공유해서 여러 서비스(Routes)가 사용할 경우
- 서비스마다 `Gateway`(NLB)를 분리할 경우

##### 2. Envoy Gateway Helm, Custom Resources 배포 방식

Helm Chart는 플랫폼 IaC 레포에서 배포하고,
클러스터마다 배포해야 하는 Custom Resources(CR)가 다를 수 있기 때문에
Envoy Gateway CR은 ArgoCD App-of-Apps 패턴으로 별도 레포에서 배포하는 편이 나을 것으로 보인다.

##### 3. RateLimit

RateLimit 기능을 사용하려면 Redis가 필요하다.
`BackendTrafficPolicy` CR을 통해 RateLimit을 설정할 수 있는데, 이 CR은 서비스 개발팀(서비스 네임스페이스)에서 배포해야 한다.

- CRD 스키마가 `LocalPolicyTargetReference`를 사용하기 때문에 플랫폼(Gateway 네임스페이스)에서 배포하는 것은 불가능하다.
- `gateway.envoyproxy.io`의 `BackendTrafficPolicy` 스키마에서 `targetRefs` 항목은 `LocalPolicyTargetReferenceWithSectionName` 타입이며, `group` / `kind` / `name` / `sectionName`만 있고 namespace 필드 자체가 없다.

차후 이 기능이 필요할 경우 Redis는 플랫폼 제공자가 ElastiCache 또는 Pod로 제공하고, `BackendTrafficPolicy` CR은 개발팀 레포에서 배포하는 방향으로 설계한다.


#### Ref.
_Gateway API Introduction : [Gateway API](https://gateway-api.sigs.k8s.io/docs/introduction/)
_Kubernetes Gateway : [Kubernetes Gateway](https://kubernetes.io/ko/docs/concepts/services-networking/gateway/)
_Envoy Gateway Extension API : [Envoy Gateway API Reference](https://gateway.envoyproxy.io/docs/api/extension_types/)
