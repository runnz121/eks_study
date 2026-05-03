# 7주차 — EKS 클러스터 업그레이드 (In-place / Blue-Green)

> **목표**: AWS EKS 워크숍 환경에서 EKS 1.30 → 1.31 업그레이드를 **컨트롤 플레인 → 애드온 → 데이터 플레인** 순서로 진행하며, 노드 유형(관리형 / 카펜터 / 셀프 / Fargate)별 업그레이드 전략을 직접 실습한다.
>
> 실습 환경: AWS EKS Workshop (us-west-2), Terraform으로 프로비저닝된 EKS 1.30 클러스터, ArgoCD GitOps 기반 샘플 애플리케이션(retail-store-sample).

---

## 📑 목차

1. [실습 환경 개요](#1-실습-환경-개요)
2. [샘플 애플리케이션 (ArgoCD GitOps)](#2-샘플-애플리케이션-argocd-gitops)
3. [업그레이드 사전 준비](#3-업그레이드-사전-준비)
4. [In-place 클러스터 업그레이드 (1.30 → 1.31)](#4-in-place-클러스터-업그레이드-130--131)
   - [4-1. Control Plane 업그레이드](#4-1-control-plane-업그레이드)
   - [4-2. Add-on 업그레이드](#4-2-add-on-업그레이드)
   - [4-3. 관리형 노드그룹 — In-place](#4-3-관리형-노드그룹--in-place)
   - [4-4. 관리형 노드그룹 — Blue/Green](#4-4-관리형-노드그룹--bluegreen)
   - [4-5. Karpenter 노드 업그레이드](#4-5-karpenter-노드-업그레이드)
   - [4-6. Self-managed 노드 업그레이드](#4-6-self-managed-노드-업그레이드)
   - [4-7. Fargate 노드 업그레이드](#4-7-fargate-노드-업그레이드)
5. [핵심 정리 & 도전 과제](#5-핵심-정리--도전-과제)

---

## 1. 실습 환경 개요

### 1-1. 환경 구성

워크숍 환경은 **CloudFormation으로 IDE-Server(EC2)를 프로비저닝**하고, **Terraform으로 EKS 클러스터를 배포**한 구조로 되어 있다. EKS 1.30, AL2023(커널 6.1), containerd 2.2.1.

![EKS Architecture](https://docs.aws.amazon.com/images/eks/latest/userguide/images/eks-architecture.png)
*출처: [AWS EKS Architecture Docs](https://docs.aws.amazon.com/eks/latest/userguide/eks-architecture.html)*

### 1-2. 클러스터 구성 요약

| 구분 | 종류 | 비고 |
|---|---|---|
| 노드그룹 | **Managed (initial)** | m5/m6a/m6i.large, min 2 / max 10, 기본 AMI |
| 노드그룹 | **Managed (blue-mng)** | AZ-a 단일 배치, `dedicated=OrdersApp:NoSchedule` taint, stateful 워크로드용 |
| 노드그룹 | **Self-managed (default-selfmng)** | 사용자 지정 AMI, AL2023 |
| 노드그룹 | **Fargate (fp-profile)** | namespace=`assets` |
| 노드풀 | **Karpenter (default)** | spot 우선, c5/m5/m6i/m6a/r4/c4, `dedicated=CheckoutApp:NoSchedule` |
| 애드온 | coredns, kube-proxy, vpc-cni, ebs-csi-driver | EKS Managed Add-on |
| GitOps | **ArgoCD** | CodeCommit `eks-gitops-repo` 사용 |
| Storage | EBS (gp3 default), EFS | gp2는 default class 해제 |

### 1-3. 노드 종류별 분포 확인

```bash
# capacity type별 분류
kubectl get node --label-columns=eks.amazonaws.com/capacityType,node.kubernetes.io/lifecycle,karpenter.sh/capacity-type,eks.amazonaws.com/compute-type

# 노드그룹/노드풀별 분류
kubectl get node -L eks.amazonaws.com/nodegroup,karpenter.sh/nodepool

# 노드별 taint 확인
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints[*].key,VALUES:.spec.taints[*].value,EFFECTS:.spec.taints[*].effect'
```

### 1-4. 인증 모드 (API_AND_CONFIG_MAP)

EKS의 인증 모드가 `API_AND_CONFIG_MAP`이라 **Access Entries(API)와 aws-auth ConfigMap이 모두 활성화**되어 있다.

```bash
# API mode
aws eks list-access-entries --cluster-name $CLUSTER_NAME

# ConfigMap mode
eksctl get iamidentitymapping --cluster $CLUSTER_NAME
kubectl describe cm -n kube-system aws-auth
```

**IRSA를 사용하는 컨트롤러 4종**: `karpenter`, `aws-load-balancer-controller`, `ebs-csi-driver`, `aws-efs-csi-driver`

```bash
kubectl describe sa -A | grep role-arn
```

### 1-5. 편의 도구

`kube-ops-view`, `krew`, `eks-node-viewer`, `k9s` 설치는 이전 EKS 스터디 정리 문서 참고.

```bash
# kube-ops-view 설치 + NLB 노출
helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
helm install kube-ops-view geek-cookbook/kube-ops-view --version 1.2.2 -n kube-system

# eks-node-viewer
wget -O eks-node-viewer https://github.com/awslabs/eks-node-viewer/releases/download/v0.7.1/eks-node-viewer_Linux_x86_64
chmod +x eks-node-viewer && sudo mv eks-node-viewer /usr/local/bin
eks-node-viewer --resources cpu,memory --extra-labels eks-node-viewer/node-age
```

---

## 2. 샘플 애플리케이션 (ArgoCD GitOps)

### 2-1. 애플리케이션 구조

retail-store-sample (EKS Workshop 공식 데모 앱). 마이크로서비스 6개로 구성:

| Component | 역할 | 배치 노드 |
|---|---|---|
| **UI** | 프론트엔드 + API 게이트웨이 | initial |
| **Catalog** | 상품 카탈로그 API + MySQL (statefulset) | initial |
| **Cart** | 장바구니 API + DynamoDB local | self-managed |
| **Checkout** | 주문 오케스트레이션 + Redis | **Karpenter (CheckoutApp taint)** |
| **Orders** | 주문 처리 API + MySQL + RabbitMQ | **blue-mng (OrdersApp taint)** |
| **Static assets** | 이미지/정적 파일 | **Fargate** |

### 2-2. ArgoCD App-of-Apps 패턴

![ArgoCD GitOps Workflow](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)
*출처: [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/en/stable/)*

```bash
# CodeCommit 클론
cd ~/environment
git clone codecommit::${REGION}://eks-gitops-repo

tree eks-gitops-repo/ -L 2
# eks-gitops-repo/
# ├── app-of-apps/        # 상위 애플리케이션 차트 (Application of Applications)
# │   ├── Chart.yaml
# │   ├── templates/
# │   └── values.yaml
# └── apps/               # 각 마이크로서비스 매니페스트
#     ├── assets, carts, catalog, checkout
#     ├── karpenter, orders, other, rabbitmq, ui
#     └── kustomization.yaml
```

`apps`는 모두 **Auto-Sync + Prune + SelfHeal** 정책. `Deployment.spec.replicas`와 `HPA.status`는 `ignoreDifferences`로 무시.

```yaml
# app-of-apps/templates/_application.yaml (요약)
ignoreDifferences:
- group: apps
  kind: Deployment
  jsonPointers:
    - /spec/replicas
    - /metadata/annotations/deployment.kubernetes.io/revision
- group: autoscaling
  kind: HorizontalPodAutoscaler
  jsonPointers: [/status]
syncPolicy:
  automated: { prune: true, selfHeal: true }
  syncOptions: [RespectIgnoreDifferences=true]
```

### 2-3. ArgoCD CLI 로그인

```bash
export ARGOCD_SERVER=$(kubectl get svc argo-cd-argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export ARGOCD_USER="admin"
export ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

argocd login ${ARGOCD_SERVER} --username ${ARGOCD_USER} --password ${ARGOCD_PWD} \
  --insecure --skip-test-tls --grpc-web

argocd app list
```

### 2-4. UI 외부 노출 (NLB) — 옵션

UI는 기본적으로 ClusterIP라 외부 접근이 안 되므로, 실습 동안 **무중단 검증**을 위해 NLB Service를 추가.

```bash
cat << EOF > ~/environment/eks-gitops-repo/apps/ui/service-nlb.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-type: external
  name: ui-nlb
  namespace: ui
spec:
  type: LoadBalancer
  ports:
  - { name: http, port: 80, protocol: TCP, targetPort: 8080 }
  selector:
    app.kubernetes.io/instance: ui
    app.kubernetes.io/name: ui
EOF

# kustomization.yaml에 등록 후 commit/push/sync
echo "  - service-nlb.yaml" >> ~/environment/eks-gitops-repo/apps/ui/kustomization.yaml
cd ~/environment/eks-gitops-repo
git add apps/ui/ && git commit -m "Add UI NLB" && git push
argocd app sync ui
```

---

## 3. 업그레이드 사전 준비

### 3-1. EKS Upgrade Insights

EKS는 다음 마이너 버전 업그레이드 시 발생할 수 있는 호환성 이슈를 사전 점검해주는 **Upgrade Insights**를 제공한다.

```bash
aws eks list-insights --filter kubernetesVersions=1.31 --cluster-name $CLUSTER_NAME | jq .
```

체크 항목 예시:
- **kube-proxy version skew** — 클러스터-노드 버전 차이 정책 위반 여부
- **Deprecated APIs removed in target version** — 다음 버전에서 제거되는 API 사용 여부
- **kubelet version skew**

> 💡 **핵심**: 업그레이드 전 반드시 모든 항목 `PASSING` 상태인지 확인. Deprecated API를 쓰고 있다면 미리 마이그레이션해야 한다.

### 3-2. PodDisruptionBudget (PDB) 설정

![Pod Disruption Budget](https://kubernetes.io/images/blog/2024-07-19-pdb-pitfalls/pdb-rolling-update.svg)
*Pod Disruption Budget 동작 — 노드 드레인 시 minAvailable 보장*

데이터 플레인 업그레이드 시 노드가 cordon/drain되는데, **PDB가 없으면 모든 Pod가 동시에 evict될 수 있다**. 실습에서는 `orders` 앱에 PDB를 추가:

```yaml
# apps/orders/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: orders-pdb
  namespace: orders
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: service
      app.kubernetes.io/instance: orders
      app.kubernetes.io/name: orders
```

### 3-3. PDB 동작 검증

```bash
# orders Pod가 떠있는 노드 추출
nodeName=$(kubectl get pods -n orders \
  -l app.kubernetes.io/name=orders \
  -o jsonpath="{.items[*].spec.nodeName}")

# 노드 드레인 시도 → PDB 위반으로 evict 거부됨
kubectl drain "$nodeName" --ignore-daemonsets --force --delete-emptydir-data
# error when evicting pods/"orders-..." -n "orders":
#   Cannot evict pod as it would violate the pod's disruption budget.

# 검증 후 uncordon
kubectl uncordon "$nodeName"
```

> ⚠️ **단일 replica + minAvailable=1 PDB 조합은 노드 업그레이드를 영원히 블록한다.** Blue/Green 노드그룹 마이그레이션 실습 직전에 `replicas=2`로 미리 늘려두는 이유.

---

## 4. In-place 클러스터 업그레이드 (1.30 → 1.31)

### 📋 실습 순서 한눈에 보기

```
[1] Control Plane (10분)        — Terraform으로 cluster_version 변경
        ↓
[2] EKS Add-ons (2분)           — coredns / kube-proxy / ebs-csi-driver 버전 업
        ↓
[3] Data Plane Nodes
    ├─ Managed In-place (25분)  — initial + custom MNG 업그레이드
    ├─ Managed Blue/Green (20분)— blue-mng → green-mng 마이그레이션
    ├─ Karpenter (10분)         — EC2NodeClass AMI ID 교체 → drift 자동 처리
    ├─ Self-managed (10분)      — Launch Template AMI 교체 → ASG instance refresh
    └─ Fargate (즉시)           — Deployment restart만으로 신규 버전 노드
```

> 📌 **모든 작업은 Terraform 코드 수정 → `terraform apply`로 실행**한다. AWS 콘솔이나 `eksctl`로도 가능하지만, GitOps + IaC 실무 패턴을 따른다.

---

### 4-1. Control Plane 업그레이드

#### 변경 사항

`terraform/variables.tf`에서 `cluster_version` 변수를 `1.30` → `1.31`로 변경.

```hcl
variable "cluster_version" {
  description = "EKS cluster version."
  type        = string
  default     = "1.31"   # ← 1.30에서 변경
}
```

#### 모니터링 세팅 (별도 터미널)

```bash
# 1) 업그레이드 전 파드 이미지 스냅샷 저장
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" \
  | tr -s '[[:space:]]' '\n' | sort | uniq -c > 1.30.txt

# 2) UI liveness 무중단 모니터링
UI_WEB=$(kubectl get svc -n ui ui-nlb \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')/actuator/health/liveness
while true; do curl -s $UI_WEB; echo; date; sleep 1; done

# 3) 클러스터 버전/엔드포인트 모니터링 (다른 터미널)
while true; do
  date
  aws eks describe-cluster --name $CLUSTER_NAME \
    | egrep 'version|endpoint"|issuer|platformVersion'
  sleep 2; echo
done
```

#### 적용

```bash
cd ~/environment/terraform
terraform plan -no-color > plan-output.txt
terraform apply -auto-approve   # 약 10분 소요
```

#### 검증 포인트

| 항목 | 변화 여부 | 의미 |
|---|---|---|
| `cluster.version` | `1.30` → `1.31` | ✅ 컨트롤 플레인 업그레이드 완료 |
| `cluster.endpoint` | **불변** | API 엔드포인트 동일 → kubeconfig 재발급 불필요 |
| `oidc.issuer` | **불변** | **IRSA 4개 컨트롤러 정상 동작** |
| `platformVersion` | 그대로 또는 +1 | EKS 패치 레벨 |
| 워커 노드 / 파드 | **재생성 없음** | n-3 skew 정책 안에서 컨트롤만 먼저 올라감 |

```bash
# 컨트롤 플레인만 1.31, 노드는 아직 1.30
kubectl get node
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" \
  | tr -s '[[:space:]]' '\n' | sort | uniq -c > 1.31.txt
diff 1.30.txt 1.31.txt   # 동일함
```

---

### 4-2. Add-on 업그레이드

#### 호환 버전 조회

```bash
# 현재 버전 + 가능한 업그레이드 버전 확인
eksctl get addon --cluster $CLUSTER_NAME

# 1.31에 호환되는 coredns 최신 10개
aws eks describe-addon-versions --addon-name coredns --kubernetes-version 1.31 \
  --query "addons[].addonVersions[:10].{Version:addonVersion,DefaultVersion:compatibilities[0].defaultVersion}" \
  --output table

# kube-proxy도 동일 패턴
aws eks describe-addon-versions --addon-name kube-proxy --kubernetes-version 1.31 \
  --query "addons[].addonVersions[:10].{Version:addonVersion,DefaultVersion:compatibilities[0].defaultVersion}" \
  --output table
```

#### `terraform/addons.tf` 수정

```hcl
eks_addons = {
  coredns = {
    addon_version = "v1.11.4-eksbuild.33"   # 1.31 호환
  }
  kube-proxy = {
    addon_version = "v1.31.14-eksbuild.9"   # 1.31 호환
  }
  vpc-cni = {
    most_recent = true
  }
  aws-ebs-csi-driver = {
    addon_version = "v1.59.0-eksbuild.1"
    service_account_role_arn = module.ebs_csi_driver_irsa.iam_role_arn
  }
}
```

#### 적용 + 모니터링

```bash
# 모니터링: coredns/kube-proxy 롤링 업데이트 관찰
while true; do
  date
  kubectl get pod -n kube-system -l 'k8s-app in (kube-dns, kube-proxy)'
  sleep 2; echo
done

# 적용 (약 1분)
terraform apply -auto-approve
```

> 💡 coredns는 Deployment(롤링), kube-proxy는 DaemonSet(노드 단위 롤링)으로 업데이트된다. CoreDNS는 PDB가 걸려있어 **항상 최소 1개는 Ready** 상태 유지 → DNS 무중단.

---

### 4-3. 관리형 노드그룹 — In-place

![EKS Node Types](https://docs.aws.amazon.com/images/eks/latest/userguide/images/managed-node-groups.png)

#### 핵심 개념

- **`mng_cluster_version`만 올리면** Terraform이 EKS Managed NG의 AMI를 자동으로 최신 EKS-optimized AMI로 교체.
- **`ami_id`를 명시적으로 지정한 NG**(custom AMI)는 자동 업그레이드되지 않으며, **`ami_id`도 같이 변경**해야 한다.

#### 사전 준비 — custom NG 추가

먼저 1.30용 AMI ID를 가져와서 `variables.tf`의 `ami_id`에 넣고, `base.tf`에 `custom` NG 추가.

```bash
# 1.30용 AMI
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.30/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --region $AWS_REGION --query "Parameter.Value" --output text
# ami-06441265f3c3cef0a
```

```hcl
# base.tf의 eks_managed_node_groups 블록에 추가
custom = {
  instance_types             = ["t3.medium"]
  min_size                   = 1
  max_size                   = 2
  desired_size               = 1
  ami_id                     = try(var.ami_id)
  ami_type                   = "AL2023_x86_64_STANDARD"
  enable_bootstrap_user_data = true
  update_config = { max_unavailable_percentage = 35 }
}
```

```bash
terraform apply -auto-approve   # custom NG 생성 (~2분)
```

#### 동시 업그레이드 — initial + custom

```hcl
# variables.tf
variable "mng_cluster_version" { default = "1.31" }     # 1.30 → 1.31
variable "ami_id"              { default = "ami-00e0cfd6e5895fe3a" }  # 1.31용 AMI
```

```bash
# 1.31용 AMI 조회
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.31/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --region $AWS_REGION --query "Parameter.Value" --output text
```

#### 모니터링 + 적용

```bash
# 모니터링 (별도 터미널)
while true; do
  aws autoscaling describe-auto-scaling-groups \
    --query 'AutoScalingGroups[*].AutoScalingGroupName' --output json | jq
  echo
  kubectl get node -L eks.amazonaws.com/nodegroup
  echo
  date
  echo
  kubectl get node -L eks.amazonaws.com/nodegroup-image | grep ami
  sleep 1; echo
done

# 적용 (약 20분)
terraform apply -auto-approve
```

#### Surge Upgrade 동작

EKS Managed NG는 **`max_unavailable_percentage = 35`** 설정에 따라:
1. 기존 노드를 cordon (`SchedulingDisabled`)
2. 신규 AMI로 노드 생성 → 파드 스케줄
3. 기존 노드 drain → terminate
4. PDB 위반 시 자동 대기

#### 정리 — custom NG 제거

다음 실습을 위해 `custom` NG 블록 제거 후 `terraform apply`.

---

### 4-4. 관리형 노드그룹 — Blue/Green

![Blue Green Node Group Upgrade](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2022/08/02/Blue-Green-Architecture-1.png)
*출처: AWS Containers Blog — Mitigating disruption during Amazon EKS cluster upgrade with blue/green node groups*

#### 왜 Blue/Green인가

- **Stateful 워크로드**(EBS PV 사용)는 in-place drain 시 PV 재마운트 이슈가 생길 수 있다.
- AZ별 EBS 격리 정책 때문에 stateful NG는 **단일 AZ에 묶어두는 게 모범 사례**.
- 따라서 **동일 AZ + 동일 taint/label로 새 NG를 만들고**, 구 NG를 삭제하면서 자연스럽게 마이그레이션.

#### 시나리오

| Before | After |
|---|---|
| `blue-mng` (1.30, AZ-a, OrdersApp taint) | `green-mng` (1.31, AZ-a, OrdersApp taint) |
| orders + orders-mysql 파드 실행 중 | 동일 파드가 green-mng로 이전 |

#### Step 1 — green-mng 추가

```hcl
# base.tf - eks_managed_node_groups에 추가
green-mng = {
  instance_types = ["m5.large", "m6a.large", "m6i.large"]
  subnet_ids     = [module.vpc.private_subnets[0]]   # AZ-a 동일
  min_size       = 1
  max_size       = 2
  desired_size   = 1
  update_config  = { max_unavailable_percentage = 35 }
  labels = { type = "OrdersMNG" }                     # 동일 label
  taints = [{
    key    = "dedicated"
    value  = "OrdersApp"
    effect = "NO_SCHEDULE"
  }]
  # cluster_version 미지정 → 부모(eks_managed_node_group_defaults)의 1.31 사용
}
```

```bash
terraform apply -auto-approve   # ~3분
kubectl get node -l type=OrdersMNG -L topology.kubernetes.io/zone
# blue-mng (1.30) + green-mng (1.31) 모두 us-west-2a에 위치
```

#### Step 2 — orders 파드 replicas 2로 증설 (PDB 안전장치)

```bash
cd ~/environment/eks-gitops-repo
sed -i 's/replicas: 1/replicas: 2/' apps/orders/deployment.yaml
git add apps/orders/deployment.yaml
git commit -m "scale orders to 2 for blue/green migration"
git push
argocd app sync orders

# ArgoCD가 replicas를 ignoreDifferences하므로 직접 scale도 필요
kubectl scale deploy -n orders orders --replicas 2
```

> 🚨 **이 단계를 스킵하면 NG 삭제가 PDB 위반으로 영원히 블록된다.** `replicas=1 + minAvailable=1` 조합은 drain 불가.

#### Step 3 — blue-mng 제거

`base.tf`에서 `blue-mng` 블록 통째로 삭제 후 apply.

```bash
# 모니터링
while true; do
  kubectl get node -l type=OrdersMNG
  echo
  kubectl get pod -n orders -l app.kubernetes.io/component=service -owide
  echo
  kubectl get deploy -n orders orders
  kubectl get pdb -n orders
  echo
  date
  sleep 2
done

terraform apply -auto-approve   # ~10분
```

#### 관찰되는 이벤트

```bash
kubectl get events --sort-by='.lastTimestamp' --watch
# NodeNotSchedulable    → cordon
# NodeNotReady          → drain 진행
# DeletingNode          → cloud provider에서 EC2 종료
# RemovingNode          → controller가 노드 객체 정리
```

---

### 4-5. Karpenter 노드 업그레이드

![Karpenter Architecture](https://karpenter.sh/docs/concepts/nodepools/nodepool-overview.png)
*출처: [Karpenter 공식 문서](https://karpenter.sh/)*

#### 핵심 개념

Karpenter는 **NodePool + EC2NodeClass** CRD로 동작:
- **NodePool**: 어떤 조건의 파드에 어떤 스펙의 노드를 붙일지 정의 (taint, label, instance family 등)
- **EC2NodeClass**: AMI, 보안그룹, 서브넷, IAM Role 등 인프라 레벨 설정

**업그레이드는 EC2NodeClass의 `amiSelectorTerms`만 바꾸면 끝.** Karpenter가 **drift**를 감지해 자동으로 노드를 교체한다.

#### 사전 정보

```bash
kubectl describe ec2nodeclass default | grep -A2 "Ami"
# Ami Family:        AL2023
# Ami Selector Terms:
#   Id:  ami-0f676a166352f02ab   ← 현재 1.30 AMI

kubectl describe nodepool default | grep -A5 "Disruption"
# Consolidate After:     Never
# Consolidation Policy:  WhenEmpty
```

#### Step 1 — 부하 생성으로 Karpenter 노드 확장 트리거

```bash
# checkout 앱 replicas 1 → 10
cd ~/environment/eks-gitops-repo
sed -i 's/replicas: 1/replicas: 10/' apps/checkout/deployment.yaml
git add apps/checkout/deployment.yaml
git commit -m "scale checkout to 10"
git push
argocd app sync checkout

# 또는 직접 scale (ArgoCD ignoreDifferences로 인해)
kubectl scale deploy checkout -n checkout --replicas 10
```

→ Karpenter NodePool이 **2번째 노드를 자동 프로비저닝** (1.30 AMI로).

#### Step 2 — AMI 교체 + Disruption Budget 설정

```bash
# 1.31 AMI 조회
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.31/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --region $AWS_REGION --query "Parameter.Value" --output text
# ami-00e0cfd6e5895fe3a
```

```yaml
# apps/karpenter/default-ec2nc.yaml
spec:
  amiSelectorTerms:
    - id: ami-00e0cfd6e5895fe3a   # ← 1.31로 변경

# apps/karpenter/default-np.yaml
spec:
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: Never
    budgets:                       # ← 추가
      - nodes: "1"                 # 한 번에 1개씩만 drift 처리
        reasons:
          - Drifted
```

> 💡 **`budgets.reasons: Drifted`**: AMI 변경으로 인한 drift만 노드 1개씩 처리하도록 제한. 운영에서는 보통 `nodes: "10%"` 같이 비율 기반으로 설정.

#### Step 3 — 적용 + 관찰

```bash
cd ~/environment/eks-gitops-repo
git add apps/karpenter/
git commit -m "karpenter: update AMI to 1.31 + drift budget"
git push
argocd app sync karpenter

# Karpenter 컨트롤러 로그에서 disruption 이벤트 관찰
kubectl -n karpenter logs deployment/karpenter -c controller --tail=33 -f
# {"effect":"NoSchedule","key":"karpenter.sh/disruption","value":"disrupting"}
```

```bash
# 결과 확인 (~10분 소요 추정)
kubectl get nodes -l team=checkout
# NAME                                       VERSION
# ip-10-0-2-216.us-west-2.compute.internal   v1.31.14-eks-bbe087e
# ip-10-0-4-136.us-west-2.compute.internal   v1.31.14-eks-bbe087e
```

---

### 4-6. Self-managed 노드 업그레이드

#### 핵심 개념

Self-managed NG는 **Launch Template의 `image_id`를 갱신**하면 ASG가 **Instance Refresh**를 통해 노드를 순차 교체.

#### 적용

```hcl
# base.tf - self_managed_node_groups
default-selfmng = {
  instance_type = "m5.large"
  ami_type      = "AL2023_x86_64_STANDARD"
  min_size      = 1
  max_size      = 2
  desired_size  = 2
  ami_id        = "ami-00e0cfd6e5895fe3a"   # ← 1.31용으로 교체
  subnet_ids    = module.vpc.private_subnets
  disk_size     = 100
  # ... cloudinit_pre_nodeadm 등
}
```

```bash
# 모니터링
while true; do
  kubectl get nodes -l node.kubernetes.io/lifecycle=self-managed
  echo
  aws ec2 describe-instances \
    --query "Reservations[*].Instances[*].[Tags[?Key=='Name'].Value | [0], ImageId]" \
    --filters "Name=tag:Name,Values=default-selfmng" --output table
  echo; date; sleep 2
done

terraform apply -auto-approve
```

#### Instance Refresh 동작 순서

1. 신규 AMI로 EC2 1대 추가 → Ready
2. **5분 워밍 대기** (Refresh 기본값)
3. 기존 EC2 1대 종료 + 신규 EC2 1대 생성
4. 반복하여 모두 교체

```bash
kubectl get nodes -l node.kubernetes.io/lifecycle=self-managed
# v1.31.14-eks-bbe087e   ← 모두 신규 버전
```

---

### 4-7. Fargate 노드 업그레이드

#### 핵심 개념

**Fargate는 노드 == 파드**. 파드를 재시작하면 자동으로 최신 EKS 호환 Fargate microVM이 새로 뜬다. **Deployment rollout restart가 곧 노드 업그레이드.**

```bash
# 현재 상태
kubectl get pods -n assets -o wide
kubectl get pod -n assets -o yaml | grep schedulerName
# schedulerName: fargate-scheduler

kubectl get node $(kubectl get pods -n assets -o jsonpath='{.items[0].spec.nodeName}') -o wide
# fargate-ip-10-0-47-39   v1.30.14-eks-f69f56f
```

#### 적용

```bash
# 단순히 deployment 재시작
kubectl rollout restart deployment assets -n assets
kubectl wait --for=condition=Ready pods --all -n assets --timeout=180s

# 결과 — 새 fargate 노드는 1.31
kubectl get node $(kubectl get pods -n assets -o jsonpath='{.items[0].spec.nodeName}') -o wide
# fargate-ip-10-0-35-224   v1.31.14-eks-f69f56f
```

---

### 🎯 최종 검증

```bash
kubectl get node
# 모든 노드가 v1.31.x로 통일되어야 함

# 추가 검증 매트릭스
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].[Tags[?Key=='Name'].Value | [0], ImageId]" \
  --output table

kubectl get node --label-columns=eks.amazonaws.com/capacityType,node.kubernetes.io/lifecycle,karpenter.sh/capacity-type,eks.amazonaws.com/compute-type
kubectl get node -L eks.amazonaws.com/nodegroup,karpenter.sh/nodepool
```

---

## 5. 핵심 정리 & 도전 과제

### 5-1. 노드 유형별 업그레이드 전략 비교

| 노드 유형 | 메커니즘 | 트리거 | 무중단성 | 운영 복잡도 |
|---|---|---|---|---|
| **Managed NG (in-place)** | EKS API surge update | `version` or `ami_id` 변경 | PDB로 보호 | ⭐⭐ |
| **Managed NG (blue/green)** | 신규 NG 생성 → 구 NG 삭제 | 새 NG 추가 + 구 NG 제거 | PDB + replicas≥2 필수 | ⭐⭐⭐ |
| **Karpenter** | Drift detection | EC2NodeClass `amiSelectorTerms` | Disruption Budget | ⭐⭐⭐ (제일 자동) |
| **Self-managed** | ASG Instance Refresh | Launch Template `image_id` | warm pool, 점진적 | ⭐⭐ |
| **Fargate** | Pod 재시작 | `kubectl rollout restart` | 가장 단순 | ⭐ |

### 5-2. 업그레이드 골든 룰

1. **순서 엄수**: Control Plane → Add-on → Data Plane 순. n-3 skew 정책 안에서.
2. **한 번에 한 마이너 버전**: `1.30 → 1.32` 점프 불가. 무조건 `1.30 → 1.31 → 1.32`.
3. **PDB는 필수, replicas≥2 권장**: 단일 replica는 drain 블로커.
4. **Stateful은 AZ 격리 + Blue/Green**: EBS PV 가진 워크로드는 in-place 위험.
5. **IRSA 영향 없음**: OIDC issuer가 그대로라서 컨트롤 플레인 업그레이드해도 SA 토큰은 유효.
6. **Add-on 호환 버전 확인**: `aws eks describe-addon-versions --kubernetes-version` 항상 먼저.
7. **Upgrade Insights 사전 점검**: Deprecated API 사용 여부.

### 5-3. 도전 과제

- [ ] 동일 절차로 **1.31 → 1.32** 업그레이드
- [ ] **1.32 → 1.33 → 1.34 → 1.35** 컨트롤 플레인만 연속 업그레이드 → 워커 노드와의 **버전 격차 허용 한계** (n-3) 검증
- [ ] **Blue/Green Cluster Upgrade** — 클러스터 자체를 새로 띄우고 서비스를 이전하는 방식 실습
- [ ] **AWS Backup for EKS** 연동 — Kubernetes 리소스 + EBS 스냅샷 백업/복원 시나리오
  - 참고: [AWS 한글 블로그](https://aws.amazon.com/ko/blogs/korea/secure-eks-clusters-with-the-new-support-for-amazon-eks-in-aws-backup/)

### 5-4. 참고 자료

- [Amazon EKS 표준 지원 버전](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-standard.html)
- [Amazon EKS 확장 지원](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-extended.html)
- [EKS Best Practices — Cluster Upgrades](https://aws.github.io/aws-eks-best-practices/upgrades/)
- [EKS Workshop — Cluster Upgrades](https://catalog.us-east-1.prod.workshops.aws/event/dashboard/en-US/workshop)
- [terraform-aws-eks-blueprints](https://github.com/aws-ia/terraform-aws-eks-blueprints)

