# AWS EKS Networking 정리
## Kubernetes Networking Model + AWS VPC CNI

> 본 문서는 다음 내용을 바탕으로 정리했다.
>
> - Kubernetes Networking Model
> - AWS VPC CNI
> - 실습 노션 메모
>
> 목적
>
> 1. 쿠버네티스가 요구하는 네트워크 모델이 무엇인지 이해한다.
> 2. AWS EKS에서 기본 CNI인 VPC CNI가 그 요구사항을 어떻게 만족시키는지 이해한다.
> 3. 실무에서 확인해야 할 포인트(ENI, Pod IP, Warm Pool, Prefix, Custom Networking, kube-proxy, 디버깅 포인트)를 한 번에 정리한다.

---

# 1. TL;DR

- Kubernetes 네트워킹의 핵심 요구사항은 **모든 Pod가 NAT 없이 서로 직접 통신할 수 있어야 한다**는 것이다.
- 같은 Pod 안의 컨테이너는 **같은 network namespace**를 공유하므로 `localhost`로 통신한다.
- 같은 노드의 Pod 간 통신은 보통 **veth pair + bridge(cni0)** 로 해결된다.
- 다른 노드의 Pod 간 통신은 구현 방식에 따라 크게 3가지다.
  - **Overlay**: VXLAN/IPIP로 캡슐화
  - **BGP**: 파드 대역의 라우팅 정보를 물리 네트워크에 전파
  - **Cloud-native routing**: 클라우드 인프라가 Pod IP를 직접 라우팅
- **AWS VPC CNI**는 세 번째 방식이다.
- EKS에서 VPC CNI는 Pod에 **VPC의 실제 IP**를 부여한다.
- 이때 IP는 AWS가 관리하므로, 매번 API를 직접 호출하면 느리다.
- 그래서 **ipamd(Local IP Address Management)** 가 ENI/IP를 미리 확보해서 **Warm Pool**로 관리한다.
- 운영에서 중요한 포인트는 다음이다.
  - **Pod density**
  - **ENI limit**
  - **서브넷 IP 고갈**
  - **WARM_ENI_TARGET / WARM_IP_TARGET / MINIMUM_IP_TARGET**
  - **Prefix Delegation**
  - **Custom Networking**
  - **Security Groups for Pods**

---

# 2. Kubernetes Networking Model

## 2.1 Kubernetes가 요구하는 네트워크 모델

Kubernetes는 Pod 네트워크에 대해 다음을 요구한다.

1. 모든 Pod는 같은 노드든 다른 노드든 **NAT 없이** 다른 Pod와 직접 통신할 수 있어야 한다.
2. 노드에서 실행되는 시스템 데몬(kubelet 등)은 같은 노드의 Pod와 통신할 수 있어야 한다.
3. 클러스터의 각 Pod는 **고유한 cluster-wide IP**를 가져야 한다.

즉, Kubernetes는 Pod를 마치 하나의 VM이나 물리 서버처럼 취급하고 싶어 한다.  
각 Pod가 고유 IP를 가지고, 그 IP로 직접 통신해야 한다.

---

## 2.2 왜 "NAT 없이"가 중요한가

Kubernetes 네트워킹에서 가장 중요한 말은 **"NAT 없이"**이다.

NAT는 패킷이 네트워크 경계를 넘을 때 출발지/목적지 IP를 바꾸는 기술이다.

- **SNAT**: 출발지 IP 변경
- **DNAT**: 목적지 IP 변경

하지만 Kubernetes가 원하는 것은 다음과 같다.

### 요청 예시

- Pod A → Pod B 요청 시
  - Pod A가 보낸 패킷: `src=PodA_IP, dst=PodB_IP`
  - Pod B가 받은 패킷도 그대로여야 한다: `src=PodA_IP, dst=PodB_IP`

### 응답 예시

- Pod B → Pod A 응답 시
  - Pod B가 보낸 패킷: `src=PodB_IP, dst=PodA_IP`
  - Pod A가 받은 패킷도 그대로여야 한다: `src=PodB_IP, dst=PodA_IP`

즉, **요청/응답 어느 구간에서도 IP가 변조되면 안 된다.**

---

## 2.3 Pod 내부 컨테이너는 왜 localhost로 통신할까

Kubernetes에서 Pod는 "컨테이너 묶음"이지만 네트워크 관점에서는 **하나의 네트워크 namespace**를 공유한다.

이걸 가능하게 하는 핵심이 `pause` 컨테이너다.

- Pod 생성 시 가장 먼저 pause 컨테이너가 뜬다.
- pause가 네트워크 namespace를 만든다.
- 이후 앱 컨테이너들은 자기 namespace를 따로 만드는 대신, pause의 namespace에 합류한다.

그래서 Pod 내부 컨테이너들은 다음을 공유한다.

- IP 주소
- network namespace
- 포트 공간

즉, 같은 Pod 안에서는 `localhost`로 통신 가능하다.

---

## 2.4 같은 노드의 Pod 간 통신

같은 노드의 Pod 간 통신은 일반적으로 아래 구조로 이뤄진다.

1. 각 Pod는 독립 network namespace를 가진다.
2. Pod 내부에는 `eth0` 인터페이스가 생긴다.
3. Host 쪽에는 그 Pod와 연결된 `veth` 인터페이스가 생긴다.
4. 이 veth 들을 `cni0` 같은 브리지에 붙여 하나의 L2 네트워크처럼 동작시킨다.

### 같은 노드 통신 이미지

![same-node-veth](https://sirzzang.github.io/assets/images/kubernetes-networking-model-pod-on-same-nodes-4-77b843fa0a5e95b089bd6bc57e3fbba2.png)

---

## 2.5 veth pair란

`veth pair`는 커널 내부의 가상 이더넷 케이블이다.

- 한쪽 끝은 Pod namespace 안의 `eth0`
- 반대쪽 끝은 Host namespace 안의 `veth-xxx`

한쪽으로 들어간 패킷은 반대쪽에서 바로 나온다.  
즉, namespace의 벽을 관통하는 양방향 파이프다.

---

## 2.6 bridge(cni0)란

Host에는 여러 Pod의 veth가 존재한다.  
이들을 하나의 네트워크처럼 묶어주는 것이 **bridge(cni0)**다.

브리지는 L2 스위치처럼 동작한다.

- Pod A의 패킷이 host의 veth로 올라온다.
- bridge가 목적지 Pod가 연결된 veth로 전달한다.
- 결과적으로 같은 노드 Pod끼리는 NAT 없이 직접 통신하게 된다.

---

## 2.7 다른 노드의 Pod 간 통신이 어려운 이유

문제는 **다른 노드**다.

같은 노드 안에서는 bridge로 해결할 수 있지만, 노드가 달라지면 물리 네트워크는 일반적으로 **Pod IP 대역 자체를 모른다**.

즉, Pod IP가 다른 노드에 있어도 그 IP를 어디로 보내야 하는지 모른다.

이를 해결하는 방식이 세 가지다.

---

## 2.8 방식 1: Overlay Network

Overlay는 파드 네트워크를 물리 네트워크 위에 하나 더 만드는 방식이다.

- Pod는 별도의 Pod CIDR을 사용
- 물리 네트워크는 Pod IP를 모름
- 대신 Pod 패킷을 **노드 IP로 캡슐화**해서 전달

대표 예시:

- Flannel VXLAN
- Calico IPIP

### Overlay 이미지

![overlay](https://sirzzang.github.io/assets/images/kubernetes-networking-model-pod-on-different-nodes-overlay-dc5cbf4cb65ef98ebcba546fca61ae94.png)

### Overlay 특징

- 장점
  - 인프라가 Pod CIDR을 몰라도 됨
  - 비교적 범용적
- 단점
  - 캡슐화/디캡슐화 오버헤드
  - MTU 이슈 가능
  - 경로 추적 복잡

---

## 2.9 방식 2: BGP 방식

BGP 방식은 물리 네트워크에게 Pod 대역의 경로를 직접 알려준다.

예시:

- `10.244.1.0/24 는 노드 B로 보내라`

대표적으로 Calico의 BGP 모드가 이 방식을 사용한다.

### BGP 이미지

![bgp](https://sirzzang.github.io/assets/images/kubernetes-networking-model-pod-on-different-nodes-bgp-1db54f7d1f9f07c2af47fca53d69cf4f.png)

### BGP 특징

- 장점
  - 캡슐화 오버헤드 없음
- 단점
  - 네트워크 장비가 BGP 지원해야 함
  - 운영 복잡도 증가

---

## 2.10 방식 3: Cloud-native Routing

클라우드 환경에서는 인프라가 Pod IP를 직접 라우팅하게 만들 수 있다.

AWS VPC CNI가 바로 이 방식이다.

핵심은 다음이다.

- Pod에 **VPC의 실제 IP**를 부여
- 물리/클라우드 네트워크 입장에서도 그 IP는 원래 아는 주소
- 별도 오버레이/캡슐화/BGP 없이 바로 라우팅 가능

---

## 2.11 세 방식 비교

| 방식 | 핵심 아이디어 | 장점 | 단점 |
| --- | --- | --- | --- |
| Overlay | Pod 패킷을 노드 IP로 캡슐화 | 인프라 의존 적음 | 오버헤드 존재 |
| BGP | Pod 대역 경로를 물리 네트워크에 광고 | 캡슐화 없음 | 운영 복잡 |
| Cloud-native | Pod에 인프라가 아는 실제 IP 부여 | 가장 직관적, 성능 좋음 | 클라우드 구조 의존 |

---

# 3. AWS VPC CNI

## 3.1 AWS VPC CNI란

AWS VPC CNI는 EKS의 기본 CNI 플러그인이다.

핵심 특징은 다음과 같다.

- Pod에 **VPC의 실제 IP**를 부여한다.
- Pod IP가 노드 IP와 같은 VPC 대역에 있다.
- 따라서 Pod 간 통신 시 별도 Overlay가 필요 없다.
- VPC 라우팅, Security Group, Flow Log 같은 AWS 네트워크 기능과 자연스럽게 연결된다.

### VPC CNI 개념 이미지

<img width="745" height="345" alt="image" src="https://github.com/user-attachments/assets/a0aac1d3-9ea2-4327-ac74-6558dd36b0ef" />


### Overlay vs VPC CNI 비교 이미지

![overlay-vs-vpc-cni](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/91ec90e4-0f6b-431c-9c8d-73516c93f5cf/Untitled.png)

### 현재 설정된 Node env
<img width="688" height="508" alt="image" src="https://github.com/user-attachments/assets/86baa4b0-aa65-403b-96db-bdac0d34c11c" />

### daemonset node 네용
<img width="689" height="277" alt="image" src="https://github.com/user-attachments/assets/468bfaf4-ab10-4904-8a20-a4221d746380" />

### 보조 프라이빗 확인
<img width="973" height="402" alt="image" src="https://github.com/user-attachments/assets/854b65db-70d3-4c1b-a353-0e0a686cad1d" />


---

## 3.2 VPC CNI의 핵심 구성 요소

AWS VPC CNI는 크게 두 요소로 나뉜다.

### 1) CNI 바이너리

- kubelet이 Pod 생성/삭제 시 호출한다.
- 실제 Pod namespace에 네트워크 인터페이스를 구성한다.

### 2) ipamd

- 장기 실행 노드 로컬 데몬
- ENI/IP를 미리 확보해 둔다
- 로컬 IPAM 역할을 한다
- Pod 생성 시 즉시 IP를 줄 수 있게 Warm Pool을 유지한다

---

## 3.3 왜 ipamd가 필요한가

AWS에서 VPC IP는 AWS 컨트롤 플레인이 관리한다.  
즉, Pod 하나 띄울 때마다 AWS API를 직접 호출해서 IP를 받으면 느리고 비효율적이다.

그래서 `ipamd`가 다음 일을 한다.

- 미리 ENI/IP를 확보
- 노드 안에서 local warm pool 유지
- Pod 생성 시 CNI가 `ipamd`에 로컬로 요청
- AWS API를 실시간으로 매번 치지 않고 빠르게 응답

즉:

- **AWS API 호출은 느리고 비싸다**
- **로컬 warm pool은 빠르다**

---

## 3.4 VPC CNI가 배포되는 방식

VPC CNI는 각 노드에 `aws-node`라는 DaemonSet으로 배포된다.

즉, 모든 워커 노드에 하나씩 올라가며 ENI/IP 관리를 담당한다.

---

## 3.5 슬롯(slot) 개념

VPC CNI의 IP 관리 기본 단위는 **slot**이다.

- ENI에는 인스턴스 타입마다 정해진 수의 슬롯이 있다.
- 각 슬롯에는 다음 중 하나가 들어간다.
  - IP 주소 1개
  - `/28` prefix 1개
- 각 ENI의 첫 슬롯은 **primary IP**로 예약되어 파드에 쓸 수 없다.
- Pod가 쓰는 것은 그 외 **secondary IP** 또는 prefix다.

---

## 3.6 노드당 최대 Pod 수

기본 공식은 다음과 같다.

```text
Max Pods = ENI 수 × (ENI당 슬롯 수 - 1) + 2
각 항의 의미는 다음과 같다.

ENI당 슬롯 수 - 1
각 ENI의 첫 번째 슬롯은 primary IP이므로 Pod 할당 대상에서 제외한다.
+ 2
aws-node, kube-proxy 같은 hostNetwork: true 파드를 보정하기 위한 값이다.

즉, Node의 Pod density는 단순히 CPU/메모리만이 아니라 ENI 수와 ENI당 슬롯 수에도 제한된다.
```

# EKS VPC CNI 네트워크 가이드

### 파드간 통신 확인 테스트

<img width="600" height="639" alt="image" src="https://github.com/user-attachments/assets/932f974e-1694-42f9-9c31-92a1c57d2944" />

### 파드 외부 통신 확인 테스트

<img width="1577" height="351" alt="image" src="https://github.com/user-attachments/assets/04665bb2-3d8d-4cd6-8d91-0db078e6bbc8" />


---

## 3.7 Pod IP 할당 흐름

Pod가 생성될 때의 흐름은 다음과 같다.

1. `kubelet`이 Pod 생성 이벤트를 받는다.
2. `kubelet`이 CNI 바이너리를 호출한다.
3. CNI가 `ipamd`에 사용 가능한 IP를 요청한다.
4. `ipamd`가 warm pool에서 IP를 꺼내 준다.
5. CNI가 Pod namespace에 `eth0`를 구성하고 IP를 붙인다.
6. Host 쪽 라우팅/iptables 설정이 반영된다.

> 📷 **이미지**: Pod 생성 흐름 — `![Pod 생성 흐름](이미지URL을_여기에_입력)`

### kube-ops-view 설치 화면

<img width="596" height="284" alt="image" src="https://github.com/user-attachments/assets/861dc662-33cb-4999-95c0-486304e641da" />

### 최대 파드 생성 확인

<img width="1009" height="324" alt="image" src="https://github.com/user-attachments/assets/038adeb8-373e-4217-8282-73efcd944cae" />


---

## 3.8 Pod가 삭제될 때

Pod가 삭제되면 **IP가 즉시 재사용되지 않는다.**

VPC CNI는 삭제된 Pod의 IP를 약 **30초** 정도 `cooldown cache`에 넣는다.

이유는 다음과 같다.

- `kube-proxy`가 iptables를 갱신할 시간을 주기 위함
- 너무 빠르게 IP를 재사용하면 엉뚱한 트래픽이 새 Pod로 갈 수 있음

> 즉, **IP 재사용 안정성을 위한 보호 장치**다.

---

## 4. VPC CNI IP 할당 모드

VPC CNI는 크게 **3가지 방식**으로 Pod IP를 할당할 수 있다.

### 4.1 Secondary IP Mode (기본)

기본 모드다.

- ENI의 슬롯 하나에 **개별 IP(`/32`)**를 넣는다.
- Pod 1개당 IP 1개 사용
- 구조가 가장 단순하다

> 📷 **이미지**: Secondary IP — `![Secondary IP Mode](이미지URL을_여기에_입력)`  
> 📷 **이미지**: AWS 문서 기반 Secondary IP — `![AWS Secondary IP](이미지URL을_여기에_입력)`

| 구분 | 내용 |
|------|------|
| ✅ 장점 | 이해하기 쉽다 / 소규모 클러스터에 적합 |
| ❌ 단점 | Pod density 한계가 비교적 빨리 옴 / 서브넷 IP 빠르게 소모 |

---

### 4.2 Prefix Delegation Mode

이 모드에서는 ENI slot 하나에 개별 IP 대신 **`/28` prefix**를 붙인다.

- `/28` 하나에는 **16개 IP**가 있다.
- 즉, slot 하나가 Pod IP 여러 개를 담는 컨테이너 역할을 하게 된다.

> 📷 **이미지**: Prefix Delegation — `![Prefix Delegation](이미지URL을_여기에_입력)`

| 구분 | 내용 |
|------|------|
| ✅ 장점 | Pod density 크게 증가 / ENI 슬롯 효율 좋아짐 |
| ❌ 단점 | Subnet 설계와 prefix 할당 이해 필요 / 운영 시 prefix 단위 개념 숙지 필요 |

---

### 4.3 Custom Networking

기본적으로 Pod IP는 Node의 ENI가 연결된 subnet에서 가져온다.  
하지만 IP가 부족해지면 **Pod 전용 subnet**을 따로 쓰고 싶어질 수 있다.

이때 쓰는 것이 **Custom Networking**이다.

- Node와 Pod가 서로 다른 subnet 대역을 사용
- 보조 CIDR을 추가해서 Pod IP 공간 확보
- IPv4 고갈 문제 완화에 유용

> 📷 **이미지**: Custom Networking — `![Custom Networking](이미지URL을_여기에_입력)`

> 💡 **실무 포인트**  
> 대규모 EKS에서 IPv4 부족이 흔하다.  
> 이때 **보조 CIDR + ENIConfig + Custom Networking** 조합이 자주 쓰인다.  
> 노드 대역과 파드 대역을 분리할 수 있다.

---

### 4.4 세 모드 비교

| 항목 | Secondary IP | Prefix Delegation | Custom Networking |
|------|:---:|:---:|:---:|
| 기본 사용 여부 | ✅ 기본값 | 옵션 | 옵션 |
| 슬롯 단위 | IP 1개 | `/28` prefix | IP 또는 Prefix |
| Pod density | 낮음~중간 | 높음 | 설계에 따라 다름 |
| IP 효율 | 상대적으로 낮음 | 높음 | 매우 유연 |
| 운영 난이도 | 낮음 | 중간 | 높음 |

---

## 5. Warm Pool 전략

VPC CNI는 빠른 Pod 생성을 위해 ENI/IP/Prefix를 미리 확보해 둔다.  
이를 조절하는 핵심 파라미터는 아래 3개다.

### 5.1 `WARM_ENI_TARGET`

> 추가로 유지할 **빈 ENI 개수**

```
WARM_ENI_TARGET = 1
# 현재 쓰는 ENI 외에, 사용하지 않는 여분 ENI 1개를 항상 확보
```

| 구분 | 내용 |
|------|------|
| ✅ 장점 | 다음 Pod 생성 시 빠름 |
| ❌ 단점 | ENI와 IP를 많이 낭비할 수 있음 |

---

### 5.2 `WARM_IP_TARGET`

> 추가로 유지할 **여유 IP 개수**

```
WARM_IP_TARGET = 10
# 현재 Pod가 5개면 → 총 15개 IP를 확보하려고 시도
```

| 구분 | 내용 |
|------|------|
| 특징 | ENI 단위보다 훨씬 세밀한 제어 가능 |
| 특징 | IP 부족 환경에서 더 효율적 |

---

### 5.3 `MINIMUM_IP_TARGET`

> 노드가 최소한 확보해야 할 **총 IP 수**

```
MINIMUM_IP_TARGET = 30
# Pod가 아직 없어도, 시작 시 최소 30개 IP 확보 시도
```

| 구분 | 내용 |
|------|------|
| 특징 | 대량 scale-out 시 초반 IP 할당 지연 감소 |
| 특징 | `WARM_IP_TARGET`과 함께 자주 사용 |

---

### 5.4 Warm Pool 전략 비교

| 항목 | `WARM_ENI_TARGET` | `WARM_IP_TARGET` | `MINIMUM_IP_TARGET` |
|------|:---:|:---:|:---:|
| 제어 단위 | ENI | IP | IP |
| 목적 | 여유 ENI 확보 | 여유 IP 확보 | 초기 최소 IP 확보 |
| 속도 | 매우 빠름 | 빠름 | 초기 확장에 유리 |
| 자원 효율 | 낮음 | 높음 | 중간 |

---

### 5.5 조합 정리

| 모드 | 웜 풀 전략 | 사용 파라미터 |
|------|------|------|
| Secondary IP | ENI 단위 | `WARM_ENI_TARGET` |
| Secondary IP | IP 단위 | `WARM_IP_TARGET`, `MINIMUM_IP_TARGET` |
| Prefix Delegation | ENI 단위 | `WARM_ENI_TARGET` |
| Prefix Delegation | IP 단위 | `WARM_IP_TARGET`, `MINIMUM_IP_TARGET` |
| Prefix Delegation | Prefix 단위 | `WARM_PREFIX_TARGET` |

---

### 5.6 운영 관점 추천

> 💡 일반적으로 다음 방향이 더 낫다.
> - `WARM_ENI_TARGET` 기반은 단순하지만 **낭비가 크다**.
> - 가능하면 `WARM_IP_TARGET` + `MINIMUM_IP_TARGET` 기반이 더 **세밀하고 효율적**이다.
> - 대규모 환경이라면 **Prefix Delegation** 고려 가치가 크다.

---

## 6. Security Groups for Pods

기본적으로 Pod는 Node ENI의 Security Group을 공유한다.  
하지만 필요한 경우 **Pod 단위로 Security Group**을 붙일 수 있다.

> 📷 **이미지**: Security Groups for Pods — `![Security Groups for Pods](이미지URL을_여기에_입력)`

**언제 유용한가**

- 특정 Pod만 외부 DB 접근 허용
- 보안 규칙을 워크로드 단위로 나누고 싶을 때
- 멀티테넌트 성격 클러스터

> ⚠️ **주의점**  
> 네트워크 구조가 더 복잡해진다.  
> 성능/운영 비용 고려 필요.  
> 모든 곳에 무조건 쓰기보다는 **필요한 워크로드에만** 적용하는 게 낫다.

---

## 7. 실습 환경 배포 및 기본 확인

### 7.1 Terraform으로 실습 환경 배포

```bash
git clone https://github.com/gasida/aews.git
cd aews/2w

export TF_VAR_KeyName=$(aws ec2 describe-key-pairs --query "KeyPairs[].KeyName" --output text)
export TF_VAR_ssh_access_cidr=$(curl -s ipinfo.io/ip)/32
echo $TF_VAR_KeyName $TF_VAR_ssh_access_cidr

terraform init
terraform plan
nohup sh -c "terraform apply -auto-approve" > create.log 2>&1 &
tail -f create.log
```

<img width="1618" height="926" alt="image" src="https://github.com/user-attachments/assets/a6853dbc-e199-40a1-8fe2-ac22fa2a5f14" />


---

### 7.2 kubeconfig 설정

```bash
terraform output -raw configure_kubectl
aws eks --region ap-northeast-2 update-kubeconfig --name myeks
kubectl config rename-context $(cat ~/.kube/config | grep current-context | awk '{print $2}') myeks
```

---

### 7.3 배포 후 확인 포인트

| 카테고리 | 확인 항목 |
|----------|-----------|
| **EKS 콘솔** | Overview / API Server endpoint / OIDC provider |
| **Compute** | Node Group / 노드 라벨 |
| **Networking** | Service CIDR / Subnet / Public·Private access |
| **Add-ons** | VPC CNI 버전·설정 |
| **Access** | IAM access entries |

---

### 7.4 기본 명령어

```bash
kubectl cluster-info
eksctl get cluster

kubens default

kubectl get node --label-columns=node.kubernetes.io/instance-type,eks.amazonaws.com/capacityType,topology.kubernetes.io/zone
kubectl get node --show-labels
kubectl get node -l tier=primary

kubectl get pod -A
kubectl get pdb -n kube-system

aws eks describe-nodegroup --cluster-name myeks --nodegroup-name myeks-1nd-node-group | jq
aws eks list-addons --cluster-name myeks | jq
eksctl get addon --cluster myeks
```

### 실행 결과


```
Kubernetes control plane is running at https://1CD283C90C41FF2D15B454FDCCF99C9F.gr7.ap-northeast-2.eks.amazonaws.com
  CoreDNS is running at https://1CD283C90C41FF2D15B454FDCCF99C9F.gr7.ap-northeast-2.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

  NAME  REGION          EKSCTL CREATED
  myeks ap-northeast-2  False

  NAME                                               STATUS   ROLES    AGE   VERSION               INSTANCE-TYPE   CAPACITYTYPE   ZONE
  ip-192-168-1-113.ap-northeast-2.compute.internal   Ready    <none>   65m   v1.34.4-eks-f69f56f   t3.medium       ON_DEMAND      ap-northeast-2a
  ip-192-168-2-64.ap-northeast-2.compute.internal    Ready    <none>   65m   v1.34.4-eks-f69f56f   t3.medium       ON_DEMAND      ap-northeast-2b

  NAME                                               STATUS   ROLES    AGE   VERSION               LABELS
  ip-192-168-1-113.ap-northeast-2.compute.internal   Ready    <none>   65m   v1.34.4-eks-f69f56f
  beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.medium,beta.kubernetes.io/os=linux,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup-image=ami-00c08b951e60be37e,eks.amazonaws.com/nodegroup=myeks-node-group,eks.amazonaws.com/sourceLau
  nchTemplateId=lt-0476fe65a00075438,eks.amazonaws.com/sourceLaunchTemplateVersion=1,failure-domain.beta.kubernetes.io/region=ap-northeast-2,failure-domain.beta.kubernetes.io/zone=ap-northeast-2a,k8s.io/cloud-provider-aws=5553ae84a0d29114870f67bbabd07d44,kubernetes.io/arc
  h=amd64,kubernetes.io/hostname=ip-192-168-1-113.ap-northeast-2.compute.internal,kubernetes.io/os=linux,node.kubernetes.io/instance-type=t3.medium,topology.k8s.aws/zone-id=apne2-az1,topology.kubernetes.io/region=ap-northeast-2,topology.kubernetes.io/zone=ap-northeast-2a
  ip-192-168-2-64.ap-northeast-2.compute.internal    Ready    <none>   65m   v1.34.4-eks-f69f56f
  beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=t3.medium,beta.kubernetes.io/os=linux,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup-image=ami-00c08b951e60be37e,eks.amazonaws.com/nodegroup=myeks-node-group,eks.amazonaws.com/sourceLau
  nchTemplateId=lt-0476fe65a00075438,eks.amazonaws.com/sourceLaunchTemplateVersion=1,failure-domain.beta.kubernetes.io/region=ap-northeast-2,failure-domain.beta.kubernetes.io/zone=ap-northeast-2b,k8s.io/cloud-provider-aws=5553ae84a0d29114870f67bbabd07d44,kubernetes.io/arc
  h=amd64,kubernetes.io/hostname=ip-192-168-2-64.ap-northeast-2.compute.internal,kubernetes.io/os=linux,node.kubernetes.io/instance-type=t3.medium,topology.k8s.aws/zone-id=apne2-az2,topology.kubernetes.io/region=ap-northeast-2,topology.kubernetes.io/zone=ap-northeast-2b

  No resources found

  NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
  default       netshoot-pod-f4b6b8dcd-cjs6j    1/1     Running   0          27m
  default       netshoot-pod-f4b6b8dcd-qmdpn    1/1     Running   0          27m
  default       netshoot-pod-f4b6b8dcd-tkmxg    1/1     Running   0          27m
  kube-system   aws-node-hhc52                  2/2     Running   0          65m
  kube-system   aws-node-xd827                  2/2     Running   0          65m
  kube-system   coredns-d487b6fcb-vjbmd         1/1     Running   0          65m
  kube-system   coredns-d487b6fcb-x879w         1/1     Running   0          65m
  kube-system   kube-ops-view-97fd86569-rs76p   1/1     Running   0          13m
  kube-system   kube-proxy-44wph                1/1     Running   0          65m
  kube-system   kube-proxy-rqswr                1/1     Running   0          65m

  NAME      MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
  coredns   N/A             1                 1                     65m

  {
    "nodegroup": {
      "nodegroupName": "myeks-node-group",
      "nodegroupArn": "arn:aws:eks:ap-northeast-2:239472355654:nodegroup/myeks/myeks-node-group/a2ce9523-69d9-a74a-26f0-332805dac9ff",
      "clusterName": "myeks",
      "version": "1.34",
      "releaseVersion": "1.34.4-20260318",
      "createdAt": "2026-03-26T22:13:01.091000+09:00",
      "modifiedAt": "2026-03-26T23:14:19.312000+09:00",
      "status": "ACTIVE",
      "capacityType": "ON_DEMAND",
      "scalingConfig": {
        "minSize": 1,
        "maxSize": 4,
        "desiredSize": 2
      },
      "instanceTypes": [
        "t3.medium"
      ],
      "subnets": [
        "subnet-0c1ab6ad41f4154d8",
        "subnet-0304e9e15a3202d6c",
        "subnet-03f0dc458915f8411"
      ],
      "amiType": "AL2023_x86_64_STANDARD",
      "nodeRole": "arn:aws:iam::239472355654:role/myeks-node-group-eks-node-group-20260326130541937500000005",
      "labels": {},
      "resources": {
        "autoScalingGroups": [
          {
            "name": "eks-myeks-node-group-a2ce9523-69d9-a74a-26f0-332805dac9ff"
          }
        ]
      },
      "health": {
        "issues": []
      },
      "updateConfig": {
        "maxUnavailablePercentage": 33
      },
      "launchTemplate": {
        "name": "default-20260326131252220100000008",
        "version": "1",
        "id": "lt-0476fe65a00075438"
      },
      "tags": {
        "Terraform": "true",
        "Environment": "cloudneta-lab",
        "Name": "myeks-node-group"
      }
    }
  }

  {
    "addons": [
      "coredns",
      "kube-proxy",
      "vpc-cni"
    ]
  }

  2026-03-26 23:20:49 [ℹ]  Kubernetes version "1.34" in use by cluster "myeks"
  2026-03-26 23:20:49 [ℹ]  getting all addons
  2026-03-26 23:20:50 [ℹ]  to see issues for an addon run `eksctl get addon --name <addon-name> --cluster <cluster-name>`
  NAME          VERSION                 STATUS  ISSUES  IAMROLE UPDATE AVAILABLE        CONFIGURATION VALUES
  coredns               v1.13.2-eksbuild.3      ACTIVE  0
  kube-proxy    v1.34.5-eksbuild.2      ACTIVE  0
  vpc-cni               v1.21.1-eksbuild.5      ACTIVE  0
```


---

## 8. 노드에서 네트워크 정보 확인

### 8.1 EC2 인스턴스 IP 확인

```bash
aws ec2 describe-instances \
  --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" \
  --filters Name=instance-state-name,Values=running \
  --output table
```

<img width="570" height="225" alt="image" src="https://github.com/user-attachments/assets/4d9ab837-b98f-4f98-b685-878ffeadb46d" />


---

### 8.2 워커 노드 SSH 접속

```bash
  - N1=13.124.39.149 (192.168.1.113, ap-northeast-2a)                                                                            
  - N2=3.36.17.255 (192.168.2.64, ap-northeast-2b)

for i in $N1 $N2 $N3; do
  echo ">> node $i <<"
  ssh -o StrictHostKeyChecking=no ec2-user@$i hostname
  echo
done
```

### 실행 결과

```
  >> node 13.124.39.149 <<                                                                                                       
  ip-192-168-1-113.ap-northeast-2.compute.internal                                                                
                                                                                                                                 
  >> node 3.36.17.255 <<                                                                                                       
  ip-192-168-2-64.ap-northeast-2.compute.internal
```

---

### 8.3 aws-node / kube-proxy 설정 확인

```bash
kubectl get daemonset aws-node --namespace kube-system -o wide
kubectl describe daemonset aws-node --namespace kube-system

kubectl describe cm -n kube-system kube-proxy-config
kubectl describe cm -n kube-system kube-proxy-config | grep iptables: -A5

kubectl get ds aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env'
```

### 실행 결과
<img width="636" height="648" alt="image" src="https://github.com/user-attachments/assets/510b1796-30bf-4a49-983e-85fce832f2f1" />

<img width="656" height="628" alt="image" src="https://github.com/user-attachments/assets/3263d043-d4f5-4003-a65c-5ad966187058" />

- kube-proxy configmap

<img width="388" height="604" alt="image" src="https://github.com/user-attachments/assets/72157339-ce93-4f99-b11e-a2210a5c1072" />


---

### 8.4 파드/노드 IP 확인

```bash
kubectl get pod -n kube-system -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase
kubectl get pod -A -o name
kubectl get pod -A -o name | wc -l
```

### 실행 결과

<img width="429" height="477" alt="image" src="https://github.com/user-attachments/assets/be93e119-fc87-4032-8269-7e4d213965f9" />


---

### 8.5 CNI 로그 및 네트워크 상태 확인

```bash
for i in $N1 $N2 $N3; do
  echo ">> node $i <<"
  ssh ec2-user@$i tree /var/log/aws-routed-eni
  echo
done

for i in $N1 $N2 $N3; do
  echo ">> node $i <<"
  ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/plugin.log | jq
  echo
done

for i in $N1 $N2 $N3; do
  echo ">> node $i <<"
  ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/ipamd.log | jq
  echo
done
```
### 실행 결과

<img width="655" height="835" alt="image" src="https://github.com/user-attachments/assets/89154c54-1409-4648-a1c7-9263b6d440fb" />


---

### 8.6 라우팅 / 인터페이스 / iptables 확인

```bash
for i in $N1 $N2 $N3; do
  echo ">> node $i <<"
  ssh ec2-user@$i sudo ip -br -c addr
  echo
done

for i in $N1 $N2 $N3; do
  echo ">> node $i <<"
  ssh ec2-user@$i sudo ip -c route
  echo
done

ssh ec2-user@$N1 sudo iptables -t nat -S
ssh ec2-user@$N1 sudo iptables -t nat -L -n -v
```

### 실행 결과
```
⏺ EKS 워커 노드 네트워크 정리                         

  1. IP 주소 (ip addr)                                                                                                           
   
  ┌──────────────────────┬───────────────────────┬─────────────────────┐                                                         
  │         항목         │ Node1 (13.124.39.149) │ Node2 (3.36.17.255) │                                                       
  ├──────────────────────┼───────────────────────┼─────────────────────┤
  │ Primary ENI (ens5)   │ 192.168.1.113/24      │ 192.168.2.64/24     │
  ├──────────────────────┼───────────────────────┼─────────────────────┤
  │ Secondary ENI (ens7) │ 192.168.1.40/24       │ 192.168.2.56/24     │
  ├──────────────────────┼───────────────────────┼─────────────────────┤
  │ veth 인터페이스      │ 3개 (Pod 연결용)      │ 3개 (Pod 연결용)    │
  └──────────────────────┴───────────────────────┴─────────────────────┘

  - ens6은 nginx 스케일다운 후 해제되어 없음
  - 각 veth는 Pod의 네트워크 네임스페이스와 연결

  ---
  2. 라우팅 (ip route)

  공통 구조:
  - default via 192.168.X.1 → 서브넷 게이트웨이로 기본 라우팅
  - Pod IP → 개별 eniXXX veth 인터페이스로 라우팅 (호스트 네트워크에서 Pod로 전달)

  Node1 예시:
  192.168.1.217 dev eni8a5b42590bf scope link    # netshoot pod
  192.168.1.87  dev eni632c6a34c72 scope link    # coredns 등

  Node2 예시:
  192.168.2.175 dev eniXXX scope link            # netshoot pod
  192.168.2.xxx dev eniXXX scope link            # coredns 등

  → VPC CNI는 각 Pod IP에 대해 호스트의 라우팅 테이블에 /32 엔트리를 추가하여 veth로 트래픽 전달

  ---
  3. iptables NAT 규칙

  SNAT (외부 통신):
  AWS-SNAT-CHAIN-0:
    -d 192.168.0.0/16 → RETURN (VPC 내부는 SNAT 안 함)
    ! -d 192.168.0.0/16 → SNAT to 192.168.1.113 (Node1 Primary ENI IP)
  - Pod → 외부 인터넷 트래픽: Node의 Primary ENI IP로 SNAT
  - Pod → VPC 내부 트래픽: SNAT 없이 Pod IP 그대로 사용
  - 매칭 패킷: 2,291건 (외부 통신 발생한 횟수)

  kube-proxy (서비스 로드밸런싱):
  KUBE-SVC-xxx (kube-dns 10.100.0.10:53):
    → 50% 확률로 coredns Pod1 (예: 192.168.1.87:53)
    → 50% 확률로 coredns Pod2 (예: 192.168.2.175:53)
  - kube-proxy가 iptables 모드로 동작
  - ClusterIP 서비스 → iptables statistic random probability 규칙으로 Pod에 분배

  ---
  요약 다이어그램

  [Pod] → veth → ens5/ens7 → VPC 라우팅
                   │
                   ├─ VPC 내부 (192.168.0.0/16): Pod IP 그대로
                   └─ 외부 인터넷: SNAT → Node Primary ENI IP

```

---

### 8.7 ipamd 디버깅 엔드포인트

```bash
for i in $N1 $N2 $N3; do
  echo ">> node $i <<"
  ssh ec2-user@$i curl -s http://localhost:61679/v1/enis | jq
  echo
done
```

### 실행 결과

<img width="546" height="476" alt="image" src="https://github.com/user-attachments/assets/cb5dbc4b-4717-435a-849f-b8cfa1878eb1" />


---

## 9. kube-proxy와 iptables

EKS의 `kube-proxy`는 일반적으로 **iptables 모드**로 동작한다.

Service가 많고 Endpoint가 많아지면 iptables rule도 많아진다.  
대규모 클러스터에서는 이게 **성능 문제**로 이어질 수 있다.

### 9.1 주요 설정값

| 파라미터 | 설명 |
|----------|------|
| `minSyncPeriod` | Service/Endpoint 변경이 몰릴 때 iptables 갱신 배치 주기. 작을수록 반영 빠르나 CPU 사용량 증가 |
| `syncPeriod` | 주기적 정리/재동기화 주기. 너무 크게 늘리는 것은 비추천 |

---

## 10. 워커 노드 네트워크 구조 이해

실습 노드에서 관찰할 수 있는 핵심은 다음과 같다.

- **Host namespace**와 **Pod namespace**가 분리된다.
- `aws-node`, `kube-proxy` 같은 인프라 파드는 `hostNetwork: true`로 동작할 수 있다.
- Pod는 Host 쪽의 `eniX` / `veth` / `route` / `iptables`와 연결된다.
- `CoreDNS` 같은 일반 Pod는 VPC CNI로 할당된 secondary IP를 사용한다.

> 📷 **이미지**: 워커 노드 네트워크 구조 — `![Worker Node Network](이미지URL을_여기에_입력)`  
> 📷 **이미지**: ENI / Secondary IP 관찰 — `![ENI Secondary IP](이미지URL을_여기에_입력)`

---

## 12. 운영 관점에서 꼭 알아야 할 것

### 12.1 EKS에서 자주 맞는 문제

#### 1) 서브넷 IP 고갈 (가장 흔함)

**증상:** Pod Pending / ENI attach 실패 / IP 할당 실패

**대응:**
- Subnet 확장 (`/24` → `/22` 이상 고려)
- Prefix Delegation
- Custom Networking

---

#### 2) Pod density 부족

CPU/Memory는 남는데 Pod를 더 못 올리는 상황.

**이유:** ENI 수 한도 / ENI당 슬롯 수 한도 / `maxPods` 설정

**대응:**
- 인스턴스 타입 변경
- Prefix Delegation
- `max-pods` 계산 재검토

---

#### 3) ENI attach / EC2 API 의존

VPC CNI는 AWS API를 사용한다.  
대규모 scale-out 시 **ENI attach/API throttling**이 병목이 될 수 있다.

**대응:**
- Warm Pool 적절히 조정
- Prefix 사용
- 미리 capacity 확보

---

#### 4) kube-proxy / iptables 복잡도

서비스 수가 많고 endpoint churn이 심하면 iptables 동기화 비용이 커진다.

**대응:**
- `kube-proxy` 설정 확인
- CoreDNS / 서비스 구조 최적화
- 대규모라면 네트워킹 구조 전반 점검

---

### 12.2 Prefix Delegation을 고려할 시점

아래 조건에 해당하면 Prefix Delegation을 진지하게 검토할 가치가 크다.

- 노드당 파드 수를 많이 올리고 싶다.
- IP 효율을 개선하고 싶다.
- 대규모 워크로드를 운영한다.
- ENI/slot 한계가 체감된다.

### 12.3 Custom Networking을 고려할 시점

- RFC1918 대역이 부족하다.
- 기존 VPC 대역이 거의 고갈됐다.
- Pod 대역과 Node 대역을 분리하고 싶다.
- 대규모 멀티클러스터/멀티워크로드 운영 중이다.

---

## 13. 추천 점검 순서

네트워크 이슈가 생기면 아래 순서로 확인한다.

1. Pod가 IP를 받았는가
2. 해당 IP가 어느 ENI에서 왔는가
3. `aws-node` 상태는 정상인가
4. `ipamd` 로그에 ENI/IP 확보 실패는 없는가
5. 노드 라우팅 테이블은 정상인가
6. `kube-proxy` / `iptables` 규칙은 정상인가
7. Subnet 사용률이 너무 높지 않은가
8. Security Group / NACL이 막고 있지 않은가

---

## 14. 핵심 요약

### Kubernetes Networking Model

- Pod는 **cluster-wide unique IP**를 가진다.
- Pod 간 통신은 **NAT 없이 직접** 가능해야 한다.
- 같은 Pod 내부 컨테이너는 `localhost`로 통신한다.
- 다른 노드 Pod 간 통신 방식: **Overlay / BGP / Cloud-native routing**

### AWS VPC CNI

- Pod에 **VPC 실제 IP**를 부여한다.
- Overlay 없이 VPC가 Pod IP를 직접 라우팅한다.
- `ipamd`가 ENI/IP를 미리 확보해 **Warm Pool**로 관리한다.
- 운영 핵심: **Pod density, ENI limit, Subnet IP 고갈, Warm Pool 튜닝**

### 실무 결론

| 상황 | 권장 방식 |
|------|-----------|
| 작은 환경 | Secondary IP mode로 시작 |
| 파드 밀도 중요 | Prefix Delegation 고려 |
| IPv4 부족 | Custom Networking 고려 |
| 보안 분리 필요 | Security Groups for Pods 검토 |
| 성능/안정성 | kube-proxy, iptables, CoreDNS, Subnet 설계까지 함께 검토 |

---

## 15. 자주 보는 명령어 모음

### 클러스터 / 노드 / 애드온

```bash
kubectl cluster-info
kubectl get node -o wide
kubectl get node --show-labels
kubectl get pod -A
kubectl get ds -n kube-system
eksctl get addon --cluster myeks
aws eks list-addons --cluster-name myeks | jq
```

### aws-node / kube-proxy

```bash
kubectl describe ds aws-node -n kube-system
kubectl get ds aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env'
kubectl describe cm kube-proxy-config -n kube-system
```

### ENI / IP 상태

```bash
curl -s http://localhost:61679/v1/enis | jq
ip -br -c addr
ip -c route
iptables -t nat -S
iptables -t nat -L -n -v
```

### 파드/서비스 확인

```bash
kubectl get pod -o wide
kubectl get svc -A
kubectl describe pod <pod-name>
kubectl exec -it <pod-name> -- sh
```

---

## 16. 참고 이미지 목록

> 아래 이미지들은 실제 URL로 교체가 필요합니다.  
> `![설명](https://실제이미지URL)` 형식으로 입력하세요.

| 이미지 설명 | 마크다운 문법 예시 |
|-------------|-------------------|
| 같은 노드 Pod 통신 | `![같은 노드 Pod 통신](URL)` |
| 다른 노드 Pod 통신 - Overlay | `![다른 노드 Pod 통신 Overlay](URL)` |
| 다른 노드 Pod 통신 - BGP | `![다른 노드 Pod 통신 BGP](URL)` |
| VPC CNI와 일반 CNI 비교 | `![VPC CNI 비교](URL)` |
| Secondary IP Mode | `![Secondary IP Mode](URL)` |
| Prefix Delegation | `![Prefix Delegation](URL)` |
| Custom Networking | `![Custom Networking](URL)` |
| Worker Node 네트워크 구조 | `![Worker Node 네트워크 구조](URL)` |
| ENI / Secondary IP 예시 | `![ENI Secondary IP 예시](URL)` |
