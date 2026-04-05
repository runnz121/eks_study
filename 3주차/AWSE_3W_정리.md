# EKS Scaling 완전 정리 - 3주차

> 작성 기준: AEWS(Amazon EKS Workshop Study) 3주차 실습 내용 기반  
> 이 문서는 EKS 환경에서 파드와 노드를 어떻게 자동으로 늘리고 줄이는지, 그 핵심 메커니즘과 실무 적용 포인트를 정리한 것입니다.

---

## 목차

1. [실습 환경 개요](#1-실습-환경-개요)
2. [EKS 관리형 노드 그룹](#2-eks-관리형-노드-그룹)
3. [스케일링 전략 전체 그림](#3-스케일링-전략-전체-그림)
4. [HPA - Horizontal Pod Autoscaler](#4-hpa---horizontal-pod-autoscaler)
5. [VPA - Vertical Pod Autoscaler](#5-vpa---vertical-pod-autoscaler)
6. [KEDA - 이벤트 기반 오토스케일링](#6-keda---이벤트-기반-오토스케일링)
7. [CPA - Cluster Proportional Autoscaler](#7-cpa---cluster-proportional-autoscaler)
8. [CAS - Cluster Autoscaler](#8-cas---cluster-autoscaler)
9. [Karpenter - Just-in-time 노드 프로비저닝](#9-karpenter---just-in-time-노드-프로비저닝)
10. [Fargate - Serverless 컨테이너 실행](#10-fargate---serverless-컨테이너-실행)

---

## 1. 실습 환경 개요

이번 주차에서는 **Terraform**으로 EKS 클러스터를 구성하고, 그 위에서 다양한 스케일링 도구들을 직접 실습한다. 클러스터는 Kubernetes 1.35 버전 기반이고, 워커 노드는 private subnet에 배치되어 있으며 SSM을 통해 접근한다.

### 주요 구성 요소

| 구성 요소 | 역할 |
|-----------|------|
| `metrics-server` | 파드/노드의 CPU·메모리 사용량 수집 → HPA, VPA가 참조 |
| `external-dns` | k8s 서비스/인그레스 변경 시 Route53 레코드 자동 업데이트 |
| `aws-load-balancer-controller` | ALB/NLB 인그레스를 k8s 오브젝트로 관리 |
| `kube-ops-view` | 노드·파드 상태를 시각적으로 보여주는 웹 UI |
| `kube-prometheus-stack` | Prometheus + Grafana 모니터링 스택 |

### eks-node-viewer

실습 중에 노드의 리소스 요청량을 실시간으로 확인할 때 쓰는 유용한 도구다. 실제 사용량이 아니라 파드의 `requests` 합계를 기준으로 표시한다는 점을 기억해야 한다.

```bash
eks-node-viewer --resources cpu,memory
eks-node-viewer --extra-labels topology.kubernetes.io/zone
```

---

## 2. EKS 관리형 노드 그룹

EKS에서 노드는 단순히 EC2 인스턴스가 아니라 노드 그룹(Node Group) 단위로 관리된다. 각 노드 그룹은 서로 다른 인스턴스 타입, 용량 타입(온디맨드/스팟), 아키텍처(x86/ARM)를 가질 수 있다.

### 2-1. On-Demand 노드 그룹 (myeks-ng-1)

```
인스턴스 타입 : t3.medium
용량 타입     : ON_DEMAND
아키텍처      : amd64 (x86_64)
AMI           : Amazon Linux 2023
```

가장 일반적인 형태의 노드 그룹이다. 예측 가능한 비용과 안정적인 가용성이 장점이다.

### 2-2. Graviton(ARM) 노드 그룹 (myeks-ng-2)

![AWS Graviton Multi-Architecture](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2020/08/16/multi-arch-phases-1024x554.png)

> 출처: [Amazon EKS on AWS Graviton2 GA 블로그](https://aws.amazon.com/blogs/containers/eks-on-graviton-generally-available/) | 공식 자료: [AWS Graviton Getting Started](https://github.com/aws/aws-graviton-getting-started)

**실무 포인트**: Graviton 노드에 파드를 배포하려면 두 가지가 필요하다.

1. **nodeSelector** 또는 **nodeAffinity**: `kubernetes.io/arch: arm64`로 해당 노드를 명시적으로 지정
2. **tolerations**: 노드 그룹에 `cpuarch=arm64:NoExecute` Taint가 설정되어 있기 때문에, 이 Taint를 허용하는 toleration 없이는 파드가 스케줄되지 않는다

```yaml
tolerations:
- key: "cpuarch"
  operator: "Equal"
  value: "arm64"
  effect: "NoExecute"
```

**주의**: 컨테이너 이미지가 ARM 아키텍처를 지원하지 않으면 `exec format error`가 발생한다. Docker Hub에서 이미지의 지원 플랫폼을 반드시 확인해야 한다.

### 2-3. Spot 인스턴스 노드 그룹 (myeks-ng-3)

![EKS Managed Node Groups with Spot Instances](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2020/12/03/image-85.png)

> 출처: [Amazon EKS now supports Spot Instances in managed node groups](https://aws.amazon.com/blogs/containers/amazon-eks-now-supports-provisioning-and-managing-ec2-spot-instances-in-managed-node-groups/)

```
capacity_type  : SPOT
instance_types : ["c5a.large", "c6a.large", "t3a.large", "t3a.medium"]
```

여러 인스턴스 타입을 지정하는 게 중요하다. 하나의 인스턴스 타입에서 스팟 용량이 부족해도 다른 타입으로 대체가 가능하기 때문이다. 이를 **Instance Type Diversification**이라고 한다.

**관리형 노드 그룹에서 Spot 중단 처리**: AWS EKS 관리형 노드 그룹은 별도의 Node Termination Handler 없이도 Spot 중단을 자동으로 처리한다. Spot 중단 알림이 오면 해당 노드를 cordon → drain하고, 대체 노드를 미리 프로비저닝한다.

---

## 3. 스케일링 전략 전체 그림

쿠버네티스에서 스케일링은 크게 두 축으로 나뉜다: **파드를 늘리느냐** vs **노드를 늘리느냐**. 그리고 각각 수평/수직 확장이 있다.

```
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Autoscaling                 │
├──────────────────────┬──────────────────────────────────┤
│   Pod Level          │   Node Level                     │
├──────────────────────┼──────────────────────────────────┤
│ HPA                  │ CAS (Cluster Autoscaler)         │
│ - CPU/메모리 기준    │ - ASG 기반 노드 추가/삭제        │
│ - 파드 수 증감       │ - Pending 파드 감지 후 동작      │
├──────────────────────┼──────────────────────────────────┤
│ VPA                  │ Karpenter                        │
│ - requests 최적화    │ - EC2 Fleet API 직접 호출        │
│ - 파드 재시작 필요   │ - 수십 초 내 노드 프로비저닝    │
├──────────────────────┼──────────────────────────────────┤
│ KEDA                 │ Fargate                          │
│ - 이벤트 기반 확장  │ - 노드리스(Serverless)           │
│ - SQS, Kafka, Cron  │ - MicroVM 격리                  │
└──────────────────────┴──────────────────────────────────┘
```

### 스케일링 전략 의사결정

| 접근법 | 핵심 전략 | E2E 스케일링 시간 | 복잡도 |
|--------|-----------|-----------------|--------|
| 반응형 고속화 | Karpenter + KEDA + Warm Pool | 5~45초 | 매우 높음 |
| 예측형 스케일링 | CronHPA + Predictive Scaling | 사전 확장(0초) | 낮음 |
| 아키텍처 복원력 | SQS/Kafka + Circuit Breaker | 지연 허용 | 중간 |
| 적정 기본 용량 | 기본 replica 20~30% 증설 | 불필요 | 매우 낮음 |

실제 대부분의 서비스에서는 **예측형 스케일링**이나 **적정 기본 용량**이 가장 비용 효율적이다. "초고속 반응형"이 정말 필요한지를 먼저 물어봐야 한다.

---

## 4. HPA - Horizontal Pod Autoscaler

![HPA Architecture](https://kubernetes.io/images/docs/horizontal-pod-autoscaler.svg)

> 출처: [Kubernetes 공식 문서 - Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

### 동작 원리

HPA는 일정 주기(기본 15초)로 metrics-server에서 파드의 리소스 사용량을 가져와서 목표값과 비교한다. 현재 CPU 사용률이 목표치를 초과하면 파드를 늘리고, 한참 낮으면 줄인다.

```
원하는 레플리카 수 = ceil[현재 레플리카 수 × (현재 메트릭값 / 목표 메트릭값)]
```

예를 들어 목표가 50% CPU이고 현재 실제 CPU가 80%라면:
```
원하는 레플리카 수 = ceil[2 × (80 / 50)] = ceil[3.2] = 4
```

### HPA 생성 예시

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
```

### HPA 타이밍 이해

- **스케일 아웃(증가)**: 3분 안에 응답 (기본 30초 간격으로 체크 후 즉시)
- **스케일 인(감소)**: 5분 대기 후 감소 (급격한 제거를 방지하는 안정화 윈도우)

```bash
# 감소 대기 시간 변경
--horizontal-pod-autoscaler-downscale-stabilization=5m  # 기본값
```

### HPA 사용 시 실무 주의사항

**1. 평균의 함정**: 여러 파드 중 하나만 100% 상태이고 나머지가 낮으면 평균이 낮아서 스케일 아웃이 안 될 수 있다. 그 파드는 probe 실패 → 재시작 → 다른 파드에 부하 전가 → 연쇄 실패의 사이클이 생긴다.

**2. 초기화 지연 문제**: 자바 같은 JVM 기반 앱은 시작 시 CPU가 폭발적으로 올라간다. 이 시점에 HPA가 오버스케일할 수 있다. `--horizontal-pod-autoscaler-cpu-initialization-period` 옵션을 활용하거나, **Kube Startup CPU Boost** 기법을 검토해볼 만하다.

**3. 스파이크 대응 한계**: HPA가 인식 → 판단 → 파드 생성까지 최소 수십 초가 걸린다. 갑작스러운 스파이크에는 Over-Provisioning이나 KEDA Cron 트리거를 같이 쓰는 게 현실적이다.

### HPA 메트릭 타입

```yaml
metrics:
# CPU/메모리 (기본)
- type: Resource
  resource:
    name: cpu
    target:
      averageUtilization: 50

# 파드 커스텀 메트릭
- type: Pods
  pods:
    metric:
      name: packets-per-second
    target:
      averageValue: 1k

# 오브젝트 커스텀 메트릭 (Ingress의 RPS 등)
- type: Object
  object:
    metric:
      name: requests-per-second
    describedObject:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      name: main-route
    target:
      value: 10k

# 외부 메트릭 (SQS, Kafka 등)
- type: External
  external:
    metric:
      name: queue_messages_ready
    target:
      averageValue: 30
```

---

## 5. VPA - Vertical Pod Autoscaler

![VPA Architecture](https://kubernetes.io/images/docs/vertical-pod-autoscaler.svg)

> 출처: [Kubernetes 공식 문서 - Vertical Pod Autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/) | GitHub: [kubernetes/autoscaler - VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)


> 공식 GitHub: [kubernetes/autoscaler - VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)

### VPA가 하는 일

HPA가 "파드를 몇 개 띄울까"를 결정한다면, VPA는 "파드 하나에 CPU와 메모리를 얼마나 줄까"를 결정한다. VPA는 실제 사용량을 관찰해서 `resources.requests`의 최적값을 추천하고, 설정에 따라 자동으로 적용(파드 재시작)한다.

### VPA 구성 요소

- **Recommender**: 메트릭을 수집해서 권장 requests 값을 계산
- **Updater**: Auto 모드일 때 조건에 맞는 파드를 evict
- **Admission Controller**: 새 파드 생성 시 권장값으로 requests를 변경

### VPA 사용 모드

```yaml
spec:
  updatePolicy:
    updateMode: "Off"       # 권장값만 보여주고 자동 적용 안 함
    # updateMode: "Initial" # 파드 생성 시에만 권장값 적용
    # updateMode: "Auto"    # 실시간으로 evict 후 재시작 (기본값)
    # updateMode: "Recreate"# Auto와 유사하지만 Pod 업데이트도 포함
```

### HPA와 VPA를 함께 쓸 수 없는 이유

HPA와 VPA는 둘 다 CPU를 기준으로 동작하면 서로 충돌한다. VPA가 CPU requests를 늘리면 HPA는 "CPU 활용률이 낮아졌다"고 판단해서 파드를 줄이려 하고, 반대로 HPA가 파드를 늘리면 VPA는 "개별 파드의 CPU가 낮다"고 판단해서 requests를 줄이려 한다.

**공존 전략**: VPA는 메모리 기반으로, HPA는 커스텀 메트릭(RPS 등)으로 각각 다른 차원을 담당하면 함께 쓸 수 있다.

### VPA의 JVM 힙 사이즈 불일치 문제

```yaml
# 문제: 고정된 JVM 힙 설정
env:
- name: JAVA_OPTS
  value: "-Xmx2g"    # VPA가 메모리를 4Gi로 바꿔도 JVM은 2g만 씀 → 낭비

# 해결: 동적 힙 설정
env:
- name: JAVA_OPTS
  value: "-XX:MaxRAMPercentage=75.0"  # 컨테이너 메모리의 75%를 자동으로 힙에 할당
```

---

## 6. KEDA - 이벤트 기반 오토스케일링

![KEDA Architecture](https://keda.sh/img/keda-arch.png)

> 출처: [KEDA 공식 사이트](https://keda.sh/docs/2.16/concepts/)

### KEDA를 쓰는 이유

HPA는 CPU/메모리 같은 리소스 메트릭만 본다. 그런데 실제로 중요한 건 "지금 처리해야 할 작업이 얼마나 밀려 있냐"이다. SQS 큐에 메시지가 1000개 쌓여 있어도 현재 CPU가 낮다면 HPA는 스케일 아웃을 하지 않는다. KEDA는 이 문제를 해결한다.

### KEDA 구성 요소

| 컴포넌트 | 역할 |
|----------|------|
| `keda-operator` | 이벤트 소스를 감시하고 Deployment를 0 ↔ N으로 제어 |
| `keda-operator-metrics-apiserver` | 외부 이벤트 메트릭을 HPA가 읽을 수 있는 형태로 노출 |
| `keda-admission-webhooks` | ScaledObject 설정 검증 (잘못된 설정 방지) |

### ScaledObject - Cron 트리거 예시

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: php-apache-cron-scaled
spec:
  minReplicaCount: 0    # 트리거 조건 없으면 0개까지 줄일 수 있음
  maxReplicaCount: 2
  pollingInterval: 30   # 30초마다 트리거 조건 확인
  cooldownPeriod: 300   # 스케일 다운 전 대기 시간(초)
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Seoul
      start: 00,15,30,45 * * * *  # 매시 0, 15, 30, 45분에 스케일 아웃
      end: 05,20,35,50 * * * *    # 매시 5, 20, 35, 50분에 스케일 인
      desiredReplicas: "1"
```

### KEDA가 HPA와 다른 점

KEDA는 HPA를 대체하는 게 아니라 **HPA 위에서 동작**한다. KEDA가 외부 이벤트를 감지해서 HPA에 "원하는 레플리카 수는 N개야"라고 알려주고, 실제 파드 조정은 HPA가 한다. 그래서 0개까지 줄일 수 있는 것도 KEDA의 `ScaledObject`가 직접 Deployment를 0으로 만들기 때문이다.

### KEDA Scalers 종류

- `kafka`: Kafka 토픽의 lag 기준
- `aws-sqs-queue`: SQS 큐 메시지 수 기준
- `cron`: 시간 기반 (특정 시간대에 미리 스케일 아웃)
- `prometheus`: Prometheus 메트릭 기반
- `rabbitmq`: RabbitMQ 큐 기준

---

## 7. CPA - Cluster Proportional Autoscaler

### 개념

HPA가 "메트릭 기준으로 파드를 늘린다"면, CPA는 "노드 수에 비례해서 특정 파드를 늘린다"는 개념이다. 대표적인 사용 사례가 **CoreDNS**다.

```
노드 1개 → CoreDNS 1개
노드 2개 → CoreDNS 2개
노드 3개 → CoreDNS 3개
노드 5개 → CoreDNS 5개
```

> 공식 GitHub: [kubernetes-sigs/cluster-proportional-autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler)

### 설정 예시

```yaml
config:
  ladder:
    nodesToReplicas:
      - [1, 1]   # 노드 1개면 CoreDNS 1개
      - [2, 2]   # 노드 2개면 CoreDNS 2개
      - [3, 3]   # 노드 3개면 CoreDNS 3개
      - [4, 3]   # 노드 4개는 3개 유지 (여유 충분)
      - [5, 5]   # 노드 5개면 CoreDNS 5개
```

---

## 8. CAS - Cluster Autoscaler

![Cluster Autoscaler on EKS](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2020/12/03/image-85.png)

> 출처: [Cluster Autoscaler on AWS](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md) | 공식 GitHub: [kubernetes/autoscaler - Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)



### Auto-Discovery 설정

CAS는 특정 태그가 달린 ASG를 자동으로 찾아서 관리한다.

```bash
# EKS 노드에 자동으로 붙는 태그 (CAS가 이걸 보고 ASG를 찾음)
k8s.io/cluster-autoscaler/enabled: "true"
k8s.io/cluster-autoscaler/myeks: owned
```

### CAS 배포 핵심 설정

```
--cloud-provider=aws
--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/myeks
--expander=least-waste         # 리소스 낭비를 최소화하는 방식으로 노드 그룹 선택
--skip-nodes-with-local-storage=false  # 로컬 스토리지 있는 노드도 scale-down 허용
```

### CAS의 한계 (Karpenter 등장 배경)

1. **느린 스케일링**: CAS → ASG → EC2 Fleet API 경로를 거쳐서 실제 노드 준비까지 수 분 소요
2. **관리 복잡성**: 노드 그룹을 수동으로 관리해야 하고, 인스턴스 타입마다 별도 노드 그룹이 필요
3. **ASG 의존성**: k8s와 AWS ASG 두 곳에서 노드 정보를 각자 관리하면서 동기화 문제 발생
4. **특정 노드 삭제 어려움**: 특정 노드를 골라서 줄이는 게 구현하기 복잡함
5. **폴링 방식**: 너무 자주 체크하면 API 호출 제한에 걸릴 수 있음

### Scale-down 조건

CAS는 기본 **10분** 동안 노드 활용률이 낮을 때(50% 미만) 해당 노드의 파드를 다른 곳에 재배치하고 노드를 종료한다. `--scale-down-delay-after-add` 옵션으로 조정 가능하다.

---

## 9. Karpenter - Just-in-time 노드 프로비저닝

![Karpenter Architecture](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2021/11/23/2021-karpenter-diagram.png)

> 출처: [Introducing Karpenter - AWS News Blog](https://aws.amazon.com/blogs/aws/introducing-karpenter-an-open-source-high-performance-kubernetes-cluster-autoscaler/)

┌───────────────────────────────────────────────────────────────┐
│                    Karpenter vs CAS 비교                      │
├──────────────────────┬────────────────┬───────────────────────┤
│ 항목                 │ CAS            │ Karpenter             │
├──────────────────────┼────────────────┼───────────────────────┤
│ 노드 프로비저닝 경로 │ CAS→ASG→EC2   │ Karpenter→EC2 Fleet  │
│ 속도                 │ 수 분          │ 수십 초               │
│ 인스턴스 타입 유연성 │ 노드그룹별 고정│ 수백 가지 자동 선택  │
│ ASG 필요 여부        │ 필수           │ 불필요                │
│ Consolidation        │ 미지원         │ 기본 지원             │
│ 스팟 처리            │ 기본 미지원    │ 내장 지원             │
└──────────────────────┴────────────────┴───────────────────────┘
```

> 공식 사이트: [karpenter.sh](https://karpenter.sh) | GitHub: [aws/karpenter-provider-aws](https://github.com/aws/karpenter-provider-aws)

### Karpenter 핵심 개념

#### NodePool (구 Provisioner)

어떤 종류의 노드를 만들 수 있는지 "가드레일" 형태로 설정한다. Karpenter는 이 범위 안에서 파드 요구에 맞는 최적 인스턴스 타입을 스스로 선택한다.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]       # c, m, r 계열만 허용
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]                  # 3세대 이상만 허용
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 720h   # 30일 후 자동 만료 (Drift 유도)
  limits:
    cpu: 1000             # 클러스터 전체 CPU 한도
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m  # 활용도 낮은 노드는 1분 후 정리 시도
```

#### EC2NodeClass

보안 그룹, 서브넷, AMI 등 EC2 인스턴스의 구체적인 설정을 담는다.

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  role: "KarpenterNodeRole-my-cluster"
  amiSelectorTerms:
    - alias: "al2023@latest"   # 항상 최신 AL2023 AMI 사용
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "my-cluster"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "my-cluster"
```

### Karpenter 스케줄링 동작 흐름

```
1. 파드 생성 요청 → kube-scheduler가 스케줄 불가 판단
         ↓
2. 파드가 Pending 상태로 전환
         ↓
3. Karpenter Watch API로 감지
         ↓
4. 파드 스펙 분석 (resource requests, affinity, toleration 등)
         ↓
5. NodePool 요구사항 교집합으로 후보 인스턴스 타입 필터링
         ↓
6. 비용 기준으로 최적 인스턴스 타입 선택 (빈 패킹)
         ↓
7. EC2 Fleet API로 인스턴스 생성 (수십 초)
         ↓
8. 노드가 클러스터에 조인 → 파드 스케줄
```

### Disruption (중단) 메커니즘

Karpenter는 단순히 노드를 추가하는 것뿐만 아니라 **지속적으로 클러스터를 최적화**한다.

#### Consolidation (통합)
가장 중요한 기능이다. 활용도가 낮은 노드들의 파드를 다른 노드로 이동시키고, 빈 노드를 종료해서 비용을 절감한다.

```
Before:                          After:
┌────────────┐                   ┌────────────┐
│ 노드A: 20% │                   │ 노드A: 75% │
└────────────┘                   └────────────┘
┌────────────┐     Consolidation  (노드B 종료)
│ 노드B: 15% │  ───────────────►
└────────────┘                   (노드C 종료)
┌────────────┐
│ 노드C: 10% │
└────────────┘
```

#### Expiration (만료)
설정된 기간(`expireAfter`)이 지난 노드를 자동으로 교체한다. 이를 통해 항상 최신 AMI를 사용하고, 오래된 노드에 쌓일 수 있는 문제들을 자연스럽게 해소한다.

#### Drift (드리프트)
NodePool이나 EC2NodeClass의 설정이 변경되면, 그 변경사항과 맞지 않는 노드를 자동으로 교체한다. EKS 버전 업그레이드 시 매우 유용하다.

### Spot-to-Spot Consolidation 주의사항

스팟 인스턴스 간 통합이 작동하려면 NodePool에 **최소 15개 이상의 다양한 인스턴스 타입**이 포함되어야 한다. 그렇지 않으면 가용성이 낮고 중단 빈도가 높은 인스턴스를 선택할 위험이 있다.

### Over-Provisioning 전략

Karpenter를 써도 EC2 인스턴스가 준비되는 데 1~2분이 걸린다. 갑작스러운 스케일 아웃이 필요할 때 이 시간을 줄이려면 **우선순위가 낮은 더미 파드**를 미리 배치해 노드를 항상 준비 상태로 유지하는 방법을 쓴다.

```yaml
# 낮은 우선순위 클래스 생성
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: placeholder-priority
value: -10                    # 음수 우선순위 → 다른 파드에게 자리를 양보
preemptionPolicy: Never       # 먼저 선점하지 않음

---
# 더미 파드 배치 (실제 파드 오면 비켜줌)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: placeholder
spec:
  replicas: 5
  template:
    spec:
      priorityClassName: placeholder-priority
      terminationGracePeriodSeconds: 0
      containers:
      - name: ubuntu
        image: ubuntu
        command: ["sleep", "infinity"]
        resources:
          requests:
            cpu: 200m
            memory: 250Mi     # 실제 파드 크기보다 약간 작게
```

---

## 10. Fargate - Serverless 컨테이너 실행

![EKS Fargate Architecture](https://d2908q01vomqb2.cloudfront.net/fe2ef495a1152561572949784c16bf23abb28057/2019/12/16/fargate-eks-hpa-1024x518.png)

> 출처: [Autoscaling EKS on Fargate with custom metrics - AWS Containers Blog](https://aws.amazon.com/blogs/containers/autoscaling-eks-on-fargate-with-custom-metrics/)  
> 공식 문서: [AWS EKS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html) | Firecracker: [firecracker-microvm/firecracker](https://github.com/firecracker-microvm/firecracker)


> 공식 문서: [AWS EKS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)  
> Firecracker GitHub: [firecracker-microvm/firecracker](https://github.com/firecracker-microvm/firecracker)

### Fargate의 핵심 아이디어

일반적인 EKS에서는 EC2 인스턴스(워커 노드)를 프로비저닝하고 관리해야 한다. Fargate를 쓰면 EC2를 직접 관리할 필요가 없다. 파드를 배포하면 AWS가 알아서 그 파드를 실행할 전용 MicroVM을 만들어준다.

### Firecracker MicroVM

Fargate의 핵심 기술이다. AWS가 개발한 오픈소스 경량 가상화 기술로, KVM을 사용해 **125ms 이내**에 MicroVM을 시작할 수 있다. 각 파드는 완전히 격리된 커널, CPU, 메모리, ENI를 갖는다. 다른 파드와 공유하는 것이 없다.

### Fargate Profile 설정

```yaml
fargate_profiles:
  study_wildcard:
    selectors:
      - namespace: "study-*"   # study-로 시작하는 네임스페이스의 파드는 모두 Fargate로

  kube_system:
    selectors:
      - namespace: "kube-system"
```

### Fargate 파드의 특이점

Fargate 파드는 배포하면 자동으로 **schedulerName이 `fargate-scheduler`로 변경**된다. 일반 파드의 기본 스케줄러(`default-scheduler`)와 다르다.

```bash
# Fargate 파드의 용량 어노테이션 확인
kubectl get pod -n study-aews \
  -o jsonpath='{.items[0].metadata.annotations.CapacityProvisioned}'
# → 0.5vCPU 1GB  (자동으로 적절한 크기로 반올림된 결과)
```

### Fargate 파드 리소스 할당 규칙

Fargate는 요청한 CPU와 메모리의 합에 **256MB를 추가**한 뒤, 이를 수용할 수 있는 가장 가까운 조합으로 **반올림**한다.

예) 1 vCPU + 8GB 요청 → 256MB 추가 → 1 vCPU + 8.25GB → 이 조합 없음 → **2 vCPU + 9GB**로 프로비저닝 (비용도 이 기준으로 청구)

| vCPU | 메모리 선택지 |
|------|-------------|
| 0.25 | 0.5 GB, 1 GB, 2 GB |
| 0.5  | 1 ~ 4 GB |
| 1    | 2 ~ 8 GB |
| 2    | 4 ~ 16 GB |
| 4    | 8 ~ 30 GB |

### Fargate 핵심 제약사항 (실무에서 중요)

| 제약사항 | 설명 |
|---------|------|
| DaemonSet 미지원 | 사이드카 컨테이너로 대체해야 함 |
| EBS 마운트 불가 | EFS는 가능 (자동 마운트) |
| Spot 미지원 | Fargate는 항상 온디맨드 |
| ARM 프로세서 미지원 | amd64 아키텍처만 지원 |
| private 서브넷 필수 | NAT Gateway 필요 |
| IMDS 접근 불가 | IAM 권한은 반드시 IRSA 사용 |
| 특권 컨테이너 불가 | `privileged: true` 설정 불가 |
| SSH 접속 불가 | 노드 OS에 직접 접근 안 됨 |
| GPU 미지원 | - |
| topologySpreadConstraints 미지원 | - |

### Fargate 로깅

Fargate는 사이드카 없이 **내장 Fluent Bit** 로그 라우터를 제공한다. `aws-observability` 네임스페이스에 ConfigMap을 만들면 자동으로 로그 라우팅이 설정된다.

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: aws-logging
  namespace: aws-observability   # 반드시 이 네임스페이스여야 함
data:
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match kube.*
        region ap-northeast-2
        log_group_name /my-cluster/fargate-logs
        auto_create_group true
```

---

## 정리: 언제 무엇을 써야 할까

```
파드 자동화가 필요하다
  ├─ CPU/메모리 기준 → HPA
  ├─ 이벤트/큐 기준 → KEDA
  ├─ requests 최적화 → VPA (Off 모드로 권장값만 보는 것도 유용)
  └─ DNS 같은 인프라 파드 → CPA

노드 자동화가 필요하다
  ├─ 단순하고 안정적 → CAS (ASG 기반)
  ├─ 빠르고 비용 최적화 → Karpenter
  └─ 노드 관리 자체가 싫다 → Fargate

단, Fargate는 제약이 많아서 범용 솔루션이 아님.
DaemonSet, EBS, Spot, GPU, SSH가 필요하면 선택 불가.
```

