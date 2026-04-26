# [AEWS 6주차] GitOps로 구현하는 멀티테넌트 SaaS 플랫폼 엔지니어링

> Amazon EKS 위에서 **Flux v2 + Tofu Controller + Helm + Argo Workflows** 조합으로 SaaS 멀티테넌트 환경의 온보딩/오프보딩을 완전 자동화하는 패턴을 정리한 글.
>
> 출처: [AWS Workshop - Building SaaS applications on Amazon EKS using GitOps](https://catalog.workshops.aws/eks-saas-gitops/en-US)

---

## 📚 목차

1. [학습 핵심](#1-학습-핵심)
2. [DevOps → GitOps → 플랫폼 엔지니어링](#2-devops--gitops--플랫폼-엔지니어링)
3. [GitOps 4원칙](#3-gitops-4원칙)
4. [Amazon EKS와 GitOps 도구 비교](#4-amazon-eks와-gitops-도구-비교)
5. [SaaS DevOps에 GitOps가 필요한 이유](#5-saas-devops에-gitops가-필요한-이유)
6. [실습 1: Terraform 모듈 + Tofu Controller + Helm](#6-실습-1-terraform-모듈--tofu-controller--helm)
7. [실습 2: SaaS 티어 전략 (Basic/Advanced/Premium)](#7-실습-2-saas-티어-전략)
8. [실습 3: Argo Workflows로 온보딩/오프보딩 자동화](#8-실습-3-argo-workflows로-온보딩오프보딩-자동화)
9. [참고: Argo 생태계 도구들](#9-참고-argo-생태계-도구들)
10. [회고](#10-회고)

---

## 1. 학습 핵심

이번 주차는 단순히 "GitOps 도구 써보기"가 아니라, **SaaS 비즈니스 요구사항(테넌트별 격리, 티어 전략, 자동화된 온보딩)을 GitOps 패러다임으로 어떻게 풀어내는가**가 진짜 주제였다.

핵심 키워드 4가지:

- **GitOps 4원칙**과 SaaS 환경 시나리오에서의 CI/CD 적용
- **Flux v2**로 Git 저장소와 클러스터 상태 동기화
- **Flux Tofu Controller + Terraform**으로 테넌트별 AWS 인프라 자동 프로비저닝
- **Helm 차트**를 활용한 멀티테넌트 SaaS 배포 패턴

목표 아키텍처:

![GitOps SaaS Architecture](https://aws-solutions-library-samples.github.io/_images/eks-saas-gitops-arch.png)

> 위 이미지가 안 보일 경우 [공식 솔루션 가이드](https://aws-solutions-library-samples.github.io/compute/building-saas-applications-on-amazon-eks-using-gitops.html)에서 확인 가능.

---

## 2. DevOps → GitOps → 플랫폼 엔지니어링

### 진화의 흐름

| 단계 | 핵심 가치 | 한계 |
|---|---|---|
| **DevOps** | 개발/운영 협업, 자동화 | 도구 분산, 일관성 부족 |
| **GitOps** | Git을 단일 진실의 소스로 | 셀프서비스 추상화 부재 |
| **Platform Engineering** | 개발자 추상화 + 셀프서비스 IDP | (조직 차원 투자 필요) |

### 플랫폼 엔지니어링의 3대 이점

- **속도(Velocity)**: 셀프서비스 배포로 아이디어 → 프로덕션까지의 시간 단축
- **거버넌스(Governance)**: 보안/신뢰성/확장성 요구사항을 플랫폼 차원에서 자동 강제
- **효율성(Efficiency)**: 멀티테넌시로 인프라 비용 절감, 전문성 중앙 집중화

### 플랫폼 구현 패턴 (AWS 관점)

- **Account as a Service**: 계정 단위 자유도
- **Template as a Service**: 표준화된 템플릿
- **Cluster as a Service**: EKS 클러스터를 서비스로
- **Namespace as a Service**: 공유 클러스터 내 격리 (Salesforce 등)
- **Platform as a Service**: 완전 추상화

### 국내 사례

- [당근 개발자 플랫폼](https://speakerdeck.com/outsider/danggeun-gaebalja-peulraespomeun-eoddeon-munjereul-haegyeolhago-issneunga)
- [무신사 플랫폼 엔지니어링](https://youtu.be/9FKbQRu6lVs)

---

## 3. GitOps 4원칙

[OpenGitOps.dev](https://opengitops.dev) 가 정의한 4가지 원칙:

1. **Declarative (선언적 정의)**
   - "어떻게(How)"가 아니라 "어떤 상태(What)"를 선언
   - 예: "서버 3대 실행하라" → YAML로 최종 상태 기술

2. **Versioned and Immutable (버전 관리 + 불변성)**
   - 모든 선언적 설정은 Git에 저장
   - 변경 이력 추적 + 롤백 용이

3. **Pulled Automatically (자동 반영)**
   - Git에 저장된 Desired State를 시스템이 스스로 감지/배포

4. **Continuously Reconciled (지속적 조정)**
   - 실제 상태(Actual State) ↔ 희망 상태(Desired State) 차이 감지 시 자동 복구

### 핵심 메커니즘: Auto Reconciliation

`kubectl edit` 같은 수동 변경이 발생해도 GitOps 에이전트가 감지하고 **Git에 정의된 원래 상태로 자동 복구**한다. 이것이 시스템 일관성의 핵심.

---

## 4. Amazon EKS와 GitOps 도구 비교

[AWS 공식 문서](https://docs.aws.amazon.com/prescriptive-guidance/latest/eks-gitops-tools/introduction.html) 기준 정리.

### 주요 도구 매트릭스

| 도구 | K8s 특화 | 핵심 특징 | 적합 상황 |
|---|---|---|---|
| **Argo CD** | ✅ 매우 강함 | Web UI, App-of-Apps, RBAC/SSO 내장 | UI 기반 멀티 클러스터 운영 |
| **Flux v2** | ✅ 매우 강함 | CLI 중심, Tofu Controller, CNCF 졸업 | GitOps + IaC 통합, SaaS |
| **Weave GitOps** | ✅ 강함 | Flux 기반 UI 레이어 | Flux + Web UI 동시 필요 |
| **Rancher Fleet** | ✅ 강함 | 수천 클러스터 관리 특화 | 대규모 클러스터 중앙 관리 |
| **Spinnaker** | 🔶 보통 | 멀티 클라우드, 카나리/블루그린 | 복잡한 배포 전략 |

### Argo CD vs Flux 핵심 차이

| 영역 | Argo CD | Flux |
|---|---|---|
| 아키텍처 | E2E 애플리케이션 | K8s CRD + 컨트롤러 |
| GUI | 풀스택 Web UI | CLI + 경량 UI |
| RBAC | 세분화된 자체 제어 | K8s 네이티브 활용 |
| 멀티 클러스터 | 매우 강함 | 멀티테넌시에 강함 |
| 부분 동기화 | ✅ | ❌ |
| 확장성 | 커스텀 플러그인 | 커스텀 컨트롤러 + 서드파티 통합 |

### 🔥 re:Invent 2025 신규: EKS Managed Argo CD

| 구분 | Self-Managed | EKS Managed |
|---|---|---|
| 설치/관리 | 사용자 직접 | AWS 콘솔에서 활성화 |
| 책임 | 사용자 전부 | AWS가 패치/모니터링 |
| 확장성 | 직접 관리 | AWS 자동 확장 |
| 인증 | Dex/OIDC | AWS Identity Center 통합 |
| 클러스터 지원 | EKS 외 모든 K8s | EKS 전용 (ARN 기반) |

관련 글: [Announcing Amazon EKS capabilities](https://aws.amazon.com/blogs/containers/deep-dive-streamlining-gitops-with-amazon-eks-capability-for-argo-cd/)

---

## 5. SaaS DevOps에 GitOps가 필요한 이유

### SaaS의 본질 = 민첩성

SaaS는 운영자가 직접 소프트웨어를 실행/관리하는 모델이라 다음이 필수:

- **잦은 릴리스**로 빠른 피드백
- **운영 효율성**: 모든 테넌트를 일관된 프로세스로 관리
- **자동화된 온보딩**: 신속/일관된 테넌트 생성

### SaaS 배포 모델 비교

| 모델 | 설명 | 격리 | 비용 | 활용 |
|---|---|---|---|---|
| **Silo** | 테넌트별 전용 인프라 | 높음 | 높음 | Premium 티어 |
| **Pool** | 인프라 공유 | 낮음 | 낮음 | Basic 티어 |
| **Hybrid** | 티어별 혼합 | 중간 | 중간 | 실제 SaaS 대부분 |

### 레퍼런스 아키텍처 (Flux v2 기반)

핵심 구성:

- **Terraform 모듈**: DB, 큐 등 애플리케이션 인프라 패키징
- **Helm 차트**: K8s 리소스 + 애플리케이션 이미지 패키징
- **Flux v2**: Git을 immutable한 진실의 소스로 사용
- **Flux Tofu Controller**: 기존 Terraform 모듈을 수정 없이 재사용

샘플 테넌트 앱 구조:

```
[Producer] --(메시지)--> [SQS 큐] --(폴링)--> [Consumer] --(저장)--> [DynamoDB]
```

- **Producer**: API 호출 수신 → 메시지 생성 → SQS 큐 전송
- **Consumer**: SQS 큐에서 메시지 fetch → DynamoDB 저장

---

## 6. 실습 1: Terraform 모듈 + Tofu Controller + Helm

> 실습 명령어 모음은 별도 [실습 가이드 문서](./EKS_GitOps_6주차_실습.md) 참조.

### 6.1 왜 개발자 주도형 인프라(Developer-Driven Infrastructure)인가?

전통적인 조직 구조의 병목:

```
개발자 → 인프라팀에 IAM/SQS/DynamoDB 요청 → 티켓 큐 적체 → 배포 지연
```

테넌트가 많아질수록 이 문제는 **기하급수적으로 악화**된다.

해결책 = **사전 정의된 가드레일 안에서 개발자가 직접 인프라 셀프서비스**

```
Terraform 모듈 → Tofu 컨트롤러 → Helm Chart → HelmRelease → 배포
```

> 💡 Terraform 공식 문서: *"모듈은 구성의 일관성을 유지하고, 복잡한 구성을 더 쉽게 이해할 수 있도록 하며, 모범 사례가 적용되도록 보장한다."*

### 6.2 Terraform 모듈 구조

```
gitops-gitea-repo/terraform/modules/
├── flux_cd/           # EKS에 Flux 설치 리소스
├── gitea/             # Gitea 저장소 리소스
├── gitops-saas-infra/ # 워크숍 전체 인프라
└── tenant-apps/       # ⭐ 테넌트 앱 전용 인프라
```

**핵심은 `tenant-apps` 모듈**: 변수 하나(`enable_producer`, `enable_consumer`)로 SQS, DynamoDB, IRSA 생성 범위를 제어.

```hcl
module "test_tenant_apps" {
  source          = "./terraform/modules/tenant-apps"
  tenant_id       = "test"
  enable_producer = true
  enable_consumer = true
}
```

- `enable_producer = true, enable_consumer = true` → **Plan: 11 to add**
- `enable_producer = false` → **Plan: 10 to add**

→ 모듈 변수 하나로 배포 범위가 명확하게 제어된다. 이것이 **추상화의 가치**.

### 6.3 Tofu Controller 통합 (GitOps × Terraform)

수동 `terraform apply`를 GitOps로 자동화하는 흐름:

```
1. Git에 Terraform CRD 파일 추가 (git push)
   ↓
2. Flux가 변경 감지 → 조정 시작
   ↓
3. tf-controller가 Terraform CRD 감지
   ↓
4. tf-runner Pod 실행 → Terraform 모듈 pull
   ↓
5. Terraform apply → SQS, DynamoDB, IRSA 생성
   ↓
6. 실행 상태/플랜을 K8s Secret으로 저장
```

핵심 매니페스트:

```yaml
apiVersion: infra.contrib.fluxcd.io/v1alpha2
kind: Terraform
metadata:
  name: example-tenant
  namespace: flux-system
spec:
  path: ./terraform/modules/tenant-apps
  interval: 1m
  approvePlan: auto
  destroyResourcesOnDeletion: true   # ⭐ CRD 삭제 시 AWS 리소스도 자동 정리
  sourceRef:
    kind: GitRepository
    name: terraform-v0-0-1
  vars:
    - name: tenant_id
      value: example-tenant
    - name: enable_producer
      value: true
    - name: enable_consumer
      value: true
  writeOutputsToSecret:
    name: example-tenant-infra-output
```

플랜/스테이트는 K8s Secret으로 저장됨:

```bash
$ kubectl get secret -n flux-system | grep -E 'tfplan|tfstate'
tfplan-default-example-tenant    Opaque   1   6m
tfstate-default-example-tenant   Opaque   1   6m
```

### 6.4 Helm 차트 구조

```
helm-charts/
├── helm-tenant-chart/   # 테넌트 앱 (Producer + Consumer 묶음)
└── application-chart/   # 개별 앱 (Onboarding Service 등)
```

| 차트 | 용도 | 특징 |
|---|---|---|
| `helm-tenant-chart` | 테넌트별 앱 배포 | `tenant_id`로 격리 |
| `application-chart` | 개별 애플리케이션 | 테넌트 구분 없음 |

### 6.5 HelmRelease로 GitOps 통합

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: example-tenant-premium
  namespace: flux-system
spec:
  releaseName: example-tenant-premium
  targetNamespace: example-tenant
  chart:
    spec:
      chart: helm-tenant-chart
      version: "0.x"
      sourceRef:
        kind: HelmRepository
        name: helm-tenant-chart
  values:
    tenantId: example-tenant
    apps:
      producer:
        enabled: true   # Silo: 전용 deployment
      consumer:
        enabled: true
```

### 6.6 핵심 패턴 요약

| 단계 | 도구 | 핵심 개념 |
|---|---|---|
| Terraform 모듈 테스트 | Terraform | 인프라 추상화 |
| Tofu Controller 통합 | Flux + Terraform CRD | GitOps로 IaC 자동화 |
| Helm 차트 테스트 | Helm | 앱 패키징 |
| HelmRelease 배포 | Flux + HelmRelease | 선언적 배포 관리 |

> 💡 **모든 과정의 본질은 동일**:
> ```
> Git에 파일 추가/수정 → git push → Flux 감지 → 자동 반영
> Git에서 파일 삭제   → git push → Flux 감지 → 리소스 자동 정리
> ```

---

## 7. 실습 2: SaaS 티어 전략

### 7.1 티어 템플릿 구조

```
tier-templates/
├── basic_env_template.yaml
├── basic_tenant_template.yaml
├── advanced_tenant_template.yaml   # ⭐ 직접 생성
└── premium_tenant_template.yaml
```

### 7.2 티어별 비교 매트릭스

| 구성 요소 | Basic | Advanced | Premium |
|---|---|---|---|
| **배포 모델** | Pool (공유) | Hybrid | Silo (전용) |
| **Producer** | 공유 (pool-1) | **공유 (pool-1)** | 전용 |
| **Consumer** | 공유 (pool-1) | **전용** | 전용 |
| **K8s 네임스페이스** | pool-1 | 테넌트 전용 (Consumer만) | 테넌트 전용 |
| **SQS/DynamoDB** | 공유 | Consumer만 전용 | 전용 |
| **비용** | 낮음 | 중간 | 높음 |
| **격리 수준** | 낮음 | 중간 | 높음 |

### 7.3 Advanced 티어의 설계 의도

Premium의 비용은 부담스럽고 Basic은 격리가 부족한 고객을 위한 **Hybrid 모델**:

- **Producer**: 공유 (요청 패턴이 균일한 경우 비용 효율)
- **Consumer**: 전용 (데이터 처리 격리 필요)

→ 이는 **Noisy Neighbor 문제 완화**가 필요한 컴플라이언스 요구 고객에 적합.

### 7.4 검증 방법 (curl 응답으로 확인)

```bash
$ curl -H "tenantID: tenant-t1d6c" $APP_LB/producer | jq
{
  "environment": "pool-1",      # ⭐ Producer는 공유
  "microservice": "producer",
  "tenant_id": "tenant-t1d6c"
}

$ curl -H "tenantID: tenant-t1d6c" $APP_LB/consumer | jq
{
  "environment": "tenant-t1d6c", # ⭐ Consumer는 전용
  "microservice": "consumer",
  "tenant_id": "tenant-t1d6c"
}
```

### 7.5 핵심 패턴

> 💡 **모든 티어가 동일한 단일 Helm 차트를 사용**한다. 차이는 오로지 `values` 설정.
> 새 티어 추가 = 새 템플릿 파일 하나 작성.

---

## 8. 실습 3: Argo Workflows로 온보딩/오프보딩 자동화

### 8.1 자동화의 핵심 = SQS 메시지 1개

수동 프로세스(템플릿 복사 → 변수 치환 → Git 커밋 → 푸시)를 **이벤트 드리븐**으로 자동화:

```
SQS 메시지 1개 전송
       ↓
Argo Events (Sensor) 메시지 감지
       ↓
Argo Workflows 트리거
       ↓
Gitea 저장소 클론 → 티어 템플릿 복사 → 변수 치환 → 자동 커밋
       ↓
Flux v2 변경 감지 → EKS 배포
       ↓
Tofu Controller → AWS 리소스 생성
```

### 8.2 워크플로우 템플릿

```bash
$ kubectl get workflowtemplates -n argo-workflows
NAME                          AGE
tenant-deployment-template    2d
tenant-offboarding-template   2d
tenant-onboarding-template    2d
```

| 템플릿 | 역할 |
|---|---|
| `onboarding` | 새 테넌트 프로비저닝 |
| `offboarding` | 테넌트 제거 |
| `deployment` | HelmRelease 버전 업데이트 |

### 8.3 온보딩 SQS 메시지 예시

**Premium 티어:**
```bash
aws sqs send-message \
    --queue-url $ARGO_WORKFLOWS_ONBOARDING_QUEUE_SQS_URL \
    --message-body '{
        "tenant_id": "tenant-1",
        "tenant_tier": "premium",
        "release_version": "0.0"
    }'
```

**Advanced 티어:**
```bash
aws sqs send-message \
    --queue-url $ARGO_WORKFLOWS_ONBOARDING_QUEUE_SQS_URL \
    --message-body '{
        "tenant_id": "tenant-3",
        "tenant_tier": "advanced",
        "release_version": "0.0"
    }'
```

### 8.4 배포 결과 검증

```bash
# Premium: Producer + Consumer 전용 배포
$ kubectl -n tenant-1 get deployment
NAME                READY   AGE
tenant-1-consumer   3/3     5m
tenant-1-producer   3/3     5m

# Basic: 전용 Deployment 없음 (pool-1 공유)
$ kubectl -n tenant-2 get deployment
No resources found in tenant-2 namespace.

# Advanced: Consumer만 전용
$ kubectl -n tenant-3 get deployment
NAME                READY   AGE
tenant-3-consumer   3/3     4m

# Pool-1: Basic + Advanced 공유
$ kubectl -n pool-1 get deployment
NAME              READY   AGE
pool-1-consumer   3/3     18h
pool-1-producer   3/3     18h
```

### 8.5 E2E 데이터 흐름 검증

```bash
# Producer로 POST → DynamoDB까지 흘러가는지 확인
curl -X POST -H "tenantID: tenant-3" -H "tier: advanced" $APP_LB/producer

aws dynamodb scan --table-name $TABLE_NAME
```

응답:
```json
{
  "consumer_environment": "tenant-3",   # 전용
  "producer_environment": "pool-1",     # 공유
  "tenant_id": "tenant-3"
}
```

→ Advanced 티어의 Hybrid 모델이 데이터 레벨에서도 정확히 검증됨.

### 8.6 오프보딩

오프보딩 SQS 큐에 메시지를 보내면 끝:

```bash
aws sqs send-message \
    --queue-url $ARGO_WORKFLOWS_OFFBOARDING_QUEUE_SQS_URL \
    --message-body '{
        "tenant_id": "tenant-t1d6c",
        "tenant_tier": "advanced"
    }'
```

`destroyResourcesOnDeletion: true` 설정 덕분에 Terraform CRD 삭제 시 AWS 리소스(SQS, DynamoDB)도 자동 정리.

---

## 9. 참고: Argo 생태계 도구들

### 9.1 Argo CD Image Updater

이미지가 새로 푸시되면 GitOps 저장소를 자동 업데이트하는 도구.

![Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/assets/argo-cd-image-updater-overview.png)

> 위 이미지가 안 보이면 [공식 문서](https://argocd-image-updater.readthedocs.io/en/stable/) 참조.

**기존 ArgoCD 방식 vs Image Updater:**
- 기존: 이미지 변경 시 매니페스트 수동 수정 → git push
- Image Updater: 레지스트리 폴링 → 자동 매니페스트 업데이트

참고 글:
- [Mastering Argo CD Image Updater with Helm (CNCF)](https://www.cncf.io/blog/2024/11/05/mastering-argo-cd-image-updater-with-helm-a-complete-configuration-guide/)
- [우아한기술블로그 - 안정적 AI 서빙 시스템](https://techblog.woowahan.com/19548/)

### 9.2 Argo CD App-of-Apps 패턴

여러 애플리케이션을 **단일 부모 Application**으로 관리하는 패턴.

```
Root Application (App-of-Apps)
    ├── Child App: helm-guestbook
    ├── Child App: kustomize-guestbook
    └── Child App: sync-waves
```

**핵심:** Root Application 하나로 다수의 Child Application을 한 번에 부트스트랩.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps
  namespace: argocd
spec:
  source:
    path: apps
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
```

참고: [공식 가이드](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)

### 9.3 Argo Rollouts

Kubernetes의 기본 Deployment가 가진 한계(롤링 업데이트만 지원)를 보완하는 **Progressive Delivery 컨트롤러**.

| 기본 Deployment 한계 | Argo Rollouts 해결 |
|---|---|
| 출시 속도 제어 부족 | 단계별 가중치 제어 |
| 카나리 트래픽 라우팅 불가 | Ingress/Service Mesh 통합 |
| 외부 메트릭 검증 불가 | Prometheus, Datadog 등 통합 |
| 자동 롤백 불가 | 분석 기반 자동 롤백 |

**카나리 전략 예시:**
```yaml
strategy:
  canary:
    steps:
      - setWeight: 20
      - pause: {}              # 수동 승인 대기
      - setWeight: 40
      - pause: {duration: 10}
      - setWeight: 60
      - pause: {duration: 10}
      - setWeight: 80
      - pause: {duration: 10}
```

지원 기능:
- Blue-Green / Canary 업데이트
- 가중치 기반 트래픽 분배
- 자동 롤백/프로모션
- Ingress 통합: NGINX, ALB, Apache APISIX
- Service Mesh 통합: Istio, Linkerd, SMI
- 메트릭 제공자: Prometheus, Datadog, New Relic, CloudWatch 등

참고: [공식 문서](https://argoproj.github.io/argo-rollouts/)

### 9.4 GitOps Bridge

**Kubernetes 클러스터 생성부터 GitOps까지 연결**하는 커뮤니티 프로젝트.

```
Terraform (클러스터 생성) → GitOps Bridge → Argo CD/Flux (워크로드 관리)
```

핵심 디렉토리 구조:
- `bootstrap/control-plane`: 제어 평면 클러스터 부트스트랩
- `bootstrap/workloads`: 일반 클러스터 부트스트랩
- `environments/`: dev/qa/prod 환경별 helm values
- `clusters/`: 클러스터별 override
- `teams/`: 팀별 네임스페이스 온보딩

참고: [GitHub - gitops-bridge](https://github.com/gitops-bridge-dev/gitops-bridge)

---
