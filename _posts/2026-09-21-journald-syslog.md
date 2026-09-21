---
layout: post
title: EKS에서 systemd-journald와 syslog
color: brown
tags: [EKS, AL2023, journald, syslog, rsyslog, Fluent Bit, CloudWatch]
author: cobain
excerpt_separator: <!--more-->
---
<!--more-->

2026년 8월 28일 AWS가 [CloudWatch 에이전트 journald 로그 지원](https://aws.amazon.com/ko/about-aws/whats-new/2026/08/amazon-cloudwatch-agent-journald/)을 발표했다. Linux 인스턴스의 systemd journal을 에이전트가 직접 읽고, 디스크에 텍스트 파일로 먼저 쓰지 않은 채 CloudWatch Logs로 보낸다.

- Amazon Linux 2023을 포함한 최신 배포판은 journal이 기본이고, `/var/log/messages` 같은 텍스트 파일을 더 이상 만들지 않는다
- 이전에는 journal을 파일로 내보낸 뒤에야 CloudWatch 에이전트가 읽을 수 있었다
- 이제는 journal 항목을 native로 읽어서 systemd 유닛, 우선순위, 프로세스 정보 같은 구조화 메타데이터를 유지한다
- `units`, journal priority, 필드 매치, 정규식 필터로 양을 줄일 수 있다
- 상용 리전·GovCloud(US)에서 사용 가능하고, 수집 로그에는 [CloudWatch Logs 요금](https://aws.amazon.com/cloudwatch/pricing/)이 적용된다
- 에이전트를 최신으로 올린 뒤 설정 파일에 `journald` 섹션을 추가하면 된다. 필드는 [에이전트 구성 파일](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)에 있다

---

### EKS에서 CloudWatch 에이전트로 journal 보내기

EKS에서는[Amazon CloudWatch Observability 애드온](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Observability-EKS-addon.html) Addons 배포를 통해 구성할 수 있는데, 애드온이 CloudWatch 에이전트와 Fluent Bit를 DaemonSet으로 배포 하며 journald 수집은 **에이전트 설정**이고, 파드 stdout 수집은 Fluent Bit(`containerLogs`)다. 둘을 섞어 쓰지 않는다.

권한은 `CloudWatchAgentServerPolicy`가 필요하다. 노드 인스턴스 역할에 붙이거나, 애드온 3.1.0 이후라면 EKS Pod Identity로 `cloudwatch-agent` 서비스 계정에만 주는 편이 낫다. Kubernetes 1.23 이상.

에이전트는 `v1.300070.0`부터 journald를 지원한다. 애드온을 최신으로 맞춘 뒤 `agent.config`에 `journald`를 넣는다. 커스텀 에이전트 설정을 넘기면 기본값을 덮어쓰므로, 쓰던 Container Insights 항목은 같이 남겨야 한다.

EKS 워커 기준으로는 저널 전체가 아니라 `kubelet`·`containerd`만, `warning` 이상으로 좁혀 시작하는 편이 맞다. `units`를 생략하면 모든 유닛을 모으고, `priority`를 생략하면 기본이 `info` 이상이다. `debug`까지 필요하면 명시해야 한다.

```bash
aws eks create-addon \
  --cluster-name "$CLUSTER" \
  --addon-name amazon-cloudwatch-observability \
  --pod-identity-associations "serviceAccount=cloudwatch-agent,roleArn=$ROLE_ARN"
```

이미 애드온이 있으면 `update-addon`으로 설정을 넣는다.

```bash
aws eks update-addon \
  --cluster-name "$CLUSTER" \
  --addon-name amazon-cloudwatch-observability \
  --resolve-conflicts OVERWRITE \
  --configuration-values '{
    "agent": {
      "config": {
        "logs": {
          "metrics_collected": {
            "application_signals": {},
            "kubernetes": {
              "enhanced_container_insights": true
            }
          },
          "logs_collected": {
            "journald": {
              "collect_list": [
                {
                  "log_group_name": "/aws/eks/my-cluster/journal",
                  "log_stream_name": "{instance_id}",
                  "units": ["kubelet", "containerd"],
                  "priority": "warning",
                  "retention_in_days": 14
                }
              ]
            }
          }
        },
        "traces": {
          "traces_collected": {
            "application_signals": {}
          }
        }
      }
    }
  }'
```

필터 필드는 공식 문서 기준이 이렇다.

| 필드 | 역할 |
| --- | --- |
| `units` | systemd 유닛. 와일드카드 가능 (`sshd*`) |
| `priority` | 이 수준 **이상**(더 심각한 쪽)만 수집 |
| `matches` | journal 필드. 객체 안은 AND, 객체 사이는 OR |
| `filters` | 메시지 본문 RE2. `include` / `exclude` |

`units` + `priority` + `matches`를 한 항목에 같이 쓰면 **모두 만족**해야 나간다. 재시작 시에는 마지막 수집 지점부터 이어 읽고, 과거 저널은 다시 보내지 않는다. root가 아니면 `systemd-journal` 그룹이 필요하다.

Helm이면 같은 JSON을 `agent.config`에 넣는다.

```bash
helm upgrade --install amazon-cloudwatch-observability \
  aws-observability/amazon-cloudwatch-observability \
  --namespace amazon-cloudwatch --create-namespace \
  --set clusterName="$CLUSTER" \
  --set region="$REGION"
```

---

EKS 파드 로그는 예전처럼 `/var/log/pods`에 있는데, kubelet·containerd 쪽 시스템 로그는 텍스트 파일이 아니라 journal에만 존재하며 노드에 접속하여 확인하면 `/var/log/messages` 파일은 이제 존재하지 않는다.

로그가 없어진 건 아니다. `rsyslog`에서 `systemd-journald`로 바뀐 것이다. 

---

### syslogd

syslogd는 전통 BSD syslog 데몬(`syslogd`), 그리고 그 프로토콜을 구현한 `rsyslog`·`syslog-ng`를 통틀어 부를 때가 많다. 메시지 포맷도 RFC 3164(BSD syslog)와 RFC 5424가 같이 쓰인다.

Amazon Linux2와 Amazon Linux2023에서 실제로 갈린 것은 아래와 같다.

| | AL2 | AL2023 |
| --- | --- | --- |
| journald | 있음. systemd라서 이미 동작 | 있음. **기본이자 유일한 시스템 로거** |
| rsyslog | **기본 설치** | 기본 미설치, 선택 패키지 |
| `/var/log/messages` | 있음 | 없음 |

AWS 공식 문장:

> AL2023 doesn't install `rsyslog` by default, so the text based log files such as `/var/log/messages` that were available in AL2 aren't available by default. The default configuration for AL2023 is `systemd-journal`.

출처: [`systemd` journal replaces `rsyslog`](https://docs.aws.amazon.com/linux/al2023/ug/journald.html)

AL2도 journald는 돌고 있었다. re:Post 표현이 정확하다. AL2의 rsyslog는 **하위 호환용**이다. ([AL2023에서 로그 파일 찾기](https://repost.aws/knowledge-center/ec2-linux-al2023-find-log-files)) 즉 AL2는 로그를 사실상 두 번 남겼다.

```
서비스 stdout/stderr, /dev/log, 커널
        ↓
   systemd-journald   (/var/log/journal 또는 /run/log/journal)
        ↓  호환용으로 한 번 더
   rsyslog            (/var/log/messages, secure, maillog)
```

AL2023은 뒤의 rsyslog를 기본 이미지에서 뺀 것이다. 

그에 따라 조회 명령도 변경 되었다. ([journald.html](https://docs.aws.amazon.com/linux/al2023/ug/journald.html))

| AL2 | AL2023 |
| --- | --- |
| `cat /var/log/messages` | `journalctl` |
| `tail -f /var/log/messages` | `journalctl -f` |
| `grep foo /var/log/messages` | `journalctl -u kubelet -p err` 처럼 필드 조회 |

텍스트 파일이 다시 필요하면 공식 안내대로 `dnf install rsyslog && systemctl enable rsyslog --now`를 하면 된다. EKS에서는 이 한 줄이 노드 수명과 안 맞아서, 뒤에 따로 본다.

---

### systemd 이후 로그는 journald가 먼저 받는다

syslog 데몬이 있어야 시스템 로그가 생긴다는 전제는 systemd 이후로는 맞지 않는다. [systemd syslog 문서](https://systemd.io/SYSLOG/)의 요지는 이렇다.

- 커널 로그
- `/dev/log` (예전 syslog 소켓)
- 모든 서비스의 stdout/stderr
- journal native 프로토콜

이걸 journald 한곳에서 받는다. `/dev/log`를 듣는 쪽도 이제 journald다. 전통 syslog 데몬이 여기에 직접 bind하면 안 되고, `/run/systemd/journal/syslog`를 `syslog.socket`으로 받아야 한다. 직접 `/dev/log`를 열면 서비스 stdout/stderr를 놓친다.

```
앱 syslog(3) / sd_journal_print(3)
서비스 stdout·stderr
커널
        ↓
 systemd-journald
   ├─ 저장: /var/log/journal  (persistent) 또는 /run/log/journal (volatile)
   ├─ 선택: /run/systemd/journal/syslog  → 로컬 rsyslog
   └─ 선택: console / kmsg / wall
```

AL2023은 애플리케이션이 `/var/log`에 파일을 직접 쓰기보다 `syslog(3)` 또는 `sd_journal_print(3)`을 쓰라고 한다. ([`/var/log` (Persistent logs)](https://docs.aws.amazon.com/linux/al2023/ug/filesystem-slash-var.html)) kubelet, containerd 같은 유닛은 파일을 안 만들어도 journal에 남는다.

journald가 하지 않는 일도 분명하다. **원격 UDP/TCP 514 같은 전통 syslog 프로토콜은 말하지 않는다.** `ForwardToSyslog=yes`도 원격 서버가 아니라 로컬 소켓이다. ([journald.conf](https://www.freedesktop.org/software/systemd/man/latest/journald.conf.html))

---

### 왜 AL2023은 journal만 남겼을까

성능 벤치 때문에 rsyslog를 뺐다고 AWS가 쓴 문서는 없다. 비교 문서에 변경점이 나란히 적혀 있다. ([Comparing AL2 and AL2023](https://docs.aws.amazon.com/linux/al2023/ug/compare-with-al2.html))

- `systemd` journal replaces `rsyslog`
- Minimized package dependencies
- 부트 시간 단축

이미 journald가 로그를 모으고 있으니, 호환용 데몬과 텍스트 파일을 기본 이미지에서 빼자는 쪽에 가깝다. AL2023 릴리스 노트도 같은 이야기다. `rsyslog` 기본 미설치, syslog는 `systemd-journald` + `journalctl`. ([2022.0.20221101](https://docs.aws.amazon.com/linux/al2023/release-notes/relnotes-2022.0.20221101.html))

성능과 완전히 무관한 것은 아닌것 같다.

이중 기록(journal + `/var/log/messages`)이 사라지면 디스크 I/O와 데몬 하나가 줄어든다. 바이너리 저널은 필드 인덱스가 있어서 `journalctl -u kubelet -p err`가 `grep /var/log/messages`보다 검색에 유리하다. 텍스트 파일용 logrotate도 필요 없다.

반대로 journald가 rsyslog보다 항상 빠른 것도 아니다. 고트래픽에서 journald는 서비스별로 rate limit을 걸고, 넘치면 메시지를 버린다. 기본값은 30초에 10000건이고, 버리면 `Suppressed N messages from ...`를 남긴다. ([journald.conf](https://www.freedesktop.org/software/systemd/man/latest/journald.conf.html)) rsyslog가 journal DB를 다시 읽는 `imjournal`은 공식 문서도 “relatively performance-intense operation”이라고 한다. 구조화 필드가 필요 없으면 `imuxsock`을 쓰라고 한다. ([imjournal](https://docs.rsyslog.com/doc/configuration/modules/imjournal.html))

---

### 포맷과 메타데이터가 다른 이유

syslog 한 줄은 대략 시각, 호스트, 프로그램, 메시지다. journal은 키/값 레코드다. ([Journal File Format](https://systemd.io/JOURNAL_FILE_FORMAT/), [systemd.journal-fields](https://www.freedesktop.org/software/systemd/man/latest/systemd.journal-fields.html))

- `MESSAGE`, `PRIORITY`, `SYSLOG_IDENTIFIER` — 클라이언트가 넣을 수 있는 필드
- `_SYSTEMD_UNIT`, `_PID`, `_UID`, `_HOSTNAME` — 밑줄로 시작하는 **trusted field**. journald가 붙이고, 클라이언트가 위조해도 무시된다

텍스트 `/var/log/messages`로 한 번 풀면 이 필드 대부분이 사라진다. `ForwardToSyslog`로 로컬 소켓에 넘길 때도 구조화 메타데이터는 거의 남지 않는다. 그래서 rsyslog 문서가 “구조화 데이터가 필요할 때만 `imjournal`”이라고 하는 것이다.

저널 파일은 append 전용 바이너리고 인덱스가 있다. 사람이 `cat`으로 읽는 포맷이 아니다. 네트워크로 넘길 때도 systemd는 journal export/JSON을 쓰고, 전통 syslog와는 다른 프로토콜이다. `systemd-journal-upload` → `systemd-journal-remote`는 HTTP journal이지 syslog가 아니다. 이미 있는 중앙 syslog 서버와는 보통 호환되지 않는다.

순환도 다르다. journald는 `SystemMaxUse=`, `SystemKeepFree=` 같은 크기 기준으로 오래된 저널을 지운다. rsyslog 텍스트 파일은 logrotate에 맡기는 경우가 많다.

---

### EKS에서 로그가 갈라지는 지점

| 층 | 누가 쓰나 | 어디 있나 | 수집 |
| --- | --- | --- | --- |
| 컨트롤 플레인 | kube-apiserver 등, AWS 관리 | CloudWatch (클러스터 로그를 켰을 때) | EKS가 보냄. 워커와 무관 |
| 애플리케이션 | 파드 stdout/stderr | 노드 `/var/log/pods` → `/var/log/containers` | Fluent Bit/OTel tail. 대부분의 기존 파이프라인 |
| 노드 시스템 | kubelet, containerd, sshd | **journal** (`kubelet.service`, `containerd.service`) | journal을 읽는 수집기가 따로 필요 |
| 노드 위 시스템 파드 | kube-proxy, aws-node(VPC CNI) | 파드 로그 경로 | 애플리케이션 층과 같음 |

컨트롤 플레인은과 파드 로그는 글 범위가 아니며 kubelet이 CRI 로그를 파일로 남긴다.

EKS-optimized AL2023 AMI는 `kubelet`과 `containerd`를 심고, AL2023에서 Docker runtime은 지원하지 않는다. ([EKS-optimized AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html), [AL2 → AL2023](https://docs.aws.amazon.com/eks/latest/userguide/al2023.html)) 둘 다 systemd 유닛이라 로그는 journal이 로그를 저장한다.

Fargate는 노드에 DaemonSet을 올리지 못하므로 journal을 읽을 대상 자체가 없다. 이 글은 EC2 워커(MNG, self-managed, Karpenter) 기준이다.

AWS Container Insights 문서의 host 로그 경로는 오래전부터 `/var/log/dmesg`, `/var/log/secure`, `/var/log/messages`를 가리킨다. ([EKS logging](https://docs.aws.amazon.com/prescriptive-guidance/latest/implementing-logging-monitoring-cloudwatch/kubernetes-eks-logging.html)) AL2023에는 그 파일이 기본으로 없다. 현재 Container Insights Fluent Bit 매니페스트의 `host-log.conf`는 이미 journal의 `systemd` 입력으로 바꿔 두었다. kernel transport, PRIORITY, `SYSLOG_FACILITY=10`(authpriv)로 dmesg/messages/secure에 해당하는 레코드를 고른다. ([fluent-bit.yaml](https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml)) 문서의 파일 경로와 실제 수집 경로가 어긋나 있는 상태라, AL2 시절 설정을 그대로 가져오면 host 로그가 비는 사고가 난다.

dataplane 쪽도 비슷하다. 예전 설명은 `docker.service`를 적는다. 지금 EKS AL2023 워커는 `containerd.service`다.

---

### 원격 syslog로 보내려면 브리지가 필요하다

중앙에 syslog 서버가 있는 구성은 흔하다. journald 단독으로는 거기 보내지 못한다.

```
EKS AL2023 워커
  kubelet / containerd / sshd
        ↓
   systemd-journald
        ↓
   브리지 (노드 rsyslog 또는 Fluent Bit)
        ↓  syslog UDP/TCP/TLS
   중앙 syslog 서버
```

rsyslog 원격 전송은 **로컬 텍스트 파일을 경유하지 않는다.** rsyslog는 파일 테일러가 아니다. 메시지를 메모리로 받고, 출력 액션은 서로 독립이다. ([Forwarding Logs](https://docs.rsyslog.com/doc/getting_started/forwarding_logs.html))

```
rsyslog (메모리 큐)
  ├─ omfile  → /var/log/messages   (선택)
  └─ omfwd   → 원격 syslog          (선택)
```

최소 예시는 `omfwd`만 있다. 로컬 파일 쓰기는 필수 단계가 아니다. 원격이 죽으면 TCP 전송이 로컬 로깅을 막을 수 있어서, rsyslog는 메모리/디스크 **자체 큐**를 쓴다. 이 큐 파일은 `/var/log/messages`가 아니다. TCP면 `queue.type=linkedList`를 권장한다.

앞에서 본 CloudWatch 에이전트 journald는 목적지가 CloudWatch Logs다. 기존 중앙 syslog로 보내려면 아래 둘 중 하나다.

---

### 옵션 1. 노드에 rsyslog

```
journald → (ForwardToSyslog 또는 imjournal) → rsyslog omfwd → 중앙 syslog
```

journal을 rsyslog로 넣는 방법은 두 가지고, **하나만** 고른다. 둘을 같이 켜면 로그가 두 번 간다.

| 방식 | 경로 | 장점 | 단점 |
| --- | --- | --- | --- |
| `ForwardToSyslog=yes` + `imuxsock` | journald가 `/run/systemd/journal/syslog`로 푸시 | 가볍고 rsyslog가 권하는 쪽에 가까움 | 구조화 필드 손실. journal이 바쁠 때 drop 가능 |
| `imjournal` | rsyslog가 journal DB를 직접 읽음 | `_SYSTEMD_UNIT`, `_PID` 유지 | 더 무거움. 기본 rate limit 10분에 2만 건 |

기존 syslog 서버가 텍스트 한 줄만 받으면 전자로 충분하다. `imjournal`을 쓰면 cursor state file이 필요하다. 없으면 재시작 시 과거를 다시 읽거나, `IgnorePreviousMessages`로 새 로그만 간다.

EKS에서 설치하는 방법이 문제다. Managed Node Group과 Karpenter는 노드를 자주 교체한다. SSH나 SSM으로 한 대 설치하는 방식은 Autorepair·ASG 교체 후 사라진다.

남는 방법은 User Data 또는 커스텀 AMI다.

AL2023 MNG user data는 MIME multipart여야 한다. `application/node.eks.aws`(nodeadm)와 `text/x-shellscript`를 같이 쓴다. `nodeadm init`은 직접 실행하면 안 된다. AMI가 `nodeadm-config` → user data → `nodeadm-run` 순으로 이미 돌린다. 한 번 더 실행하면 ENI가 꼬일 수 있다. ([al2023.html](https://docs.aws.amazon.com/eks/latest/userguide/al2023.html), [Launch template support](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html))

부트 시 하는 일은 단순하다.

1. `dnf install -y rsyslog`
2. `/etc/rsyslog.d/forward.conf`에 `omfwd`만 둔다. 로컬 `omfile`은 생략 가능
3. journal 연동은 위 두 방식 중 하나
4. `systemctl enable --now rsyslog`
5. 워커 보안 그룹에서 중앙 syslog로 egress 허용

주의할 점은 이 정도다.

- 노드 부팅마다 `dnf`가 돈다. 프라이빗 클러스터면 yum 미러/프록시가 필요하다
- x86 / arm 노드그룹에 동일하게 넣어야 한다
- user data를 바꾸면 Launch Template 버전 갱신
- 패키지 설치가 실패해도 kubelet은 뜨게 할지, 부팅을 실패로 볼지 정책을 정해야 한다

커스텀 AMI에 rsyslog를 설치패 빌드 해두면 부팅이 빠르고 `dnf` 의존이 없다. 대신 AMI 파이프라인, 보안 패치, 아키텍처별 빌드가 생긴다. 클러스터가 많으면 유지비가 커진다.

노드 OS에 syslog 데몬이 있어야 한다는 기존 서버 표준을 그대로 가져가려면 이 옵션이다.
---

### 옵션 2. Fluent Bit DaemonSet

노드 OS는 그대로 두고, 파드가 host journal을 읽어 원격 syslog로 보낸다.

```
host /var/log/journal
        ↓ hostPath
 Fluent Bit DaemonSet
        ↓  [OUTPUT] syslog
 중앙 syslog 서버
```

입력은 [systemd](https://docs.fluentbit.io/manual/data-pipeline/inputs/systemd), 출력은 [syslog](https://docs.fluentbit.io/manual/data-pipeline/outputs/syslog)다. RFC 3164/5424, UDP/TCP/TLS를 지원한다. 같은 일을 Fluentd로도 할 수 있다. journal은 `fluent-plugin-systemd`, 원격 syslog는 보통 `fluent-plugin-remote_syslog`다. 둘 다 기본 이미지에 없을 수 있고, 리소스도 Fluent Bit보다 크다. 노드 journal을 한곳으로 보내는 용도면 Fluent Bit가 맞다.

파드가 journal을 읽으려면 이 정도가 필요하다.

- `/var/log/journal` 마운트. AL2023은 persistent journal을 여기에 둔다
- `/etc/machine-id` (저널 식별)
- journal 파일은 `systemd-journal` 그룹 소유. `supplementalGroups`에 해당 GID를 넣거나, 권한을 넓힌다. GID가 AMI마다 다를 수 있다
- journal cursor를 `hostPath`에 저장. 안 하면 파드 재시작 시 중복 또는 공백이 난다
- `Read_From_Tail On` — 배포 직후 과거 저널을 쏟지 않음
- 필터: `_SYSTEMD_UNIT=kubelet.service`, `containerd.service`부터. 전체 journal은 양이 크다
- `Path`가 없는 디렉터리면 프로세스가 죽는다. 없는 `/run/log/journal`을 억지로 넣지 않는 편이 안전하다

대략적인 설정은 아래와 같다.

```
[INPUT]
    Name              systemd
    Tag               host.*
    Path              /var/log/journal
    DB                /var/fluent-bit/state/systemd.db
    Read_From_Tail    On
    Systemd_Filter    _SYSTEMD_UNIT=kubelet.service
    Systemd_Filter    _SYSTEMD_UNIT=containerd.service

[OUTPUT]
    Name                  syslog
    Match                 host.*
    Host                  logs.example.com
    Port                  514
    Mode                  tcp
    Syslog_Format         rfc5424
    Syslog_Hostname_Key   _HOSTNAME
    Syslog_Appname_Key    SYSLOG_IDENTIFIER
    Syslog_Procid_Key     _PID
    Syslog_Message_Key    MESSAGE
```

`Syslog_Message_Key`는 필수다. 없으면 빈 메시지가 나간다. 구형 서버면 `rfc3164`와 UDP로 맞춘다. syslog로 가는 순간 구조화 필드는 대부분 한 줄로 평탄화된다.

이미지 버전도 함정이다. systemd 246 이후 journal 압축(zstd)을 구버전 Fluent Bit가 못 읽는다. 업스트림은 1.8.11에서 고쳤고, `aws-for-fluent-bit` 2.32.x대는 AL2023에서 systemd 입력이 실패하는 사례가 있다. ([aws-for-fluent-bit#831](https://github.com/aws/aws-for-fluent-bit/issues/831)) 3.0.0부터 베이스가 AL2023이고 Fluent Bit 4.x다. ([AWS for Fluent Bit 3.0.0](https://aws.amazon.com/about-aws/whats-new/2025/10/aws-fluent-bit-3-0-0-based-4-1-1/))

설정 변경은 ConfigMap + 파드 롤링이면 된다. 노드 롤링이 없다. Autoscaling으로 노드가 늘어도 DaemonSet이 따라간다. MNG·Karpenter처럼 노드가 자주 바뀌는 구성에서는 이쪽이 운영 비용이 낮다.

대가는 호스트 journal 접근이다. 보안 컨텍스트가 넓어지고, 수집기가 노드 OS 일부가 아니라 클러스터 워크로드가 된다.

---

### 두 옵션을 같이 쓰면 안 된다

| | 노드 rsyslog | Fluent Bit DS |
| --- | --- | --- |
| 설치 위치 | OS 데몬 | 클러스터 파드 |
| 노드 교체 | User Data/AMI에 의존 | DS가 자동 배포 |
| 설정 변경 | 노드 롤링 | 파드 롤링 |
| 기존 syslog와의 유사성 | 가장 비슷 | 프로토콜만 맞춤 |
| 리소스 | 노드당 데몬 1개 | Fluent Bit는 가벼운 편 |
| 필터·변환 | rsyslog 룰 | 플러그인이 더 유연 |
| 확대 | 노드 템플릿 복제 + 롤링 | 차트 값 복제 |

둘 다 기술적으로 된다. 같은 저널을 양쪽에 넣으면 중복만 늘어난다. CloudWatch 에이전트 journald 수집을 같이 켜도 마찬가지다.

공통 선행은 수집기보다 먼저다.

- 워커 → 중앙 syslog 네트워크와 보안 그룹
- 서버가 UDP 514만 받는지, TCP/TLS인지
- 수집 범위. kubelet/containerd부터 보고, 전체 journal은 나중에
- 파드 로그 파이프라인과 역할을 나눈다. 파드 stdout은 `/var/log/pods`, 노드 유닛은 journal

운영 관점에서는 EKS MNG + AL2023 + 노드 교체가 잦은 구성이라 Fluent Bit DS가 더 맞다. 노드 OS에 syslog 데몬이 있어야 한다는 기존 서버 표준을 그대로 가져가려면 rsyslog다.

