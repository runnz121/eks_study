# AWS EKS 스터디 정리
## 1주차: 실습 환경 구성, EKS 배포 개요, Terraform 배포, kubeconfig 인증

> 참고 자료
>
> - [EKS: 실습 환경 구성하기](https://sirzzang.github.io/kubernetes/Kubernetes-EKS-00-Prerequisites/)
> - [EKS: EKS 배포 개요](https://sirzzang.github.io/kubernetes/Kubernetes-EKS-01-01-00-Installation-Overview/)
> - [EKS: Public-Public EKS 클러스터 - 1. Terraform 코드 분석](https://sirzzang.github.io/kubernetes/Kubernetes-EKS-01-01-01-Installation/)
> - [EKS: Public-Public EKS 클러스터 - 2. 배포](https://sirzzang.github.io/kubernetes/Kubernetes-EKS-01-01-02-Installation-Result/)
> - [EKS: Public-Public EKS 클러스터 - 3. kubeconfig와 EKS API 서버 인증](https://sirzzang.github.io/kubernetes/Kubernetes-EKS-01-01-03-Kubeconfig-Authentication/)

---

## 1. 전체 흐름 요약

이번 스터디는 단순히 EKS를 한 번 생성해보는 데서 끝나지 않고, 아래 흐름 전체를 이해하는 데 목적이 있다.

1. EKS 실습을 위한 로컬 및 AWS 계정 환경을 준비한다.
2. EKS를 배포하는 여러 방법을 비교하고, 왜 Terraform을 사용하는지 이해한다.
3. Terraform 코드가 어떤 AWS 리소스를 어떤 의도로 생성하는지 분석한다.
4. 실제로 `terraform init`, `plan`, `apply`를 수행하고 생성 결과를 확인한다.
5. 마지막으로 `kubectl`이 EKS에 어떻게 인증하는지 kubeconfig와 `aws eks get-token` 구조까지 이해한다.

즉 이번 학습의 핵심은 다음과 같다.

- **실습 환경 구성**
- **배포 방식 비교**
- **Terraform 코드 분석**
- **실제 배포 결과 확인**
- **EKS 인증 구조 이해**

---

## 2. 사전 준비: 실습 환경 구성

### 2.1 왜 사전 준비가 필요한가

EKS는 AWS가 Control Plane을 관리해주는 Managed Kubernetes 서비스이지만,  
실제로 EKS 클러스터를 만들고 사용하려면 아래와 같은 준비가 필요하다.

- Terraform 설치
- AWS CLI 설치
- AWS 계정 보안 설정
- IAM User 생성 및 Access Key 발급
- CLI 자격 증명 설정
- 워커 노드 접속용 EC2 Key Pair 생성

즉 EKS는 “AWS가 다 알아서 해주는 서비스”가 아니라,  
**Control Plane 운영 부담을 줄여주는 서비스**라고 이해하는 것이 맞다.

---

### 2.2 Terraform 설치

실습에서는 Terraform 버전을 맞추기 위해 `tfenv`를 사용한다.

#### tfenv 설치

```bash
brew install tfenv
```

#### 설치 가능한 Terraform 버전 확인

```bash
tfenv list-remote
```

#### 실습 버전 설치 및 적용

```bash
tfenv install 1.8.5
tfenv use 1.8.5
```

#### 현재 사용 버전 확인

```bash
tfenv list
terraform version
```

예시:

```bash
* 1.8.5 (set by /Users/USER/.config/tfenv/version)

Terraform v1.8.5
```

#### 자동완성 설치

```bash
terraform -install-autocomplete
```

---

### 2.3 AWS CLI 설치

EKS를 다룰 때는 `aws eks update-kubeconfig`, `aws eks get-token` 등의 명령이 필요하므로 AWS CLI v2가 필요하다.

#### Homebrew 설치

```bash
brew install awscli
aws --version
```

#### 공식 설치 파일 방식

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```

#### 버전 확인

```bash
aws --version
```

정상 예시:

```bash
aws-cli/2.x.x Python/3.x.x Darwin/...
```

---

### 2.4 AWS 계정 사용 원칙

AWS에서는 **Root User**와 **IAM User**를 구분해서 이해해야 한다.

#### Root User

- AWS 계정 생성 시 자동 생성
- 계정 전체 권한 보유
- 결제, 계정 설정, 보안 관련 핵심 작업 수행 가능
- 평소 작업에는 사용하지 않는 것이 원칙

#### IAM User

- IAM에서 별도로 생성
- 필요한 권한만 부여 가능
- Access Key 발급 가능
- CLI 및 일상적인 AWS 사용은 IAM User로 수행

즉 AWS 사용의 기본 원칙은 다음과 같다.

> **Root User는 잠가두고, 실제 작업은 IAM User로 수행한다.**

---

### 2.5 Root 로그인 후 해야 할 작업

처음 AWS 계정을 만든 뒤 Root User로 해야 하는 일은 다음과 같다.

1. 리전을 서울(`ap-northeast-2`)로 설정
2. Root User에 MFA 설정
3. 필요 시 Account Alias 지정
4. 관리자용 IAM User 생성
5. IAM User에 MFA 설정
6. 이후부터는 IAM User만 사용

#### 리전 변경 이미지

![AWS Region Seoul](https://sirzzang.github.io//assets/images/eks-prerequisites-root-login-5.png)

---

### 2.6 MFA 설정

Root User는 계정 전체 권한을 가지므로 반드시 MFA를 설정해야 한다.  
IAM User도 동일하게 MFA를 설정하는 것이 안전하다.

핵심 포인트는 다음과 같다.

- Root User MFA는 사실상 필수
- IAM User MFA도 필수
- 권한이 강할수록 로그인 보안을 더 강화해야 함

---

### 2.7 AWS CLI 자격 증명 설정

IAM User 생성 후 Access Key를 발급받았다면, 로컬 환경에서 AWS CLI를 설정한다.

#### 혹시 기존 환경변수가 있다면 해제

```bash
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_DEFAULT_REGION
```

#### 자격 증명 설정

```bash
aws configure
```

입력 예시:

```bash
AWS Access Key ID [None]: AKIA...
AWS Secret Access Key [None]: ...
Default region name [None]: ap-northeast-2
Default output format [None]: json
```

#### 설정 확인

```bash
aws configure list
aws s3 ls
aws sts get-caller-identity
```

`aws sts get-caller-identity` 결과에서 현재 IAM User의 ARN이 정상적으로 출력되면 설정이 완료된 것이다.

---

### 2.8 EC2 Key Pair 생성

워커 노드 접속을 위해 EC2 Key Pair를 생성한다.

```bash
aws ec2 create-key-pair \
  --key-name my-eks-keypair \
  --query 'KeyMaterial' \
  --output text > my-eks-keypair.pem

chmod 400 my-eks-keypair.pem
```

Key Pair는 리전 단위 리소스이므로 같은 리전의 여러 인스턴스에서 재사용할 수 있다.

---

## 3. EKS 배포 개요

### 3.1 EKS 배포 방식 3가지

글에서는 EKS 배포 방식을 크게 세 가지로 설명한다.

#### 1) AWS Console

- GUI 기반
- 처음 접할 때 이해하기 쉬움
- 반복 배포나 버전 관리에는 불리함

#### 2) eksctl

- EKS 전용 CLI
- 빠르게 클러스터를 생성 가능
- 내부적으로 CloudFormation을 사용

#### 3) Terraform

- 인프라를 코드로 정의
- 버전 관리 가능
- 반복 배포 및 재현성이 좋음
- 실무 친화적

이번 실습은 이 중 **Terraform 방식**을 사용한다.

---

### 3.2 eksctl은 내부적으로 무엇을 하는가

`eksctl create cluster`를 실행하면 실제로는 다음 흐름으로 동작한다.

1. 사용자가 `eksctl create cluster` 명령 실행
2. eksctl이 CloudFormation 템플릿 생성
3. AWS CloudFormation API 호출
4. AWS 리소스 생성

즉 eksctl은 사용 편의성을 높인 도구이지만, 내부적으로는 CloudFormation을 활용한다.

---

### 3.3 왜 Terraform을 사용하는가

Terraform을 사용하는 이유는 다음과 같다.

- 선언형 방식으로 인프라 정의 가능
- Git으로 변경 이력 관리 가능
- 동일한 환경을 반복적으로 생성 가능
- 코드 리뷰 가능
- 모듈을 통해 복잡한 설정을 단순화할 수 있음

즉 Terraform은 단순한 배포 도구가 아니라,  
**인프라를 코드로 관리하게 해주는 핵심 도구**다.

---

### 3.4 Terraform 핵심 개념

#### Resource

실제 AWS 자원을 의미한다.

예:

```hcl
resource "aws_security_group" "node_group_sg" {
  name   = "myeks-node-group-sg"
  vpc_id = module.vpc.vpc_id
}
```

#### Variable

환경별로 바뀔 수 있는 값을 변수로 정의한다.

예:

```hcl
variable "ClusterBaseName" {
  type    = string
  default = "myeks"
}
```

#### Module

여러 리소스를 묶어 재사용 가능한 단위로 만든 것.

예:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.5"
}
```

#### State

Terraform이 현재 관리 중인 리소스 상태를 저장하는 파일이다.  
보통 `terraform.tfstate` 파일에 저장된다.

#### Backend

State 파일을 어디에 저장할지 정의한다.  
실무에서는 보통 S3와 DynamoDB를 조합해 사용한다.

예:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "eks/terraform.tfstate"
    region         = "ap-northeast-2"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

---

## 4. Public-Public EKS 클러스터 구조

### 4.1 이번 실습 구조

이번 실습은 **Public-Public** 구조를 만든다.

의미는 다음과 같다.

- EKS API 서버: 퍼블릭 엔드포인트 사용
- Worker Node: 퍼블릭 서브넷에 배치
- NAT Gateway 없음
- 인터넷을 통해 API 서버 접근 가능

이 구조는 학습용으로는 단순하고 직관적이지만, 운영 환경에서는 보안상 더 신중한 구성이 필요하다.

---

### 4.2 변수 파일(`var.tf`)

주요 변수는 다음과 같다.

- 클러스터 이름: `myeks`
- Kubernetes 버전: `1.34`
- 워커 노드 인스턴스 타입: `t3.medium`

예시:

```hcl
variable "ClusterBaseName" {
  description = "Base name of the cluster."
  type        = string
  default     = "myeks"
}

variable "KubernetesVersion" {
  description = "Kubernetes version for the EKS cluster."
  type        = string
  default     = "1.34"
}

variable "WorkerNodeInstanceType" {
  description = "EC2 instance type for the worker nodes."
  type        = string
  default     = "t3.medium"
}
```

변수화의 장점은 코드 재사용성과 환경 분리다.

---

### 4.3 VPC 구성(`vpc.tf`)

이 실습의 VPC는 다음과 같은 특징을 가진다.

- 퍼블릭 서브넷 3개
- 여러 AZ에 분산
- NAT Gateway 없음
- Internet Gateway 사용
- 퍼블릭 IP 자동 할당
- EKS/ELB 태그 적용

#### DNS 관련 설정

```hcl
enable_dns_support   = true
enable_dns_hostnames = true
```

이 설정은 EKS에서 매우 중요하다.

- `enable_dns_support = true`
  - VPC 내부에서 DNS 해석 가능
- `enable_dns_hostnames = true`
  - 인스턴스에 DNS 호스트명 부여 가능

---

### 4.4 모듈 input과 output 이해

Terraform 모듈을 사용할 때 자주 헷갈리는 부분이다.

```hcl
vpc_id     = module.vpc.vpc_id
subnet_ids = module.vpc.public_subnets
```

여기서 의미는 다음과 같다.

- `public_subnet_blocks = var.public_subnet_blocks`
  - 모듈에 전달하는 입력값
- `module.vpc.public_subnets`
  - 모듈이 생성 후 반환하는 출력값

즉 이름이 비슷해 보여도 하나는 입력, 하나는 출력이다.

#### 관련 이미지

![VPC Module Input](https://sirzzang.github.io//assets/images/eks-terraform-vpc-module-input.png)

![VPC Module Output](https://sirzzang.github.io//assets/images/eks-terraform-vpc-module-output.png)

---

### 4.5 EKS 모듈 설정(`eks.tf`)

핵심 설정은 다음과 같다.

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.0"

  name               = var.ClusterBaseName
  kubernetes_version = var.KubernetesVersion

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.public_subnets

  endpoint_public_access  = true
  endpoint_private_access = false

  enabled_log_types = []

  enable_cluster_creator_admin_permissions = true
}
```

#### 주요 설정 해석

##### `endpoint_public_access = true`

EKS API 서버를 퍼블릭으로 연다.  
즉 로컬 환경에서 인터넷을 통해 `kubectl` 접근이 가능하다.

##### `endpoint_private_access = false`

프라이빗 엔드포인트는 열지 않는다.

##### `enabled_log_types = []`

Control Plane 로그를 비활성화한다.  
학습 비용 절감을 위한 선택이다.

##### `enable_cluster_creator_admin_permissions = true`

클러스터 생성 주체에게 관리자 권한을 부여한다.  
이 설정이 없으면 클러스터는 생성되더라도 `kubectl` 접근이 제한될 수 있다.

---

### 4.6 Managed Node Group

워커 노드는 EKS 관리형 노드 그룹을 사용한다.

관리형 노드 그룹의 장점은 다음과 같다.

- EKS가 노드 그룹 lifecycle 일부를 관리
- Auto Scaling Group과 연동
- 버전 관리가 상대적으로 편리
- 원하는 수량, 최소/최대 수량 설정 가능

부팅 시 초기화 스크립트를 넣을 수도 있다.

예:

```bash
#!/bin/bash
echo "Starting custom initialization..."
dnf update -y
dnf install -y tree bind-utils
echo "Custom initialization completed."
```

---

### 4.7 EKS Add-on

기본 애드온은 보통 다음을 사용한다.

- CoreDNS
- kube-proxy
- VPC CNI

예시:

```hcl
addons = {
  coredns = {
    most_recent = true
  }
  kube-proxy = {
    most_recent = true
  }
  vpc-cni = {
    most_recent    = true
    before_compute = true
  }
}
```

특히 VPC CNI는 워커 노드보다 먼저 준비되도록 구성할 수 있다.

---

## 5. Terraform 배포

### 5.1 `terraform init`

Provider와 module을 다운로드한다.

```bash
terraform init
```

이 단계에서 AWS provider, VPC module, EKS module 등이 설치된다.

---

### 5.2 `terraform plan`

```bash
terraform plan
```

글 기준 결과 예시는 다음과 같다.

```bash
Plan: 52 to add, 0 to change, 0 to destroy.
```

즉 총 52개의 리소스가 새로 생성될 예정이라는 뜻이다.

여기서 확인해야 하는 핵심은 다음과 같다.

- VPC 생성 여부
- Subnet 생성 여부
- EKS Cluster 생성 여부
- Managed Node Group 생성 여부
- Security Group 규칙
- 퍼블릭 엔드포인트 설정 여부

---

### 5.3 `terraform apply`

```bash
terraform apply
```

바로 승인하려면 다음과 같이 실행할 수 있다.

```bash
terraform apply -auto-approve
```

다만 실무에서는 `-auto-approve` 사용을 신중하게 해야 한다.

---

## 6. 배포 결과 확인

### 6.1 VPC 및 퍼블릭 서브넷

배포 후 AWS 콘솔에서 VPC와 Subnet이 정상적으로 생성되었는지 확인한다.

#### VPC 리소스 맵

![VPC Resource Map](https://sirzzang.github.io//assets/images/myeks-vpc-console-result-2.png)

#### 퍼블릭 서브넷

![Public Subnets](https://sirzzang.github.io//assets/images/myeks-public-subnet-console-result.png)

퍼블릭 서브넷 예시 CIDR:

- `192.168.1.0/24`
- `192.168.2.0/24`
- `192.168.3.0/24`

---

### 6.2 라우팅 테이블 구조

퍼블릭 라우팅 테이블은 보통 아래 구조를 가진다.

- `192.168.0.0/16 -> local`
- `0.0.0.0/0 -> igw`

즉 VPC 내부 통신은 local, 외부 인터넷 통신은 Internet Gateway를 통해 수행된다.

---

### 6.3 EKS 엔드포인트 확인

#### EKS 콘솔 이미지

![EKS Endpoint Public](https://sirzzang.github.io//assets/images/myeks-endpoint-access-public-console-result.png)

여기서 확인할 수 있는 내용은 다음과 같다.

- Kubernetes 버전
- 연결된 VPC 및 Subnet
- API 서버 엔드포인트 액세스 방식
- 퍼블릭 접근 허용 범위

이번 실습에서는 퍼블릭 접근을 열어두는 구조다.

---

### 6.4 Control Plane 로그

이번 실습에서는 아래 로그를 비활성화했다.

- API Server
- Audit
- Authenticator
- Controller Manager
- Scheduler

이는 학습 비용을 줄이기 위한 선택이다.  
다만 운영 환경에서는 필요한 로그를 활성화하는 것이 일반적이다.

---

### 6.5 IAM Access Entry

#### IAM Access Entry 이미지

![IAM Access Entry](https://sirzzang.github.io//assets/images/mycluster-admin-access-console-result.png)

여기서 중요한 점은 클러스터 생성 IAM 주체가 관리자 권한을 갖는다는 것이다.

대표적으로 다음 주체가 액세스 항목에 포함될 수 있다.

1. EKS 서비스 역할
2. 노드 그룹 역할
3. 클러스터 생성 IAM User 또는 Role

---

### 6.6 관리형 노드 그룹과 EC2 인스턴스

기본 설정 기준 워커 노드는 보통 2대가 생성된다.

- 인스턴스 타입: `t3.medium`
- 서로 다른 AZ에 분산 배치
- 퍼블릭 IP 및 퍼블릭 DNS 할당 가능

Managed Node Group은 내부적으로 Auto Scaling Group과 연결된다.

---

### 6.7 보안그룹

#### Worker Node Security Group 이미지

![Worker Node Security Groups](https://sirzzang.github.io//assets/images/myeks-ec2-security-group-console-result-1.png)

워커 노드에는 다음 보안그룹이 연결될 수 있다.

- EKS 모듈이 자동 생성한 보안그룹
- 추가로 지정한 사용자 정의 보안그룹

학습용 환경에서는 다소 넓게 열 수도 있지만, 운영 환경에서는 최소 권한 원칙에 따라 제한해야 한다.

---

## 7. Public-Public 구조의 의미와 한계

### 7.1 장점

Public-Public 구조의 장점은 다음과 같다.

- 구조가 단순함
- NAT Gateway가 없어 비용이 적음
- 외부에서 바로 `kubectl` 접속 가능
- 디버깅이 쉬움

즉 학습용으로는 매우 적합하다.

---

### 7.2 한계

하지만 운영 관점에서는 한계가 있다.

#### 1) API 서버가 퍼블릭에 노출됨

인터넷을 통해 접근 시도가 가능하다.

#### 2) 접근 CIDR 제한이 없으면 보안상 불리함

퍼블릭 접근 범위를 제한하지 않으면 불필요하게 넓게 열리게 된다.

#### 3) 워커 노드도 퍼블릭 서브넷 사용

운영 환경에서는 보통 프라이빗 서브넷에 두는 것이 일반적이다.

#### 4) 학습용 설정이 그대로 운영에 적합하지는 않음

즉 이 구조는 다음처럼 이해하면 된다.

> **학습에는 좋지만, 운영에는 보완이 필요하다.**

---

## 8. kubeconfig 설정

### 8.1 kubeconfig 생성

EKS에 `kubectl`로 접근하려면 kubeconfig를 생성해야 한다.

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name myeks
```

필요하면 context 이름을 보기 좋게 변경할 수 있다.

```bash
kubectl config rename-context $(kubectl config current-context) myeks
```

확인:

```bash
kubectl get nodes
```

예시:

```bash
NAME                                              STATUS   ROLES    AGE   VERSION
ip-192-168-2-21.ap-northeast-2.compute.internal   Ready    <none>   13h   v1.34.4-eks-f69f56f
ip-192-168-3-96.ap-northeast-2.compute.internal   Ready    <none>   13h   v1.34.4-eks-f69f56f
```

---

### 8.2 kubeconfig의 핵심: `exec`

EKS kubeconfig의 핵심은 정적인 토큰을 저장하는 것이 아니라,  
필요할 때마다 토큰을 발급하는 명령을 `exec`로 정의한다는 점이다.

예시 구조:

```yaml
users:
- name: arn:aws:eks:ap-northeast-2:ACCOUNT_ID:cluster/myeks
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args:
        - --region
        - ap-northeast-2
        - eks
        - get-token
        - --cluster-name
        - myeks
        - --output
        - json
```

즉 `kubectl`은 필요할 때 다음 명령을 호출한다.

```bash
aws --region ap-northeast-2 eks get-token --cluster-name myeks --output json
```

---

## 9. EKS API 서버 인증 구조

### 9.1 일반 Kubernetes와 다른 점

일반 Kubernetes는 인증서, static token 등 다양한 인증 방식을 사용할 수 있다.  
하지만 EKS는 AWS IAM과 연동되는 방식이 핵심이다.

즉 EKS에서는 클러스터 접근 주체를 다음 기준으로 본다.

- IAM User
- IAM Role

---

### 9.2 핵심 질문

여기서 중요한 질문은 다음이다.

> Kubernetes API 서버가 AWS 자격 증명의 실제 주인을 어떻게 검증할까?

이 문제를 해결하는 핵심이 **STS `GetCallerIdentity`** 이다.

---

### 9.3 STS `GetCallerIdentity`

이 API는 현재 요청을 서명한 AWS 주체가 누구인지 반환한다.

즉 다음 정보를 확인할 수 있다.

- Account ID
- ARN
- UserId

이 덕분에 EKS는 “이 요청을 보낸 IAM 주체가 누구인지”를 식별할 수 있다.

---

### 9.4 Secret Access Key를 직접 전달하지 않는 이유

AWS 요청은 Signature V4 방식으로 서명되는데, 이때 Secret Access Key가 필요하다.  
하지만 당연히 이 Secret을 EKS API 서버에 직접 넘길 수는 없다.

그래서 사용하는 방식이 **Pre-signed URL** 이다.

---

### 9.5 Pre-signed URL 기반 인증

클라이언트는 STS `GetCallerIdentity` 요청을 미리 서명한 URL 형태로 만든다.  
이 URL은 Secret 자체를 노출하지 않으면서도 “이 요청을 누가 서명했는지” 검증할 수 있게 해준다.

즉 인증 흐름은 대략 다음과 같다.

1. 클라이언트가 AWS 자격 증명으로 STS 요청용 pre-signed URL 생성
2. 이를 인코딩해서 EKS Bearer token 생성
3. `kubectl`이 이 토큰으로 API 서버 요청
4. API 서버가 토큰을 해석
5. STS를 통해 서명 주체를 검증
6. 해당 IAM 주체의 권한을 확인
7. 요청 승인 또는 거부

---

### 9.6 Bearer Token 형태

EKS에서 사용하는 토큰은 보통 아래와 같은 prefix를 가진다.

```text
k8s-aws-v1.aHR0cHM6Ly9zdHMu...
```

즉 단순한 임의 문자열 토큰이 아니라,  
AWS IAM 기반 인증 정보를 감싼 Bearer token이다.

---

### 9.7 인증 흐름 정리

정리하면 EKS 인증 흐름은 다음과 같다.

1. `kubectl` 실행
2. kubeconfig의 `exec` 항목 확인
3. `aws eks get-token` 호출
4. AWS 자격 증명으로 임시 토큰 생성
5. 토큰을 EKS API 서버에 전달
6. EKS가 STS `GetCallerIdentity` 기반으로 IAM 주체 검증
7. Access Entry 또는 RBAC 매핑 확인
8. 요청 처리

#### 인증 구조 이미지

![EKS API Auth Overview](https://sirzzang.github.io//assets/images/eks-api-auth-overview.png)

---

## 10. `aws eks get-token`

### 10.1 직접 실행

다음 명령으로 직접 토큰을 확인할 수 있다.

```bash
aws eks get-token --cluster-name myeks
```

결과 예시:

```json
{
  "kind": "ExecCredential",
  "apiVersion": "client.authentication.k8s.io/v1beta1",
  "spec": {},
  "status": {
    "expirationTimestamp": "2026-03-14T03:15:25Z",
    "token": "k8s-aws-v1.aHR0c..."
  }
}
```

핵심 필드는 다음과 같다.

- `status.token`
- `status.expirationTimestamp`

즉 이 토큰은 영구 토큰이 아니라 **짧은 수명의 임시 토큰**이다.

---

### 10.2 왜 kubeconfig에 정적 토큰을 넣지 않는가

이 토큰은 만료 시간이 짧기 때문에 고정 값으로 저장해두면 금방 쓸 수 없게 된다.

그래서 EKS는 kubeconfig에 토큰 자체를 넣는 대신,  
토큰 발급 명령을 `exec` 형태로 넣는다.

이 방식의 장점은 다음과 같다.

- 토큰 만료를 사용자가 신경 쓸 필요가 없음
- `kubectl`이 필요할 때마다 자동 발급
- 일반 Kubernetes 클러스터처럼 자연스럽게 사용 가능

---

### 10.3 다른 프로필이나 Role 사용

다른 AWS 프로필을 사용할 수도 있다.

```bash
AWS_PROFILE=other-user aws eks get-token --cluster-name myeks
```

Role을 사용해 토큰을 발급할 수도 있다.

```bash
aws eks get-token \
  --cluster-name myeks \
  --role-arn arn:aws:iam::123456789012:role/MyEKSRole
```

이 경우 실제 클러스터에서 인식되는 주체는 해당 Role이 된다.

---

## 11. 이번 스터디에서 핵심적으로 이해해야 할 것

### 11.1 EKS는 AWS 위에 올라간 Kubernetes다

EKS를 이해하려면 Kubernetes만 보면 안 되고,  
아래 AWS 구성 요소도 함께 이해해야 한다.

- VPC
- Subnet
- Route Table
- Internet Gateway
- Security Group
- IAM
- EC2
- Auto Scaling Group
- STS

즉 EKS는 단순한 Kubernetes 서비스가 아니라,  
**AWS 네트워크/보안 체계 위에 올려진 Managed Kubernetes 서비스**다.

---

### 11.2 Terraform은 선언형 인프라 관리 도구다

Terraform의 중요한 특징은 다음과 같다.

- 코드를 기준으로 인프라를 생성/변경
- State를 통해 현재 상태를 기억
- Module을 통해 복잡한 인프라 구성을 추상화
- 재현 가능하고 리뷰 가능한 인프라 구성 가능

---

### 11.3 EKS 인증은 IAM + STS를 중심으로 동작한다

일반 Kubernetes와 다르게 EKS는 다음 흐름을 가진다.

- AWS IAM 자격 증명 사용
- `aws eks get-token`으로 임시 토큰 생성
- STS `GetCallerIdentity` 기반 검증
- Access Entry / RBAC를 통한 권한 확인

즉 `kubectl get nodes` 한 줄 뒤에 AWS IAM 인증 흐름이 숨어 있다.

---

## 12. 최종 정리

이번 실습은 가장 단순한 형태의 EKS 환경을 Terraform으로 직접 만들어보면서,  
EKS가 단순히 “AWS가 대신 운영해주는 Kubernetes”가 아니라  
실제로는 VPC, Subnet, Security Group, IAM, STS, Auto Scaling Group 같은 AWS 자원들과 강하게 연결되어 있다는 점을 보여준다.

특히 중요한 부분은 인증 구조다.

보통 Kubernetes는 인증서나 고정 토큰을 떠올리기 쉽지만,  
EKS에서는 kubeconfig의 `exec`를 통해 `aws eks get-token`을 호출하고,  
이 토큰이 다시 STS `GetCallerIdentity`와 연결되어 IAM 주체를 검증한다.

즉 아래 흐름을 이해하는 것이 핵심이다.

1. kubeconfig `exec`
2. `aws eks get-token`
3. IAM 자격 증명 사용
4. STS를 통한 주체 검증
5. EKS Access Entry / RBAC 권한 확인
6. API 서버 요청 처리

이 흐름을 이해하면 이후에 다음 주제들로 확장하기 쉬워진다.

- Private Endpoint 구성
- Private Subnet 기반 EKS
- IRSA
- aws-auth와 Access Entry 차이
- Karpenter / Cluster Autoscaler
- 운영 환경 보안 강화

---

## 13. 개인적으로 정리한 포인트

- Public-Public 구조는 학습용으로 매우 직관적이다.
- 하지만 운영 환경에서는 API 서버와 워커 노드를 모두 퍼블릭으로 두는 것은 위험할 수 있다.
- Terraform 코드에서 모듈 input과 output을 구분해 읽는 습관이 중요하다.
- `enable_cluster_creator_admin_permissions = true` 같은 설정은 작아 보여도 실제 접근 가능 여부를 좌우한다.
- `kubectl`이 단순히 동작하는 것처럼 보여도, 내부적으로는 AWS IAM 인증 체계 위에서 복잡한 흐름이 수행된다.

---

## 14. 실습 명령어 모음

### Terraform 관련

```bash
brew install tfenv
tfenv list-remote
tfenv install 1.8.5
tfenv use 1.8.5
terraform version
terraform -install-autocomplete

terraform init
terraform plan
terraform apply
```

### AWS CLI 관련

```bash
brew install awscli
aws --version

unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_DEFAULT_REGION

aws configure
aws configure list
aws sts get-caller-identity
```

### Key Pair 생성

```bash
aws ec2 create-key-pair \
  --key-name my-eks-keypair \
  --query 'KeyMaterial' \
  --output text > my-eks-keypair.pem

chmod 400 my-eks-keypair.pem
```

### kubeconfig 설정

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name myeks
kubectl config rename-context $(kubectl config current-context) myeks
kubectl get nodes
```

### 토큰 확인

```bash
aws eks get-token --cluster-name myeks
```

---
