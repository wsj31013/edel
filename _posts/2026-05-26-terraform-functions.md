---
layout: post
title: Terraform 내장 함수 정리 — tonumber부터
color: brown
tags: [Terraform, Harbor, IaC, DevOps]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

Harbor 리텐션 폴리시를 Terraform으로 관리하다가, `dynamic "rule"` 블록을 조건부로 넣을 때 `tonumber`를 처음 쓰게 되어
자주 사용하는 함수를 정리한다.   
변수는 YAML/JSON에서 오기 때문에 **문자열(string) `"3"`** 과 **숫자(integer) `3`** 이 섞여 들어오고, `> 0` 같은 비교를 하려면 타입을 맞춰야 한다.  

```hcl
dynamic "rule" {
  for_each = (
    tonumber(lookup(each.value, "prefix_retention_pushed_keep", 0)) > 0 &&
    length(trimspace(lookup(each.value, "protect_retention_prefix", ""))) > 0
  ) ? [1] : []
  content {
    most_recently_pushed = each.value.prefix_retention_pushed_keep
    tag_matching         = lookup(each.value, "protect_retention_tag_matching", "${trimspace(each.value.protect_retention_prefix)}*")
    repo_matching        = contains(keys(each.value), "repo_excluding") ? null : "**"
    repo_excluding       = contains(keys(each.value), "repo_excluding") ? each.value.repo_excluding : null
    untagged_artifacts   = false
  }
}
```

**타입 변환(`tonumber`)**, **문자열 처리(`trimspace`, `lookup`)**, **조건(`contains`, 삼항 연산자)**, **컬렉션 반복(`for_each`, `keys`)** 은 자주 쓰이는 코드 이므로 알아두는게 좋다.
이 글에서는 실무에서 자주 쓰는 Terraform 내장 함수를 카테고리별로 정리 해본다. (Terraform 1.x 기준, HCL2)

---

### 1. 문자열 함수

문자열은 변수 기본값, 태그/경로 매칭, 리소스 이름 조합에서 가장 많이 다룬다.

#### `lower` / `upper` / `title`

대소문자 변환. 리소스 이름 규칙이나 비교 시 유용

```hcl
lower("My-Project")   # "my-project"
upper("dev")          # "DEV"
title("hello world")  # "Hello World"
```

#### `trim` / `trimspace` / `trimprefix` / `trimsuffix`

앞뒤 공백·접두/접미사 제거. 사용자 입력·YAML 값 정제에 필수다.

```hcl
trimspace("  nginx  ")           # "nginx"
trimprefix("prod-api", "prod-") # "api"
trimsuffix("app.tar.gz", ".gz")  # "app.tar"
```

예시에서 `protect_retention_prefix` 앞뒤 공백을 제거한 뒤 `*` 와 붙여 `tag_matching` 을 만든다.

#### `split` / `join`

문자열 ↔ 리스트 변환. CSV 형태 변수, ARN·경로 파싱에 자주 쓴다.

```hcl
split(",", "a,b,c")           # ["a", "b", "c"]
join("-", ["web", "01", "kr"]) # "web-01-kr"
```

#### `format` / `formatlist`

`printf` 스타일 포맷. `formatlist`는 리스트 각 원소에 동일 포맷을 적용한다.

```hcl
format("sg-%s", var.env)                    # "sg-prod"
formatlist("subnet-%s", ["a", "b"])         # ["subnet-a", "subnet-b"]
```

#### `replace` / `regex` / `regexall` / `regexreplace`

단순 치환과 정규식. 태그·이미지 경로 패턴을 다룰 때 `regex` 계열이 필요하다.

```hcl
replace("1.2.3", ".", "-")                  # "1-2-3"
regex("^[0-9]+$", var.retention_count)      # true/false
regexreplace("v1.2.3", "^v", "")            # "1.2.3"
```

#### `startswith` / `endswith` / `strcontains`

조건 분기 없이 불리언만 필요할 때 가독성이 좋다.

```hcl
startswith("registry.io/myapp", "registry.io")  # true
strcontains("release-1.0", "release")         # true
```

#### `length` (문자열·리스트·맵 공통)

문자열이 비었는지, 리스트가 비었는지 확인한다. 예시의 `length(trimspace(...)) > 0` 가 대표적이다.

---

### 2. 숫자 관련 함수

#### `max` / `min`

여러 후보 중 최댓값·최솟값. `count`·용량·버전 비교에 쓴다.

```hcl
max(3, 10, 7)   # 10
min(3, 10, 7)   # 3
```

#### `abs`

절댓값. 차이 계산이나 음수 방지 시.

```hcl
abs(-5)  # 5
```

#### `ceil` / `floor` / `parseint`

실수·문자열 숫자 처리. `parseint("08", 10)` 처럼 **8진수로 해석되지 않게** 10진수를 명시하는 패턴이 있다.

```hcl
ceil(2.1)              # 3
parseint("42", 10)     # 42
```

#### `tonumber`

문자열 `"10"` → 숫자 `10`. 변환 실패 시 **plan/apply 단계에서 에러**가 난다.

```hcl
tonumber("3") > 0        # true
tonumber(lookup(map, "k", "0")) > 0
```

`lookup(..., 0)` 의 기본값이 숫자 `0`이면 `tonumber` 없이도 비교가 되지만, 외부 YAML에서 `"0"` 문자열로 오는 경우가 많아 `tonumber`를 쓰는 편이 안전하다.

---

### 3. 조건·컬렉션 함수

#### `coalesce`

인자 중 **첫 번째 non-null, non-empty** 값을 반환한다.  
`null`과 빈 문자열 `""` 은 “비어 있음”으로 취급한다.

```hcl
coalesce(var.image_tag, "latest")           # tag가 null/""이면 "latest"
coalesce("", null, "fallback")              # "fallback"
```

`try()`와 비슷해 보이지만, `coalesce`는 **값이 비었는지**에 초점이 있고 `try()`는 **표현식 평가 실패**를 잡는다.

#### `contains`

리스트·맵·문자열에 요소/키/부분 문자열이 있는지 확인.

```hcl
contains(["a", "b"], "a")                    # true
contains(keys({ repo_excluding = "x" }), "repo_excluding")  # true
contains("hello", "ell")                     # true
```

예시에서는 `repo_excluding` 키 존재 여부로 `repo_matching` / `repo_excluding` 중 하나만 설정한다.

#### `count` (리스트 필터용)

**리스트**에서 조건을 만족하는 원소 개수. `length`와 헷갈리기 쉬우니 이름을 기억할 것.

```hcl
count(["a", "bb", "ccc"], length) > 1   # 길이 1 초과인 문자열 개수 → 2
```

`for` 표현식과 함께 쓰면 “조건을 만족하는 항목이 하나라도 있는가?”를 간단히 표현할 수 있다.

```hcl
count(var.subnets, can(cidrhost(., 0))) == length(var.subnets)
```

#### `lookup`

맵에서 키 조회, 없으면 기본값.

```hcl
lookup(each.value, "prefix_retention_pushed_keep", 0)
lookup(each.value, "protect_retention_tag_matching", "${trimspace(each.value.protect_retention_prefix)}*")
```

#### `try` / `can`

동적 타입·선택 속성에서 plan 실패를 막을 때.

```hcl
try(each.value.optional_field, "default")
can(regex("^v[0-9]+$", var.tag))   # 정규식 실패 시 false, 에러 아님
```

---

### 4. 타입 변환 함수

Terraform은 **정적 타입**이지만, 변수·외부 데이터·provider 응답은 런타임에 타입이 섞인다. 리소스 인자에 넣기 전에 명시적으로 변환하는 습관이 필요하다.

| 함수 | 입력 예 | 결과 | 비고 |
|------|---------|------|------|
| `tonumber` | `"42"`, `42` | number | 실패 시 에러 |
| `tobool` | `"true"`, `true` | bool | `"false"` 문자열도 처리 |
| `tolist` | tuple, set | list | 순서·중복 규칙 주의 |
| `toset` | list | set | 중복 제거, 순서 없음 |
| `tomap` | object, `{k=v}` | map | 키는 모두 string |

```hcl
# 숫자 비교
tonumber(lookup(each.value, "keep", "0")) > 0

# feature flag
tobool(var.enable_scan)

# for_each는 set 또는 map — list를 set으로
toset(var.repo_names)

# 모듈 출력이 tuple일 때 list로
tolist(module.vpc.private_subnet_ids)
```

**자주 하는 실수**

- `"0"` > `0` 은 문자열 비교가 되어 의도와 다르게 동작할 수 있다 → `tonumber` 사용.
- `for_each`에 list를 그대로 넣으면 중복 키로 에러 → `toset` 또는 `tomap`으로 키 보장.
- `tobool("false")` 는 `false`이지만, `tobool("")` 는 에러다.

---

### 5. 기타 — 파일 읽기

#### `file`

경로의 **원시 파일 내용**을 문자열로 읽는다. 모듈 기준 **상대 경로**이며, plan 시점에 파일이 있어야 한다.

```hcl
resource "kubernetes_config_map" "policy" {
  data = {
    "policy.rego" = file("${path.module}/policies/allow.rego")
  }
}
```

#### `templatefile`

파일 안에 `${변수}` 치환을 적용한다. JSON/YAML/스크립트 템플릿에 적합하다.

```hcl
locals {
  retention_json = templatefile("${path.module}/retention.tpl.json", {
    project     = var.project
    keep_count  = tonumber(var.keep_count)
    tag_pattern = "${trimspace(var.prefix)}*"
  })
}
```

`retention.tpl.json` 예:

```json
{
  "rules": [
    {
      "action": "retain",
      "template": "${tag_pattern}",
      "params": { "latestPushedK": ${keep_count} }
    }
  ]
}
```

`file` vs `templatefile`

| | `file` | `templatefile` |
|---|--------|----------------|
| 변수 치환 | 없음 | 2번째 인자 map으로 치환 |
| 용도 | 그대로 삽입 (정책, 키, 스크립트) | 환경별 설정 파일 생성 |

`path.module`, `path.root`, `path.cwd` 는 파일 경로 해석 시 함께 기억하면 편하다.

---

### 6. 함수 선택시 참고

| 하고 싶은 일 | 함수 |
|-------------|------|
| YAML에서 온 숫자 문자열 비교 | `tonumber` |
| 빈 문자열/ null 기본값 | `coalesce`, `lookup` |
| 맵에 키 있는지 | `contains(keys(...), "key")` |
| 리스트를 `for_each`에 | `toset` / `tomap` |
| 경로·태그 조합 | `join`, `format`, `trimspace` |
| CSV → 리스트 | `split` |
| 정책/키 파일 삽입 | `file`, `templatefile` |

---


Ref : [Terraform Functions](https://developer.hashicorp.com/terraform/language/functions)
