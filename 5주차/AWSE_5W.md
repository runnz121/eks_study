# [AEWS 4기] VPA + HPA + JVM 힙 충돌 — 백엔드 개발자가 EKS 오토스케일링에서 놓치기 쉬운 함정

> EKS에서 VPA와 HPA를 같이 쓰는 구성은 문서만 읽으면 "각자 다른 일 하니까 같이 써도 되겠네" 싶다.
> 그런데 실제로 JVM 기반 애플리케이션 위에 이 둘을 얹으면 **조용히 리소스가 새고, 파드가 흔들리고, 알람은 안 울리는** 이상한 상태에 빠진다.
> 이번 글은 그 함정을 재현해 보고, 운영 관점에서 어떻게 피해야 하는지 정리한 기록이다.

---

## 목차

1. [전제 — VPA와 HPA 각자의 동작 방식](#1-전제--vpa와-hpa-각자의-동작-방식)
2. [함정 1: VPA와 HPA가 같은 리소스를 바라볼 때의 충돌](#2-함정-1-vpa와-hpa가-같은-리소스를-바라볼-때의-충돌)
3. [함정 2: JVM 고정 힙과 VPA의 리소스 재조정 불일치](#3-함정-2-jvm-고정-힙과-vpa의-리소스-재조정-불일치)
4. [진단 방법](#4-진단-방법)
5. [해결 전략](#5-해결-전략)
6. [운영 관점 체크리스트](#6-운영-관점-체크리스트)
7. [정리](#7-정리)

---

## 1. 전제 — VPA와 HPA 각자의 동작 방식

### HPA (Horizontal Pod Autoscaler)

파드의 **개수**를 늘리고 줄인다. 기본 지표는 CPU 사용률인데, 여기서 "사용률"은 절대값이 아니라 **requests 대비 실제 사용량의 비율**이다.

```
활용률 = 현재 CPU 사용량 / 파드 CPU requests
```

targetCPUUtilization이 70%라면, 파드 CPU requests가 `500m`일 때 실제 사용량이 `350m`를 넘으면 스케일 아웃이 트리거된다.

### VPA (Vertical Pod Autoscaler)

파드의 **크기(requests/limits)** 를 조정한다. 과거 사용량을 기반으로 "이 파드는 CPU 300m, 메모리 1Gi면 충분하다" 또는 "부족하니 CPU 1000m, 메모리 4Gi로 올려라"는 추천을 만들고, 모드에 따라 실제로 적용한다.

| 모드 | 동작 |
|------|------|
| `Off` | 추천만 생성 (수동 적용용) |
| `Initial` | 파드 생성 시점에만 적용 |
| `Auto` | 파드를 재시작하면서 requests/limits 자동 조정 |

여기까지만 보면 HPA는 개수, VPA는 크기 — 역할이 겹치지 않아 보인다. 문제는 **둘 다 같은 지표(CPU/메모리)를 근거로 판단한다는 점**이다.

---

## 2. 함정 1: VPA와 HPA가 같은 리소스를 바라볼 때의 충돌

### 2.1 충돌 시나리오

상황 설정:

- 파드 CPU requests: `500m`
- HPA targetCPUUtilization: `70%` (= 350m 넘으면 스케일 아웃)
- VPA 모드: `Auto`
- 트래픽은 꾸준히 들어와 CPU 사용량이 평균 `400m` 수준

처음엔 HPA가 "활용률 80%네, 파드 늘리자" 하고 레플리카를 늘린다. 파드 개수가 늘면 파드당 부하는 줄고 사용량은 `300m` 정도로 내려간다. 여기까지는 정상.

그런데 VPA는 이 상황을 다르게 해석한다. **"파드당 실제 사용량이 `400m` 언저리였으니 requests를 `500m`에서 더 올려야겠다."** VPA가 requests를 `1000m`로 올린다.

requests가 올라간 순간, HPA의 계산식이 뒤집힌다.

```
이전: 활용률 = 400m / 500m = 80%  → 스케일 아웃
현재: 활용률 = 400m / 1000m = 40% → 스케일 다운
```

CPU **실제 사용량은 그대로인데** HPA는 "활용률 떨어졌네" 하고 파드를 줄이기 시작한다. 파드가 줄어들면 남은 파드 부하가 다시 `500~600m`로 치솟고, VPA는 "requests 더 올려야겠다" 하고 재조정한다.

이 과정이 조용히 반복된다. 겉보기엔 오토스케일링이 동작하는 것처럼 보이지만, 실제로는 **리소스는 계속 증가하고 파드 수는 줄어드는 방향**으로 수렴한다. 비용은 오르는데 처리량은 늘지 않는 이상한 상태다.

### 2.2 이 문제가 알람에 잘 안 잡히는 이유

- HPA도 "정상 동작" 중이다. desiredReplicas를 계산해서 반영하고 있다.
- VPA도 "정상 동작" 중이다. recommendation을 생성해서 적용하고 있다.
- 파드는 계속 뜨고 내려가지만 **Pending으로 막히거나 OOMKilled가 나지는 않는다**. 에러 이벤트가 없다.
- 문제는 **집계 레벨**에서만 드러난다: 리소스 사용 총량이 슬금슬금 오르고, 파드 평균 개수가 기대보다 적다.

프로덕션에서 "왜 비용이 이렇게 오르지?"로 늦게 발견되는 유형의 장애다.

---

## 3. 함정 2: JVM 고정 힙과 VPA의 리소스 재조정 불일치

이번엔 충돌이 아니라 **아무 효과가 없는 상태**를 만드는 함정이다. Java/Kotlin 기반 서비스에서 특히 자주 나온다.

### 3.1 문제의 Deployment 예시

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: api
        image: my-spring-boot-app:latest
        resources:
          requests:
            memory: "2Gi"
          limits:
            memory: "4Gi"
        env:
        - name: JAVA_OPTS
          value: "-Xms2g -Xmx2g"   # ← JVM 힙이 2GB로 고정되어 있음
```

VPA가 동작 방식을 보자.

1. VPA가 과거 데이터를 분석하고 "이 서비스는 메모리 `4Gi`가 적정하다"고 판단
2. 파드 재시작과 함께 requests/limits를 `4Gi`로 조정
3. 컨테이너는 `4Gi` 할당받음
4. **하지만 JVM은 여전히 `-Xmx2g`만 쓴다**

결과적으로 컨테이너에 할당된 `4Gi` 중 실제로 JVM이 쓰는 건 `2Gi`뿐이다. 나머지 `2Gi`는 그냥 놀고 있다. GC 압박은 `2Gi` 기준 그대로 있고, OOM 위험도 여전하다.

### 3.2 왜 이 조합이 흔한가

JVM은 컨테이너 환경을 늦게 인식한 언어다. Java 8 초기에는 `/proc/meminfo`만 보고 호스트 전체 메모리를 기준으로 힙을 잡아서 cgroup limit을 무시하고 OOM을 냈다. 그 영향으로 팀들이 **방어적으로 `-Xmx`를 고정하는 관습**이 생겼다. 지금 이 설정이 VPA와 충돌한다.

### 3.3 조용한 비용 누수

VPA의 근거가 되는 메모리 사용량은 `container_memory_working_set_bytes`(cgroup 기준)다. JVM 힙 밖에서 쓰는 Metaspace, Direct Memory, native thread stack 등도 여기 잡힌다. VPA는 "이 파드 메모리 부족하네" 판단해서 requests를 계속 올리지만, **실제로는 힙이 부족한 게 아니라 힙 바깥 영역에서 쓰고 있는 것**일 수 있다. 진짜 원인(예: Direct Buffer 누수)을 덮은 채로 리소스만 커지는 상태가 된다.

---

## 4. 진단 방법

### 4.1 VPA recommendation 확인

```bash
kubectl get vpa
kubectl describe vpa <vpa-name>
```

`Recommendation` 섹션에서 target / lowerBound / upperBound 값이 어떻게 움직이는지를 본다. target이 시간이 갈수록 계속 우상향한다면 함정에 빠졌을 가능성이 있다.

### 4.2 HPA desiredReplicas 흔들림 확인

```bash
kubectl describe hpa <hpa-name> | grep -A5 "Events"
```

`SuccessfulRescale` 이벤트가 **아주 짧은 간격으로 스케일 아웃 ↔ 스케일 다운을 반복**하면 충돌 신호다.

### 4.3 JVM 힙 vs 컨테이너 메모리 비교

```bash
# 컨테이너 메모리 사용량
kubectl top pod <pod-name> --containers

# JVM 힙 설정 확인 (파드 안에서)
kubectl exec -it <pod-name> -- \
  jcmd 1 VM.flags | grep -E "MaxHeapSize|InitialHeapSize"

# JVM 실제 힙 사용량
kubectl exec -it <pod-name> -- \
  jcmd 1 GC.heap_info
```

컨테이너 메모리는 `3.5Gi`를 쓰는데 JVM MaxHeapSize가 `2g`라면, 불일치 상태다.

### 4.4 활용률과 requests의 비율이 비정상적으로 낮은지 확인

```bash
# 현재 requests
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].resources.requests}'

# 실제 사용량
kubectl top pod <pod-name>
```

활용률이 지속적으로 20~30% 이하로 떨어져 있다면 "VPA가 올려놓은 requests에 비해 실사용이 낮다"는 뜻이다.

---

## 5. 해결 전략

### 5.1 JVM을 컨테이너 메모리에 반응하게 만들기

고정 힙(`-Xmx2g`) 대신 **컨테이너 메모리에 비례**하도록 설정한다.

```yaml
env:
- name: JAVA_OPTS
  value: "-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0"
```

컨테이너 메모리가 `4Gi`면 힙 최대 `3Gi`, `8Gi`면 힙 최대 `6Gi`로 자동으로 따라간다. 나머지 25%는 Metaspace/Direct Memory/스레드 스택 몫으로 남긴다.

> 참고: Java 10+부터 `-XX:+UseContainerSupport`가 기본값이라 cgroup limit을 정상적으로 인식한다. Java 8이라면 `8u191` 이상으로 올리고 해당 플래그를 명시해야 한다.

### 5.2 VPA와 HPA가 서로 다른 차원을 담당하게 분리

| 전략 | VPA가 담당 | HPA가 담당 |
|------|-----------|------------|
| **권장** | 메모리만 (`controlledResources: ["memory"]`) | CPU 또는 커스텀 메트릭 |
| 대안 | Off 모드로 추천만 받기 | 단독 사용 |

VPA 스펙에서 대상 리소스를 명시적으로 메모리로 제한한다.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: api
      controlledResources: ["memory"]   # ← CPU는 건드리지 않음
```

이렇게 두면 HPA는 CPU 활용률 기준으로 파드 개수를 조정하고, VPA는 메모리 requests만 조정한다. 두 컨트롤러가 같은 축을 두고 싸우지 않는다.

### 5.3 HPA를 커스텀 메트릭으로 전환

가능하다면 HPA 기준을 **CPU가 아닌 애플리케이션 레벨 지표**로 바꾸는 게 가장 깔끔하다. 예를 들어 초당 요청 수(RPS), 큐 대기 길이, p99 레이턴시 같은 지표는 requests 값에 영향을 받지 않기 때문에 VPA가 requests를 아무리 건드려도 HPA 계산에 영향을 주지 않는다.

KEDA(Kubernetes Event-driven Autoscaling)를 도입하면 Kafka lag, SQS queue depth 같은 외부 이벤트 소스를 직접 HPA 메트릭으로 쓸 수 있다.

### 5.4 VPA를 Off 모드로 시작하기

프로덕션에서 처음부터 Auto로 돌리지 말고 Off 모드로 **추천값만 관찰**한다.

```yaml
spec:
  updatePolicy:
    updateMode: "Off"   # 추천만 생성
```

일주일 정도 돌려보고 추천값이 안정적으로 수렴하면 그때 Initial 또는 Auto로 올린다. 충돌 여부를 사전에 확인할 수 있는 가장 안전한 방법이다.

---

## 6. 운영 관점 체크리스트

VPA를 도입하기 전에 반드시 확인할 항목들이다.

**애플리케이션 레벨 고정 리소스 설정이 있는가?**

- JVM: `-Xmx`, `-Xms` 고정값
- Node.js: `--max-old-space-size`
- Python: worker 프로세스 수 하드코딩 (gunicorn `-w 4` 등)
- Go: `GOMEMLIMIT`, `GOMAXPROCS` 고정

이런 설정이 있으면 VPA가 아무리 컨테이너 크기를 바꿔도 애플리케이션이 따라오지 못한다.

**HPA 메트릭이 VPA가 조정하는 리소스와 겹치지 않는가?**

- VPA가 CPU를 조정하면 → HPA는 CPU 쓰지 말 것
- VPA가 메모리를 조정하면 → HPA는 메모리 쓰지 말 것
- 가능하면 HPA는 커스텀 메트릭 사용

**VPA 추천값의 변동성을 모니터링하는가?**

- Prometheus로 `kube_verticalpodautoscaler_status_recommendation_containerrecommendations_target`를 수집하고
- 변동폭이 지속적으로 커지면 알람

**파드 재시작 빈도가 정상 범위인가?**

- VPA `Auto` 모드는 requests 변경 시 파드를 재시작한다
- 너무 자주 재시작되면 서비스 가용성에 영향. `minReplicas` 확보 및 PDB(PodDisruptionBudget) 설정 필수

---

## 7. 정리

VPA와 HPA는 각자 단독으로 보면 잘 만들어진 컨트롤러다. 문제는 **두 컨트롤러가 같은 지표를 근거로 서로 다른 목적의 결정을 내린다**는 점에서 발생한다. 여기에 JVM처럼 컨테이너 리소스를 따라가지 않는 런타임이 끼면, 리소스는 늘어나는데 실제로 쓰이지 않는 조용한 낭비 상태가 만들어진다.

이번 글에서 다룬 세 가지 포인트를 요약하면 이렇다.

- **VPA + HPA가 같은 차원(CPU/메모리)을 공유하면 서로의 결정을 뒤집는다.** 메모리는 VPA, CPU는 HPA처럼 책임을 명확히 나눠야 한다.
- **JVM의 `-Xmx` 고정은 VPA의 메모리 조정을 무력화한다.** `-XX:MaxRAMPercentage`로 컨테이너 메모리에 반응하게 만들어야 한다.
- **이 유형의 장애는 에러가 나지 않는다.** 비용과 리소스 집계 레벨에서만 드러나기 때문에 사후가 아니라 도입 전 설계에서 막아야 한다.

EKS 위에 JVM 서비스를 올린 팀이라면 한 번쯤 점검해 볼 만한 조합이다. 특히 "VPA 켰는데도 메모리 부족 알람이 줄지 않는다"는 증상이 있다면 가장 먼저 의심해볼 지점이기도 하다.

---

## 참고

- [Kubernetes VPA - Known limitations](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#known-limitations)
- [JVM container awareness (-XX:+UseContainerSupport)](https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html)
- [KEDA - Custom metric based scaling](https://keda.sh/docs/latest/concepts/scaling-deployments/)
