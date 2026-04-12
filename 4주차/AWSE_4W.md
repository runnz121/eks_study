# 🔐 AWS EKS 인증(AuthN) / 인가(AuthZ) 종합 정리

> **학습 범위**
> - K8S API 접근 통제 전체 파이프라인 (AuthN → AuthZ → Admission)
> - 운영자(kubectl) → EKS API 호출 흐름 (SigV4 토큰 생성 포함)
> - ConfigMap vs EKS Access Entry 인가 방식 비교
> - ServiceAccount, RBAC, Projected Volume
> - IRSA (IAM Roles for Service Accounts) 동작 원리
> - EKS Pod Identity 동작 원리
> - IRSA vs Pod Identity 전략적 비교

---

## 📌 1. 왜 EKS 인증/인가가 어려운가?

AWS와 K8S(EKS)는 **별도의 플랫폼**으로 각자의 인증/인가 처리 방식을 가지고 있다.
EKS는 AWS 관리형 제품이지만, 통합 시스템이 아닌 **별도의 K8S 플랫폼**으로 이해해야 동작을 파악할 수 있다.

이 두 시스템을 통합하기 위해 **Webhook, OIDC** 등의 외부 인증 연동 방법이 사용된다.

| 알아야 할 영역 | 핵심 개념 |
|---|---|
| AWS 인증/인가 | IAM, STS, AssumeRole, SigV4 |
| K8S 인증/인가 | ServiceAccount, RBAC, Webhook |
| 외부 인증 연동 | OIDC, Webhook Token Authentication |

---

## 📌 2. K8S API 접근 통제 전체 파이프라인

![K8S API Access Control Overview](https://kubernetes.io/docs/concepts/security/controlling-access/access-control-overview.svg)
> 출처: [kubernetes.io — Controlling Access to the API](https://kubernetes.io/docs/concepts/security/controlling-access/)

모든 K8S API 요청은 다음 4단계를 통과한다.

```
Client ──→ [1. Transport Security (TLS)]
        ──→ [2. Authentication (AuthN)]  ← 누구인가?
        ──→ [3. Authorization (AuthZ)]   ← 무엇을 할 수 있는가?
        ──→ [4. Admission Control]       ← 요청을 변환/검증할 것인가?
        ──→ ETCD (저장)
```

### K8S 지원 인증(AuthN) 방식

| 방식 | 설명 | EKS 사용 여부 |
|------|------|---------------|
| X.509 클라이언트 인증서 | kubeconfig의 client-certificate-data | 제한적 |
| Bootstrap Token | 노드 부트스트랩용 | ✅ |
| Service Account Token | Pod 내부 JWT 토큰 | ✅ |
| **Webhook Token Auth** | Bearer 토큰을 외부 검증 서버에 위임 | ✅ **핵심** |
| OIDC Token | 외부 IdP의 JWT 토큰 | 선택적 |

### K8S 지원 인가(AuthZ) 방식

| 방식 | 설명 | EKS 사용 여부 |
|------|------|---------------|
| AlwaysAllow | 모든 요청 허용 | ❌ |
| AlwaysDeny | 모든 요청 거부 | ❌ |
| ABAC | 속성 기반 접근 제어 | ❌ |
| **RBAC** | 역할 기반 접근 제어 | ✅ |
| **Node** | 노드 권한 제한 | ✅ |
| **Webhook** | 외부 서비스에 인가 위임 | ✅ **EKS API 방식** |

> EKS API 사용 시 인가 모드: `Node, RBAC, Webhook` (3개 모두 사용)

---

## 📌 3. 운영자(kubectl) → EKS API 호출 전체 흐름

### 전체 6단계

```
[1] Client side    : aws eks get-token → Pre-signed STS URL 생성
[2] kubectl        : Bearer Token + K8S Action → EKS API Endpoint 요청
[3] EKS API Server : Webhook Token Authenticator → TokenReview 요청
[4] AWS STS        : GetCallerIdentity 검증 → IAM 주체 정보 반환
[5] EKS AuthZ      : IAM 주체 → K8S Subject 매핑 (Access Entry / ConfigMap)
[6] K8S Action 실행 후 클라이언트에 최종 응답
```

---

## 📌 4. Step 1: EKS 인증 토큰 생성 (SigV4 심화)

### 4-1. 토큰이란?

`aws eks get-token` 명령이 생성하는 토큰은 **Base64 인코딩된 Pre-signed STS URL**이다.

```
k8s-aws-v1.aHR0cHM6Ly9zdH...   ← 이 형태
```

디코딩하면:
```
https://sts.ap-northeast-2.amazonaws.com/
  ?Action=GetCallerIdentity
  &Version=2011-06-15
  &X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKIA.../sts/aws4_request
  &X-Amz-Date=20260401T135015Z
  &X-Amz-Expires=60
  &X-Amz-SignedHeaders=host;x-k8s-aws-id
  &X-Amz-Signature=35c8dff...
```

### 4-2. SigV4 서명 4단계

AWS Signature Version 4(SigV4)는 Secret Access Key를 직접 노출하지 않고 서명을 생성하는 메커니즘이다.

**1단계: 정규 요청(Canonical Request) 생성**
```
HTTP 메서드: GET
엔드포인트: https://sts.ap-northeast-2.amazonaws.com
쿼리 파라미터: Action=GetCallerIdentity&...
헤더: host, x-k8s-aws-id (클러스터명)
→ 해시(SHA256) 처리 → Hashed Canonical Request
```

**2단계: 서명할 문자열(String to Sign) 생성**
```
알고리즘: AWS4-HMAC-SHA256
요청 시각: 20260401T135015Z
Credential Scope: 20260401/ap-northeast-2/sts/aws4_request
1단계 해시값 포함
```

**3단계: 서명 키(Signing Key) 유도 — HMAC-SHA256 4단계 체인**

```
HMAC("AWS4" + SecretKey, 날짜)       → DateKey
HMAC(DateKey, 리전)                   → RegionKey
HMAC(RegionKey, 서비스명 "sts")       → ServiceKey
HMAC(ServiceKey, "aws4_request")      → Signing Key (최종)
```

> **핵심**: 시크릿 키는 첫 번째 HMAC의 재료로만 사용. 네트워크에 절대 노출되지 않음.
> 최종 서명 키는 특정 날짜 + 리전 + 서비스에서만 유효한 **일회성** 키.

**4단계: Pre-signed URL 생성 및 토큰 변환**
```
Signing Key + String to Sign → Signature(서명값)
→ URL 쿼리 파라미터로 조합 (Pre-signed URL)
→ Base64 URL 인코딩
→ 접두사 "k8s-aws-v1." 추가
→ 최종 Bearer Token 완성
```

### 4-3. AWS의 Secret Key 보관 방식

AWS가 서명을 검증하려면 시크릿 키를 알아야 한다. 하지만 평문으로 저장하지 않는다.

| 보안 조치 | 내용 |
|---|---|
| **HSM(Hardware Security Module)** | 특수 암호화 하드웨어에 저장 |
| **검증 시에만 접근** | 서명 검증 순간에만 시스템 내부에서 사용 |
| **권한 분리** | AWS 내부 직원도 평문 키 조회 불가 |

```
클라이언트: Secret Key → HMAC → Signature 생성
AWS 서버:  (저장된 Secret Key 기반) → 동일한 HMAC 계산 → Signature 비교
→ 일치하면 인증 성공 (같은 입력 → 같은 출력의 수학적 원리)
```

### 4-4. 토큰 TTL 및 자동 갱신

- 토큰 유효시간: **15분** (`X-Amz-Expires=60`은 60초가 아니라 실제 만료는 최대 15분)
- kubectl의 **Client-Go 라이브러리**가 `expirationTimestamp`를 감지해 자동 재발급
- 사용자는 인지하지 못한 채 작업 지속 가능

### 🧪 실습 결과: 토큰 발급 및 Pre-signed URL 디코딩

```bash
$ aws eks get-token --cluster-name myeks
```
```json
{
    "kind": "ExecCredential",
    "apiVersion": "client.authentication.k8s.io/v1beta1",
    "status": {
        "expirationTimestamp": "2026-04-12T12:47:20Z",
        "token": "k8s-aws-v1.aHR0cHM6Ly9zdHMuYXAtbm9ydGhlYXN0LTIuYW1hem9u..."
    }
}
```

토큰 Payload를 Base64 디코딩하면 실제 Pre-signed URL이 나온다:

```
https://sts.ap-northeast-2.amazonaws.com/?Action=GetCallerIdentity
  &Version=2011-06-15
  &X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKIATPQNKFFDIIUEOJRH/20260412/ap-northeast-2/sts/aws4_request
  &X-Amz-Date=20260412T123321Z
  &X-Amz-Expires=60
  &X-Amz-SignedHeaders=host;x-k8s-aws-id
  &X-Amz-Signature=735c6d38f999fd28...
```

> `X-Amz-Credential`의 `AKIA`로 시작하는 값이 영구 Access Key ID임을 확인할 수 있다.

---

## 📌 5. Step 2: kubectl → EKS API 요청

```
kubectl get node
→ kubeconfig의 exec 플러그인 실행 (aws eks get-token)
→ Bearer Token 생성
→ HTTP 요청: GET https://{EKS_ENDPOINT}/api/v1/nodes?limit=500
   Authorization: Bearer k8s-aws-v1.{base64}
```

실제 curl 명령어로 직접 호출도 가능:
```bash
curl -k -XGET \
  -H "Authorization: Bearer $TOKEN_DATA" \
  -H "Accept: application/json" \
  "https://{EKS_ENDPOINT}/api/v1/nodes?limit=500"
```

### 🧪 실습 결과: kubectl verbose 로그 및 curl 직접 호출

```bash
$ kubectl get node -v=6
I0412 21:34:00 loader.go:373] Config loaded from file: /Users/pupu/.kube/config
I0412 21:34:00 round_trippers.go:553] GET https://60C66EEE20B0BF72F87B927718DF7E87.gr7.ap-northeast-2.eks.amazonaws.com/api/v1/nodes?limit=500 200 OK in 912 milliseconds
```

```bash
$ curl -k -s -XGET -H "Authorization: Bearer $TOKEN_DATA" \
    "${EKS_ENDPOINT}/api/v1/nodes?limit=500" | jq '.items[].metadata.name'
"ip-192-168-1-228.ap-northeast-2.compute.internal"
"ip-192-168-3-214.ap-northeast-2.compute.internal"
```

> kubectl 없이도 Bearer 토큰과 curl만으로 K8S API에 직접 접근 가능함을 확인. 실제 API Endpoint는 EKS 클러스터 ID가 포함된 `{ID}.gr7.ap-northeast-2.eks.amazonaws.com` 형태.

---

## 📌 6. Step 3 & 4: TokenReview + AWS STS 검증

### 6-1. TokenReview

EKS API Server는 **Webhook Token Authenticator**를 통해 TokenReview를 수행한다.

```yaml
apiVersion: authentication.k8s.io/v1
kind: TokenReview
spec:
  token: k8s-aws-v1.aHR0c...
```

내부적으로:
1. Bearer 토큰(Pre-signed URL) 추출
2. AWS STS `GetCallerIdentity` 호출 (토큰 자체가 STS URL이므로)
3. STS → IAM 주체 정보 반환

### 6-2. TokenReview 응답 예시

```json
{
  "status": {
    "authenticated": true,
    "user": {
      "username": "arn:aws:iam::911283464785:user/admin",
      "uid": "aws-iam-authenticator:911283464785:AIDA5ILF2FJI...",
      "groups": ["system:authenticated"],
      "extra": {
        "accessKeyId": ["AKIA5ILF2FJI..."],
        "arn": ["arn:aws:iam::911283464785:user/admin"],
        "principalId": ["AIDA5ILF2FJI..."],
        "sessionName": [""]
      }
    }
  }
}
```

> ⚠️ `system:masters`는 **TokenReview 응답에 표시되지 않음**.
> EKS 클러스터 생성 IAM 주체에게 내부적으로 자동 부여되며 표시되는 구성에 나타나지 않는다.

### 6-3. CloudTrail에서 GetCallerIdentity 확인

EKS Authenticator가 STS를 호출할 때마다 CloudTrail에 이벤트가 기록된다.

```json
{
  "eventSource": "sts.amazonaws.com",
  "eventName": "GetCallerIdentity",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDA5ILF2FJI...",
    "arn": "arn:aws:iam::911283464785:user/admin"
  },
  "sourceIPAddress": "3.37.104.78",
  "userAgent": "Go-http-client/1.1"
}
```

### 🧪 실습 결과: TokenReview 및 SubjectAccessReview

**TokenReview 결과** — root 계정으로 인증했을 때:

```bash
$ kubectl create -f /tmp/token-review.yaml -o json | jq '.status'
```
```json
{
  "authenticated": true,
  "username": "arn:aws:iam::239472355654:root",
  "groups": ["system:authenticated"]
}
```

> `system:masters`가 groups에 표시되지 않음. Access Entry의 `AmazonEKSClusterAdminPolicy`가 실제 권한 부여. TokenReview는 인증(AuthN)만 담당하고 인가(AuthZ)는 다음 단계에서 처리.

**SubjectAccessReview 결과** — system:masters 그룹의 kube-system pods get 권한:

```bash
$ kubectl create -f /tmp/sar-request.yaml -o json | jq '.status'
```
```json
{
  "allowed": true
}
```

---

## 📌 7. Step 5: IAM 주체 → K8S Subject 매핑 (인가)

### 7-1. 방안 비교

| 구분 | aws-auth ConfigMap (deprecated) | EKS API — Access Entry (권장) |
|------|--------------------------------|-------------------------------|
| 인증 방식 | TokenReview (동일) | TokenReview (동일) |
| 데이터 저장소 | K8S etcd (ConfigMap) | AWS EKS 관리형 DB |
| 관리 도구 | kubectl YAML 직접 편집 | AWS CLI/콘솔/IaC |
| 인가 방식 | K8S RBAC (CR/CRB 필요) | AWS Access Policy (관리형) 또는 K8S RBAC |
| 리소스 관리 | 수동, 휴먼 에러 위험 | AWS API 기반, 안전 |
| 복구 | 잘못 수정 시 복구 어려움 | AWS API로 복구 가능 |

> ⚠️ **정책 중복 시 EKS API 우선 적용, ConfigMap 무시됨!**

### 7-2. aws-auth ConfigMap 방식 (deprecated)

```yaml
data:
  mapRoles: |
    - rolearn: arn:aws:iam::911283464785:role/myeks-ng-1
      username: system:node:{{EC2PrivateDNSName}}
      groups:
      - system:bootstrappers
      - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::911283464785:user/admin
      username: kubernetes-admin
      groups:
      - system:masters
```

**ConfigMap 실수 사례**: `system:nodes` 그룹을 삭제하면 약 5분 후 **모든 노드 NotReady** 상태 발생!

### 7-3. EKS Access Entry 인가 흐름 (권장)

```
① IAM 주체 인증 완료
② Access Entry DB에서 해당 IAM ARN 조회
③ 매핑된 K8S Username / Groups 확인
④-A: Access Policy 연결된 경우
      → SubjectAccessReview → EKS Authorizer Webhook → allowed: true/false
④-B: K8S Groups만 있는 경우 (커스텀 RBAC)
      → EKS Webhook "No Opinion" → K8S RBAC 검사
```

**EKS 제공 Access Policy (주요)**

| 정책명 | 수준 | K8S 매핑 유사도 |
|--------|------|-----------------|
| `AmazonEKSClusterAdminPolicy` | 클러스터 전체 관리자 | cluster-admin |
| `AmazonEKSAdminPolicy` | 대부분 리소스 관리 | admin |
| `AmazonEKSEditPolicy` | 리소스 생성/수정/삭제 | edit |
| `AmazonEKSViewPolicy` | 읽기 전용 | view |
| `AmazonEKSAdminViewPolicy` | 읽기 전용 (ClusterRole 포함) | - |

### 7-4. system:masters 그룹의 특수성

```
system:masters 그룹에 속한 사용자:
→ RBAC 기반 권한 검사 거치지 않음
→ Webhook 기반 인가도 거치지 않음
→ API 서버에서 모든 동작 무조건 허용 (슈퍼유저)

주의: AWS Root 계정처럼 최소화해서 사용할 것
권장: cluster-admin ClusterRole + ClusterRoleBinding 방식 사용
```

### 🧪 실습 결과: Access Entry 조회 및 testuser 관리

**현재 클러스터 Access Entry 목록:**

```bash
$ aws eks list-access-entries --cluster-name myeks
```
```json
{
    "accessEntries": [
        "arn:aws:iam::239472355654:role/aws-service-role/eks.amazonaws.com/AWSServiceRoleForAmazonEKS",
        "arn:aws:iam::239472355654:role/myeks-node-group-eks-node-group-...",
        "arn:aws:iam::239472355654:root"
    ]
}
```

**root 계정의 Access Entry 상세 — SA 연결 없이 클러스터 admin:**

```json
{
    "clusterName": "myeks",
    "principalArn": "arn:aws:iam::239472355654:root",
    "kubernetesGroups": [],
    "type": "STANDARD",
    "username": "arn:aws:iam::239472355654:root"
}
```

**연결된 정책 — AmazonEKSClusterAdminPolicy (cluster 스코프):**

```json
{
    "policyArn": "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy",
    "accessScope": { "type": "cluster" }
}
```

**testuser CRUD 실습 흐름:**

```bash
# 1. IAM 유저 생성
$ aws iam create-user --user-name testuser
→ UserId: AIDATPQNKFFDAE433QYGV

# 2. Access Entry 생성 → Access Policy 연결
$ aws eks create-access-entry ... --principal-arn ".../testuser"
$ aws eks associate-access-policy ... --policy-arn ".../AmazonEKSAdminPolicy" --access-scope type=cluster

# 3. K8S 그룹으로 업데이트
$ aws eks update-access-entry ... --kubernetes-group pod-admin
→ kubernetesGroups: ["pod-admin"], modifiedAt: "2026-04-12T21:34:58..."

# 4. 삭제
$ aws eks delete-access-entry ... → 삭제 완료
```

**aws-auth ConfigMap 실제 내용:**

```yaml
data:
  mapRoles: |
    - rolearn: arn:aws:iam::239472355654:role/myeks-node-group-...
      username: system:node:{{EC2PrivateDNSName}}
      groups:
      - system:bootstrappers
      - system:nodes
```

> 노드그룹 IAM Role만 매핑됨. root/testuser는 ConfigMap이 아닌 Access Entry로 관리.

---

## 📌 8. K8S RBAC 핵심 개념

### 8-1. Pod SA 인증 → RBAC 인가 전체 구조

![Pod ServiceAccount Auth Flow](https://hongchangsub.com/content/images/2026/04/image-7.png)
> 출처: [hongchangsub.com — AWS EKS 인증/인가 실습](https://hongchangsub.com/aws-eks-auth/)

### 8-2. RBAC 4대 리소스

| 리소스 | 스코프 | 역할 |
|--------|--------|------|
| `Role` | Namespace | 특정 네임스페이스의 권한 정의 |
| `ClusterRole` | Cluster | 클러스터 전체 권한 정의 |
| `RoleBinding` | Namespace | Role ↔ Subject 연결 |
| `ClusterRoleBinding` | Cluster | ClusterRole ↔ Subject 연결 |

### 8-3. RBAC 네임스페이스 구조

![RBAC Namespace Diagram](https://hongchangsub.com/content/images/2026/04/image-11.png)
> 출처: [hongchangsub.com — AWS EKS 인증/인가 실습](https://hongchangsub.com/aws-eks-auth/)

### 8-4. Role 구성 요소

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-viewer-role
  namespace: dev-team
rules:
- apiGroups: [""]           # "" = core group (Pod, Service, ConfigMap 등)
  resources: ["pods"]       # 대상 리소스
  verbs: ["get", "list", "watch"]  # 허용 동작
```

### 8-5. Verbs → HTTP Method 매핑

| K8S Verb | HTTP Method | 설명 | rbac-tool 약어 |
|----------|-------------|------|----------------|
| `get` | GET (단건) | 단일 리소스 조회 | G |
| `list` | GET (목록) | 리소스 목록 조회 | L |
| `watch` | GET + watch param | 실시간 변경 감시 | W |
| `create` | POST | 생성 | C |
| `update` | PUT | 전체 수정 | U |
| `patch` | PATCH | 부분 수정 | P |
| `delete` | DELETE | 삭제 | D |
| `deletecollection` | DELETE (컬렉션) | 다수 삭제 | DC |

### 8-6. Subjects 유형

```yaml
subjects:
- kind: User           # kubectl 사용 IAM 유저
  name: "arn:aws:iam::..."
- kind: Group          # Access Entry의 kubernetes-groups
  name: "pod-viewer"
- kind: ServiceAccount # Pod에 할당된 SA
  name: dev-k8s
  namespace: dev-team
```

### 8-7. krew RBAC 분석 플러그인

| 명령어 | 용도 |
|--------|------|
| `kubectl rbac-tool whoami` | 현재 인증 주체의 K8S Subject 정보 확인 |
| `kubectl rbac-tool lookup {group}` | 특정 그룹/유저의 바인딩된 Role 조회 |
| `kubectl rolesum {sa} -n {ns}` | SA의 권한 요약 (G/L/W/C/U/P/D 표시) |
| `kubectl access-matrix` | 클러스터 전체 접근 매트릭스 |
| `kubectl auth can-i {verb} {resource}` | 특정 권한 보유 여부 확인 |

### 🧪 실습 결과: ServiceAccount + Role + RoleBinding 권한 검증

**RBAC 적용 전 — 아무 권한 없음:**

```bash
$ kubectl exec dev-kubectl -n dev-team -- kubectl get pods
Error from server (Forbidden): pods is forbidden:
  User "system:serviceaccount:dev-team:dev-k8s" cannot list resource "pods" ...

$ kubectl exec dev-kubectl -n dev-team -- kubectl auth can-i get pods
no
```

**Role(네임스페이스 내 `*` 권한) + RoleBinding 생성 후:**

```bash
$ kubectl exec dev-kubectl -n dev-team -- kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
dev-kubectl   1/1     Running   0          44s

$ kubectl exec dev-kubectl -n dev-team -- kubectl run nginx --image nginx:1.20-alpine
pod/nginx created

$ kubectl exec dev-kubectl -n dev-team -- kubectl delete pods nginx
pod "nginx" deleted

$ kubectl exec dev-kubectl -n dev-team -- kubectl auth can-i get pods
yes
```

**다른 네임스페이스 / 클러스터 스코프 — 차단됨:**

| 동작 | 결과 | 이유 |
|------|------|------|
| 자기 NS pods 조회/생성/삭제 | ✅ **OK** | Role이 해당 NS의 모든 권한 부여 |
| kube-system pods 조회 | ❌ **Forbidden** | Role은 namespace-scoped (dev-team만) |
| nodes 조회 (클러스터 스코프) | ❌ **Forbidden** | Role은 ClusterRole이 아님 |

> Role vs ClusterRole의 스코프 차이를 명확히 확인.

---

## 📌 9. ServiceAccount와 SA 토큰

### 9-1. ServiceAccount 개념

```
User Account (사람)         ServiceAccount (기계/파드)
─────────────────────      ──────────────────────────────
K8S API 객체로 존재 X       K8S API 객체로 존재 O
외부 인증 기반               K8S 내부 인증 기반 (JWT)
특정 NS에 속하지 않음        특정 Namespace에 소속
```

- 파드 생성 시 serviceAccountName 미지정 → `default` SA 자동 적용
- `automountServiceAccountToken: false` 설정 시 토큰 미주입 (보안 강화)

### 9-2. Projected Volume (K8S 1.22+)

파드에 자동으로 마운트되는 프로젝티드 볼륨 구성:

```yaml
volumes:
- name: kube-api-access-{random}
  projected:
    sources:
    - serviceAccountToken:
        expirationSeconds: 3607   # 약 1시간
        path: token               # JWT 토큰
    - configMap:
        name: kube-root-ca.crt    # CA 번들
    - downwardAPI:
        items:
        - path: namespace         # 현재 네임스페이스
```

> K8S 1.22부터 Secret 기반 볼륨 대신 **Projected Volume** 사용. 만료 시간, Audience 지정 가능.

### 9-3. SA 토큰 JWT 구조

```json
// Header
{"alg": "RS256", "kid": "...", "typ": "JWT"}

// Payload (기본 SA 토큰)
{
  "aud": ["https://kubernetes.default.svc"],   ← K8S API 전용
  "exp": 1716619848,
  "iss": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/XXXX",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {"name": "my-pod", "uid": "..."},
    "serviceaccount": {"name": "my-sa", "uid": "..."}
  },
  "sub": "system:serviceaccount:default:my-sa"
}
```

**기본 SA 토큰 디코드 결과 (aud: kubernetes.default.svc):**

![JWT Token Decode — kubernetes.default.svc](https://hongchangsub.com/content/images/2026/04/image-8.png)
> 출처: [hongchangsub.com — AWS EKS 인증/인가 실습](https://hongchangsub.com/aws-eks-auth/)

### 🧪 실습 결과: SA 생성 및 기본 토큰 JWT 디코딩

```bash
$ kubectl create namespace dev-team && kubectl create sa dev-k8s -n dev-team

$ kubectl get sa dev-k8s -n dev-team -o yaml
# uid: bdf2ce42-7665-497a-b9bb-94d9e55ecd93  ← SA 자체에는 토큰 없음

# Pod 생성 후 마운트된 토큰 디코딩
```

```json
{
    "aud": ["https://kubernetes.default.svc"],
    "exp": 1776083803,
    "iat": 1775997403,
    "iss": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/60C66EEE20B0BF72F87B927718DF7E87",
    "kubernetes.io": {
        "namespace": "default",
        "node": { "name": "ip-192-168-1-228.ap-northeast-2.compute.internal" },
        "pod": { "name": "eks-iam-test2" },
        "serviceaccount": { "name": "default" }
    },
    "sub": "system:serviceaccount:default:default"
}
```

> `iss` 필드에서 실제 EKS OIDC Provider ID(`60C66EEE...`)를 확인. 이 URL이 IRSA에서 IAM이 공개키를 가져오는 주소.

---

## 📌 10. Admission Control 단계

![Admission Control Phases](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/admission-control-phases.svg)
> 출처: [kubernetes.io — Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

```
AuthN → AuthZ → [MutatingWebhook] → Object Schema Validation → [ValidatingWebhook] → etcd
```

IRSA/Pod Identity에서 핵심:
- **MutatingWebhook** (`pod-identity-webhook`): Pod 생성 시 ENV 변수 + Volume을 자동 주입

```bash
# MutatingWebhook 확인
kubectl get MutatingWebhookConfiguration
# pod-identity-webhook ← IRSA/Pod Identity 주입 담당
# aws-load-balancer-webhook
# cert-manager-webhook
```

### 🧪 실습 결과: MutatingWebhookConfiguration 확인

```bash
$ kubectl get MutatingWebhookConfiguration
NAME                            WEBHOOKS   AGE
pod-identity-webhook            1          11m
vpc-resource-mutating-webhook   1          11m
```

> Pod Identity Agent 설치 후 `pod-identity-webhook`이 등록됨. 이 webhook이 SA에 Pod Identity Association이 있는 Pod 생성 시 ENV + Volume을 자동 주입.

---

## 📌 11. Node IAM Role vs Pod 단위 권한

### 11-1. Node IAM Role(EC2 Instance Profile)의 한계

```
[Before IRSA/Pod Identity]
Node IAM Role → 노드 위의 모든 Pod가 동일한 권한 사용
→ 최소 권한 원칙 위반
→ Pod A가 Pod B의 권한까지 접근 가능 (보안 취약)
```

### 11-2. Pod 단위 권한 공통 요구사항

- 컨테이너 내부에 민감 키 정보 없음 (이미지 유출 시에도 안전)
- 개발팀에서 별도 리소스 없이 런타임에서 자격증명 자동 주입
- Pod 최소 단위 권한 적용

---

## 📌 12. IRSA (IAM Roles for Service Accounts)

> OIDC 소셜 로그인 비유: K8S(EKS) = 구글(신원 증명), AWS = 서비스 앱(권한 부여)

### 12-1. IRSA 동작 전체 흐름

```
[파드 생성 시]
1. Pod spec에 IRSA SA 지정 (eks.amazonaws.com/role-arn 어노테이션)
2. pod-identity-webhook (MutatingWebhook)이 Pod Spec 수정:
   - ENV 주입: AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE
   - Volume 추가: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
   - 토큰 audience: sts.amazonaws.com (2번째 토큰)

[런타임 시]
3. App → AWS SDK 호출
4. SDK가 ENV 변수 자동 감지 → AssumeRoleWithWebIdentity 요청
   - 포함: IRSA JWT 토큰 + IAM Role ARN
5. AWS STS → IAM에 토큰 검증 요청
   - iss: OIDC Provider 일치?
   - sub: IAM Trust Policy의 sub 조건 일치?
   - aud: sts.amazonaws.com 일치?
   - exp: 만료 안됨?
6. IAM → EKS OIDC Provider의 JWKS URL에서 공개키 가져와 서명 검증
7. 검증 성공 → STS가 임시 자격증명 반환 (AssumedRole + SessionToken)
8. App → 임시 자격증명으로 AWS 서비스 사용
```

### 12-2. SA 토큰 2종류 비교

| 구분 | 1번째 토큰 (K8S API용) | 2번째 토큰 (IRSA용) |
|------|----------------------|---------------------|
| **audience (`aud`)** | `https://kubernetes.default.svc` | `sts.amazonaws.com` |
| **발급 주체 (`iss`)** | EKS OIDC Provider | EKS OIDC Provider |
| **용도** | K8S API 서버 인증 | AWS STS AssumeRole |
| **만료 시간** | ~1시간 (자동 갱신) | 86400초 (24시간) |
| **마운트 경로** | `/var/run/secrets/kubernetes.io/serviceaccount/` | `/var/run/secrets/eks.amazonaws.com/serviceaccount/` |
| **생성 주체** | Kubelet (TokenRequest API) | Pod Identity Webhook |
| **ENV 연동** | 없음 | `AWS_WEB_IDENTITY_TOKEN_FILE` |

**IRSA 토큰 디코드 결과 (aud: sts.amazonaws.com):**

![JWT Token Decode — sts.amazonaws.com](https://hongchangsub.com/content/images/2026/04/image-9.png)
> 출처: [hongchangsub.com — AWS EKS 인증/인가 실습](https://hongchangsub.com/aws-eks-auth/)

### 12-3. IRSA용 MutatingWebhook 주입 내용

```yaml
# Pod Spec에 자동으로 추가되는 내용
env:
- name: AWS_STS_REGIONAL_ENDPOINTS
  value: regional
- name: AWS_DEFAULT_REGION
  value: ap-northeast-2
- name: AWS_ROLE_ARN
  value: arn:aws:iam::911283464785:role/my-irsa-role
- name: AWS_WEB_IDENTITY_TOKEN_FILE
  value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token

volumeMounts:
- mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
  name: aws-iam-token
  readOnly: true

volumes:
- name: aws-iam-token
  projected:
    sources:
    - serviceAccountToken:
        audience: sts.amazonaws.com     ← IRSA 전용 토큰
        expirationSeconds: 86400
        path: token
```

### 12-4. OIDC Provider 구성

```
EKS OIDC Discovery Endpoint:
https://oidc.eks.ap-northeast-2.amazonaws.com/id/{OIDC_ID}/.well-known/openid-configuration
→ jwks_uri: .../{OIDC_ID}/keys  ← 공개키 URL

JWKS (JSON Web Key Set) 예시:
{
  "keys": [{
    "kty": "RSA",
    "kid": "1559d9b7d...",
    "use": "sig",
    "alg": "RS256",
    "n": "7FP6H4lhM...",  ← RSA 공개키 (IAM이 토큰 서명 검증에 사용)
    "e": "AQAB"
  }]
}
```

### 12-5. IAM Role Trust Policy (IRSA)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/XXXX"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.ap-northeast-2.amazonaws.com/id/XXXX:sub":
          "system:serviceaccount:default:my-sa",   ← SA 제한
        "oidc.eks.ap-northeast-2.amazonaws.com/id/XXXX:aud":
          "sts.amazonaws.com"                       ← aud 제한
      }
    }
  }]
}
```

### 12-6. CloudTrail — AssumeRoleWithWebIdentity

```json
{
  "eventSource": "sts.amazonaws.com",
  "eventName": "AssumeRoleWithWebIdentity",
  "userIdentity": {
    "type": "WebIdentityUser",
    "userName": "system:serviceaccount:default:my-sa",
    "identityProvider": "arn:aws:iam::...oidc-provider/oidc.eks...."
  },
  "responseElements": {
    "credentials": {
      "accessKeyId": "ASIA...",  ← 임시 키 (ASIA 접두사)
      "expiration": "..."
    },
    "audience": "sts.amazonaws.com"
  }
}
```

### 12-7. IRSA 설정 시 AWS SDK 요구사항

IRSA 사용을 위한 최소 SDK 버전:

| SDK | 최소 버전 |
|-----|-----------|
| Java (v2) | 2.10.11 |
| Java (v1) | 1.12.782 |
| Python (Boto3) | 1.9.220 |
| AWS CLI | 1.16.232 |
| Go v1 | 1.23.13 |
| Node.js | 2.525.0 (v3: 3.27.0) |
| Ruby | 3.58.0 |

> **코딩 시 주의**: 별도 인증 객체 생성 말고 **Default Credential Provider** 사용 → SDK가 IRSA 토큰 자동 감지

```python
# Python (Boto3) 예시 - 별도 키 입력 없이 클라이언트 생성
import boto3
s3 = boto3.client('s3')  # SDK가 환경변수의 WebIdentityToken을 자동으로 읽음
```

### 🧪 실습 결과: IRSA 전체 흐름

**OIDC Provider 확인:**

```bash
$ aws eks describe-cluster --name myeks \
    --query "cluster.identity.oidc.issuer" --output text
→ OIDC ID: 60C66EEE20B0BF72F87B927718DF7E87
```

```json
{
    "issuer": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/60C66EEE20B0BF72F87B927718DF7E87",
    "jwks_uri": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/60C66EEE20B0BF72F87B927718DF7E87/keys",
    "id_token_signing_alg_values_supported": ["RS256"]
}
```

**eksctl로 IRSA SA 생성 → SA 어노테이션 확인:**

```bash
$ eksctl create iamserviceaccount --name my-sa --namespace default \
    --cluster myeks --approve \
    --role-name eksctl-myeks-pod-irsa-s3-readonly-role \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

$ kubectl describe sa my-sa
Annotations: eks.amazonaws.com/role-arn: arn:aws:iam::239472355654:role/eksctl-myeks-pod-irsa-s3-readonly-role
```

**MutatingWebhook 주입 ENV — 실제 Pod 확인:**

```bash
$ kubectl describe pod eks-iam-test3 | grep -A5 "Environment:"
    Environment:
      AWS_STS_REGIONAL_ENDPOINTS:   regional
      AWS_DEFAULT_REGION:           ap-northeast-2
      AWS_REGION:                   ap-northeast-2
      AWS_ROLE_ARN:                 arn:aws:iam::239472355654:role/eksctl-myeks-pod-irsa-s3-readonly-role
      AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

**IRSA 전용 JWT 토큰 디코딩 (aud: sts.amazonaws.com):**

```json
{
    "aud": ["sts.amazonaws.com"],
    "iss": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/60C66EEE20B0BF72F87B927718DF7E87",
    "sub": "system:serviceaccount:default:my-sa",
    "kubernetes.io": {
        "namespace": "default",
        "pod": { "name": "eks-iam-test3" },
        "serviceaccount": { "name": "my-sa" }
    }
}
```

**AWS 서비스 권한 테스트:**

```bash
$ kubectl exec eks-iam-test3 -- aws sts get-caller-identity --query Arn
"arn:aws:sts::239472355654:assumed-role/eksctl-myeks-pod-irsa-s3-readonly-role/botocore-session-..."

$ kubectl exec eks-iam-test3 -- aws s3 ls
2024-11-09 beadsstrings3
2024-11-09 codedeploygithubaction
2024-11-09 elasticbeanstalk-ap-northeast-2-239472355654   ✅ 성공

$ kubectl exec eks-iam-test3 -- aws ec2 describe-instances
[ERROR] UnauthorizedOperation: not authorized to perform: ec2:DescribeInstances  ❌ 실패 (정책 없음)
```

> S3ReadOnly 정책만 연결했으므로 S3 접근은 성공, EC2 접근은 실패. 최소 권한 원칙이 정상 동작함을 확인.

---

## 📌 13. EKS Pod Identity (신규 방식, 2023.11~)

### 13-1. IRSA와의 핵심 차이

IRSA가 OIDC 기반의 복잡한 설정이 필요했던 것과 달리, Pod Identity는 **EKS 서비스 자체에서 인가를 처리**한다.

```
[IRSA 경로]
Pod → IRSA 토큰 → AWS STS (AssumeRoleWithWebIdentity) → IAM (OIDC 검증) → 임시 자격증명

[Pod Identity 경로]
Pod → 링크로컬 요청 → Pod Identity Agent (DaemonSet) → EKS Auth API (AssumeRoleForPodIdentity) → 임시 자격증명
```

### 13-2. Pod Identity Agent

```bash
# DaemonSet으로 모든 워커 노드에 배포
kubectl get ds -n kube-system eks-pod-identity-agent

# 노드에서 링크로컬 주소로 리슨 (hostNetwork 사용)
sudo ss -tnlp | grep eks-pod-identit
# LISTEN 0 169.254.170.23:80   ← IPv4 링크로컬 주소
# LISTEN 0 [fd00:ec2::23]:80   ← IPv6 링크로컬 주소

# 노드에 가상 NIC 생성 확인
sudo ip addr
# pod-id-link0: 169.254.170.23/32
```

> 에이전트가 hostNetwork로 링크로컬 주소를 점유하므로 **Fargate 사용 불가**, 포트 80 충돌 주의

### 13-3. Pod Identity 동작 흐름

```
[파드 생성 시]
1. SA에 Pod Identity Association 설정
2. MutatingWebhook이 Pod Spec 수정:
   - ENV: AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
   - ENV: AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=.../eks-pod-identity-token
   - Volume: audience=pods.eks.amazonaws.com (3번째 토큰)

[런타임 시]
3. App → AWS SDK 호출 (Container Credentials Provider 경로 감지)
4. SDK → http://169.254.170.23/v1/credentials 요청 (토큰 포함)
5. Pod Identity Agent 수신
6. Agent → EKS Auth API AssumeRoleForPodIdentity 호출 (SigV4 서명)
7. EKS Auth API → JWT 검증 + Association 조회
8. 임시 자격증명 반환 (세션 태그 포함)
9. App → 임시 자격증명으로 AWS 서비스 사용
```

### 13-4. Pod Identity 토큰 (3번째 토큰)

```json
{
  "aud": ["pods.eks.amazonaws.com"],   ← Pod Identity 전용
  "iss": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/XXXX",
  "sub": "system:serviceaccount:default:s3-sa",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {"name": "eks-pod-identity", "uid": "..."},
    "serviceaccount": {"name": "s3-sa", "uid": "..."}
  }
}
```

### 13-5. IAM Role Trust Policy (Pod Identity)

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "pods.eks.amazonaws.com"   ← OIDC URL 불필요!
  },
  "Action": [
    "sts:AssumeRole",
    "sts:TagSession"   ← ABAC 세션 태그 지원
  ]
}
```

### 13-6. Pod Identity의 세션 태그 (ABAC 지원)

AssumeRoleForPodIdentity 응답에 자동으포함되는 세션 태그:

| 태그 키 | 예시 값 |
|---------|---------|
| `kubernetes-namespace` | `default` |
| `kubernetes-service-account` | `s3-sa` |
| `eks-cluster-arn` | `arn:aws:eks:...` |
| `eks-cluster-name` | `myeks` |
| `kubernetes-pod-name` | `eks-pod-identity` |
| `kubernetes-pod-uid` | `c15f3022-...` |

이 태그를 이용해 **하나의 IAM Policy로 여러 클러스터/네임스페이스를 구분**하는 ABAC 설계 가능.

### 13-7. CloudTrail — AssumeRoleForPodIdentity

```json
{
  "eventSource": "eks-auth.amazonaws.com",
  "eventName": "AssumeRoleForPodIdentity",
  "userIdentity": {
    "type": "AssumedRole",
    "arn": "arn:aws:sts::...:assumed-role/myeks-ng-1/i-0b5a..."  ← 노드 Role
  }
}
```

> 이벤트 소스가 `eks-auth.amazonaws.com`인 점이 IRSA(`sts.amazonaws.com`)와 다름.

### 🧪 실습 결과: Pod Identity 전체 흐름

**Pod Identity Agent 설치 (EKS Addon):**

```bash
$ aws eks create-addon --cluster-name myeks --addon-name eks-pod-identity-agent
→ addon version: v1.3.10-eksbuild.3, status: ACTIVE

$ kubectl get ds -n kube-system eks-pod-identity-agent
NAME                     DESIRED   CURRENT   READY
eks-pod-identity-agent   2         2         2       ← 노드 2개에 각 1개씩
```

**IAM Role Trust Policy — OIDC URL 없이 간단한 구조:**

```json
{
    "Statement": [{
        "Effect": "Allow",
        "Principal": { "Service": "pods.eks.amazonaws.com" },
        "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
}
```

**MutatingWebhook 주입 ENV — IRSA와 완전히 다른 방식:**

```bash
$ kubectl exec eks-pod-identity -- env | grep AWS
AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=.../eks-pod-identity-token
AWS_STS_REGIONAL_ENDPOINTS=regional
```

**Pod Identity 전용 JWT 토큰 디코딩 (aud: pods.eks.amazonaws.com):**

```json
{
    "aud": ["pods.eks.amazonaws.com"],
    "iss": "https://oidc.eks.ap-northeast-2.amazonaws.com/id/60C66EEE20B0BF72F87B927718DF7E87",
    "sub": "system:serviceaccount:default:s3-sa",
    "kubernetes.io": {
        "namespace": "default",
        "pod": { "name": "eks-pod-identity" },
        "serviceaccount": { "name": "s3-sa" }
    }
}
```

**토큰 3종류 audience 비교 정리:**

| 토큰 종류 | `aud` | 용도 |
|---------|-------|------|
| 기본 SA 토큰 | `https://kubernetes.default.svc` | K8S API 서버 인증 |
| IRSA 토큰 | `sts.amazonaws.com` | AWS STS AssumeRoleWithWebIdentity |
| Pod Identity 토큰 | `pods.eks.amazonaws.com` | EKS Pod Identity Agent |

**링크로컬 주소에서 직접 자격증명 획득:**

```bash
$ kubectl exec eks-pod-identity -- bash -c \
    'TOKEN=$(cat $AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE); \
     curl -s 169.254.170.23/v1/credentials -H "Authorization: $TOKEN"'
```
```json
{
    "AccessKeyId": "ASIATPQNKFFDKIXY3LBZ",
    "SecretAccessKey": "Ts+ayt3+...",
    "Token": "IQoJb3JpZ2...",
    "AccountId": "239472355654",
    "Expiration": "2026-04-12T18:40:40Z"
}
```

> `AccessKeyId`가 `ASIA`로 시작 → STS가 발급한 임시 자격증명임을 확인.

**AWS 서비스 접근 — IRSA와 동일한 결과:**

```bash
$ kubectl exec eks-pod-identity -- aws sts get-caller-identity --query Arn
"arn:aws:sts::239472355654:assumed-role/s3-eks-pod-identity-role/eks-myeks-eks-pod-id-1cc7caab-..."

$ kubectl exec eks-pod-identity -- aws s3 ls
2024-11-09 beadsstrings3
2024-11-09 codedeploygithubaction
2024-11-09 elasticbeanstalk-ap-northeast-2-239472355654   ✅ 성공
```

---

## 📌 14. IRSA vs EKS Pod Identity 전략적 비교

| 항목 | IRSA | EKS Pod Identity |
|------|------|-----------------|
| **OIDC Provider** | 필요 (클러스터당 1개) | **불필요** |
| **설정 복잡도** | 높음 (IAM 관리자 권한 필요) | **낮음** |
| **Trust Policy** | OIDC URL + sub/aud 조건 | `pods.eks.amazonaws.com` |
| **Role 재사용** | 클러스터마다 Trust Policy 수정 | **여러 클러스터 동일 Role** |
| **OIDC 한도 영향** | 계정당 100개 제한 | 제한 없음 |
| **세션 태그 (ABAC)** | 미지원 | **지원** |
| **지원 환경** | EKS, EKS Anywhere, 자체관리 K8S, Fargate | EKS 전용 (Fargate 미지원) |
| **SDK 요구사항** | 구 버전도 지원 | 최신 SDK 필요 |
| **CloudTrail 이벤트** | `AssumeRoleWithWebIdentity` | `AssumeRoleForPodIdentity` |
| **자격증명 경로** | STS 직접 (pre-signed URL) | 노드 에이전트 경유 |
| **토큰 audience** | `sts.amazonaws.com` | `pods.eks.amazonaws.com` |
| **EKS 추가 구성요소** | pod-identity-webhook (기본 내장) | Pod Identity Agent DaemonSet (별도 설치) |

### 언제 IRSA를 쓸까?

- EKS Anywhere, Fargate, 자체관리 K8S 환경
- 이미 IRSA를 사용 중이고 잘 동작하는 경우
- 특정 EKS Add-on이 Pod Identity 미지원인 경우 (vpc-cni 등)

### 언제 Pod Identity를 쓸까?

- 신규 EKS 클러스터 (K8S 1.24+)
- 멀티 클러스터 환경 (Role 재사용)
- IAM 관리자 권한 없이 설정하고 싶은 경우
- ABAC 기반 세밀한 접근 제어가 필요한 경우

---

## 📌 15. OIDC / OAuth2.0 배경 지식

### 15-1. OAuth 2.0 핵심 역할

| 역할 | IRSA 비유 |
|------|-----------|
| Resource Owner | AWS 계정 소유자 |
| Client | EKS Pod (애플리케이션) |
| Authorization Server | EKS OIDC Provider (K8S) |
| Resource Server | AWS 서비스 (S3, DynamoDB 등) |

### 15-2. OIDC 토큰 주요 클레임

| 클레임 | 설명 | IRSA 토큰 예시 |
|--------|------|----------------|
| `iss` | 토큰 발급자 | `https://oidc.eks.ap-northeast-2.amazonaws.com/id/XXXX` |
| `sub` | 주체 식별자 | `system:serviceaccount:default:my-sa` |
| `aud` | 대상 서비스 | `sts.amazonaws.com` (IRSA), `pods.eks.amazonaws.com` (Pod Identity) |
| `exp` | 만료 시각 | Unix timestamp |
| `iat` | 발급 시각 | Unix timestamp |

---

## 📌 16. AWS ID 접두사 정리

| 접두사 | 대상 리소스 | 특징 |
|--------|------------|------|
| `AKIA` | IAM Access Key (영구) | 삭제 전까지 유효 |
| `ASIA` | Temporary Access Key | STS AssumeRole 임시 키 |
| `AIDA` | IAM User ID | 사용자 고유 ID (API 결과에 포함) |
| `AROA` | IAM Role ID | Role 고유 ID |
| `ANPA` | IAM Managed Policy ID | 정책 고유 ID |

---

## 📌 17. 전체 아키텍처 요약

```
                    ┌──────────────────────────────────────┐
                    │           AWS EKS Cluster            │
  [운영자]          │  ┌──────────────────────────────────┐│
  kubectl ─────────►│  │ API Server (AuthN→AuthZ→Admission)││
  Bearer Token      │  │  ↓ Webhook → AWS STS 검증        ││
                    │  │  ↓ Access Entry / ConfigMap 인가  ││
                    │  └──────────────────────────────────┘│
                    │         Worker Node                   │
                    │  ┌──────────────────────────────────┐│
  [App Pod]         │  │ Pod (SA Token × 2~3종류 마운트)  ││
  AWS SDK ──────────►  │                                   ││
  자동 감지         │  │  [IRSA] → STS (AssumeRoleWithWeb) ││
                    │  │  [PodId] → Agent → EKS Auth API  ││
                    │  └──────────────────────────────────┘│
                    └──────────────────────────────────────┘
                              ↕
                    ┌──────────────────────────────────────┐
                    │                AWS                   │
                    │  IAM │ STS │ OIDC Provider │ EKS Auth│
                    └──────────────────────────────────────┘
```

---

## 📚 참고 링크

| 분류 | 링크 |
|------|------|
| K8S 공식 | [K8S API Access Control](https://kubernetes.io/docs/concepts/security/controlling-access/) |
| K8S 공식 | [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) |
| K8S 공식 | [Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/) |
| AWS 공식 | [EKS Access Entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html) |
| AWS 공식 | [IRSA Docs](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) |
| AWS 공식 | [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) |
| AWS Blog | [Diving into IRSA](https://aws.amazon.com/blogs/containers/diving-into-iam-roles-for-service-accounts/) |
| AWS Blog | [EKS Pod Identity 블로그](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/) |
| AWS Blog | [EKS Access Management Deep Dive](https://aws.amazon.com/blogs/containers/a-deep-dive-into-simplified-amazon-eks-access-management-controls/) |
| AWS Best Practices | [EKS Security](https://docs.aws.amazon.com/eks/latest/best-practices/security.html) |
| Workshop | [EKS Security Immersion Day](https://catalog.workshops.aws/eks-security-immersionday/en-US) |
| Youtube | [AWS IRSA Deep Dive (안지완)](https://www.youtube.com/watch?v=CJyzEGR3HJM) |
| 한국어 블로그 | [홍창섭 — AWS EKS 인증/인가 실습](https://hongchangsub.com/aws-eks-auth/) |
| CTF 실습 | [Big IAM Challenge](https://thebigiamchallenge.com/challenge/1) / [K8s LAN Party](https://k8slanparty.com/) / [EKS Cluster Games](https://eksclustergames.com/) |
