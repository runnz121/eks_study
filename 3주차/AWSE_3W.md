# Terraform 문법 정리 (3.9 ~ 3.15)

> 테라폼의 반복문, 조건문, 함수, 프로비저너, null_resource, moved 블록 등 핵심 문법 정리  
> 📚 참고: [HashiCorp 공식 Docs](https://developer.hashicorp.com/terraform/language) | [for_each](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each) | [count](https://developer.hashicorp.com/terraform/language/meta-arguments/count)

---

## 목차

1. [3.9 반복문 — for_each](#39-반복문--for_each)
2. [3.9 데이터 유형](#39-데이터-유형)
3. [3.9 for_each vs count 비교](#39-for_each-vs-count-비교)
4. [3.9 for Expressions](#39-for-expressions)
5. [3.9 dynamic 블록](#39-dynamic-블록)
6. [3.10 조건문](#310-조건문)
7. [3.11 함수](#311-함수)
8. [3.12 프로비저너](#312-프로비저너)
9. [3.13 null_resource와 terraform_data](#313-null_resource와-terraform_data)
10. [3.14 moved 블록](#314-moved-블록)
11. [3.15 CLI 시스템 환경 변수](#315-cli를-위한-시스템-환경-변수)

---

## 3.9 반복문 — `for_each`

> 📖 공식 문서: https://developer.hashicorp.com/terraform/language/meta-arguments/for_each

### 개념

`for_each`는 **map** 또는 **set** 타입의 데이터를 순회하며 **선언된 key 값 개수만큼 리소스를 생성**하는 반복문입니다.

- `count`가 숫자(인덱스) 기반이라면, `for_each`는 **키(key) 기반**입니다.
- 반복 중 현재 항목에는 `each` 객체로 접근합니다.

| 속성 | 설명 |
|------|------|
| `each.key` | 현재 인스턴스의 map 키 (set이면 값과 동일) |
| `each.value` | 현재 인스턴스의 map 값 |

> ⚠️ `for_each`는 **map, set 타입만 허용**합니다. list(tuple) 타입은 `toset()`, `tomap()` 등으로 변환 필요.

---

### 기본 사용법

#### Map 타입

```hcl
resource "azurerm_resource_group" "rg" {
  for_each = tomap({
    a_group       = "eastus"
    another_group = "westus2"
  })
  name     = each.key
  location = each.value
}
```

#### Set 타입 (toset 변환)

```hcl
resource "aws_iam_user" "the-accounts" {
  for_each = toset(["Todd", "James", "Alice", "Dottie"])
  name     = each.key
}
```

---

### 실습 1 — 기본 for_each (local_file)

**디렉터리 준비**

```bash
mkdir 3.9 && cd 3.9
touch main.tf
```

**main.tf**

```hcl
resource "local_file" "abc" {
  for_each = {
    a = "content a"
    b = "content b"
  }
  content  = each.value
  filename = "${path.module}/${each.key}.txt"
}
```

**실행 및 확인**

```bash
terraform init && terraform plan && terraform apply -auto-approve
terraform state list
```

**실행 결과 (state list)**

```
local_file.abc["a"]
local_file.abc["b"]
```

```bash
terraform state show 'local_file.abc["a"]'
```

```
# local_file.abc["a"]:
resource "local_file" "abc" {
    content              = "content a"
    content_base64sha256 = "..."
    content_md5          = "..."
    content_sha1         = "739afa0237fd8196c6b774a48676524bd95275b2"
    directory_permission = "0777"
    file_permission      = "0777"
    filename             = "./a.txt"
    id                   = "739afa0237fd8196c6b774a48676524bd95275b2"
}
```

```bash
ls *.txt    # a.txt  b.txt
cat a.txt   # content a
cat b.txt   # content b
```

**terraform console 확인**

```bash
echo "local_file.abc" | terraform console
echo 'local_file.abc["a"].content' | terraform console
# → "content a"
```

**tfstate 파일 내부 구조**

```json
"instances": [
  { "index_key": "a", "schema_version": 0 },
  { "index_key": "b", "schema_version": 0 }
]
```

> `count`의 `index_key`가 숫자(0, 1, 2...)인 반면, `for_each`는 **문자열 키**(`"a"`, `"b"`)입니다.

---

### 실습 2 — 변수(variable)에서 for_each 사용 + 리소스 간 참조

```hcl
variable "names" {
  default = {
    a = "content a"
    b = "content b"
    c = "content c"
  }
}

resource "local_file" "abc" {
  for_each = var.names
  content  = each.value
  filename = "${path.module}/abc-${each.key}.txt"
}

resource "local_file" "def" {
  for_each = local_file.abc          # abc 결과도 map이므로 바로 for_each 가능
  content  = each.value.content
  filename = "${path.module}/def-${each.key}.txt"
}
```

**실행 결과**

```
local_file.abc["b"]: Creating...
local_file.abc["a"]: Creating...
local_file.abc["c"]: Creating...
local_file.abc["c"]: Creation complete after 0s
local_file.abc["a"]: Creation complete after 0s
local_file.abc["b"]: Creation complete after 0s
local_file.def["a"]: Creating...
local_file.def["b"]: Creating...
local_file.def["c"]: Creating...
...
Apply complete! Resources: 6 added, 0 changed, 0 destroyed.
```

**state list 결과**

```
local_file.abc["a"]
local_file.abc["b"]
local_file.abc["c"]
local_file.def["a"]
local_file.def["b"]
local_file.def["c"]
```

---

### 실습 3 — 중간 값 삭제 시 동작 (for_each의 강점)

```hcl
variable "names" {
  default = {
    a = "content a"
    # b 삭제
    c = "content c"
  }
}
```

**실행 결과**

```
Plan: 0 to add, 0 to change, 2 to destroy.

local_file.def["b"]: Destroying...
local_file.def["b"]: Destruction complete after 0s
local_file.abc["b"]: Destroying...
local_file.abc["b"]: Destruction complete after 0s

Apply complete! Resources: 0 added, 0 changed, 2 destroyed.
```

**apply 후 state list**

```
local_file.abc["a"]
local_file.abc["c"]
local_file.def["a"]
local_file.def["c"]
```

> `b`에 해당하는 리소스만 정확히 삭제됩니다.  
> `count` 방식이면 인덱스가 밀려 `b`, `c` 모두 삭제 후 재생성되지만, `for_each`는 **키 기반**이라 해당 항목만 처리합니다.

---

### 실습 4 — 잘못된 타입 (tuple → 에러)

```hcl
resource "aws_iam_user" "the-accounts" {
  for_each = ["Todd", "James", "Alice", "Dottie"]  # ❌ tuple은 불가
  name     = each.key
}
```

**에러 메시지**

```
│ Error: Invalid for_each argument
│
│ The given "for_each" argument value is unsuitable: the "for_each" argument
│ must be a map, or set of strings, and you have provided a value of type tuple.
```

**해결: toset() 사용**

```hcl
resource "aws_iam_user" "the-accounts" {
  for_each = toset(["Todd", "James", "Alice", "Dottie"])  # ✅
  name     = each.key
}
```

---

## 3.9 데이터 유형

> 📖 참고: [Spacelift — Terraform Variable Types](https://spacelift.io/blog/terraform-variables)

![Terraform 데이터 타입 개요](https://cdn.hashnode.com/res/hashnode/image/upload/v1671454835658/YDCnCzKcZ.png)

> 📷 이미지 출처: [K21 Academy — Terraform Datatypes](https://k21academy.com/terraform-iac/terraform-variables-and-data-types/)

### 기본 유형

| 유형 | 설명 |
|------|------|
| `string` | 문자열 |
| `number` | 숫자 |
| `bool` | true / false |
| `any` | 모든 유형 허용 |

### 집합 유형

| 유형 | 표기 | 특징 |
|------|------|------|
| `list` | `[...]` | **인덱스** 기반 접근 |
| `map` | `{...}` | **키-값** 기반, 키 기준 정렬 |
| `set` | `[...]` | **값 기반** 집합, 키 기준 정렬, 중복 불허 |
| `object` | `{key=type}` | 서로 다른 타입 허용 key-value |
| `tuple` | `[type,...]` | 서로 다른 타입 허용 순서 배열 |

### 타입 변환 관계

```
tuple  →  tolist()  →  list
list   →  toset()   →  set

object →  tomap()   →  map  (단, 값 타입이 동일해야 함)
```

### terraform console 확인 예시

```bash
terraform console
# ------------------------------------------
type(["a","b"])              # → tuple
type(tolist(["a","b"]))     # → list(string)
type(toset(["a","b"]))      # → set(string)

type({a="a", b="b"})        # → object({a=string, b=string})
type(tomap({a="a", b="b"})) # → map(string)

var.list_set_tuple_b[0]     # list는 인덱스 접근 가능
var.map_object_a["a"]       # map은 키 접근 가능
# ------------------------------------------
```

---

## 3.9 for_each vs count 비교

> 📖 참고: [HashiCorp Support — count vs for_each](https://support.hashicorp.com/hc/en-us/articles/31348158569363-Terraform-count-versus-for-each-meta-argument)

![for_each vs count 비교](https://cdn.hashnode.com/res/hashnode/image/upload/v1735947623301/8817be6f-15b3-4374-8d38-86088d8c8561.webp)

> 📷 이미지 출처: [clouddevopsinsights — Count vs For_each](https://www.clouddevopsinsights.com/understanding-the-differences-between-terraform-count-and-foreach)

### count 방식

```hcl
resource "local_file" "abc" {
  count    = 3
  content  = "This is filename abc${count.index}.txt"
  filename = "${path.module}/abc${count.index}.txt"
}
```

**state list**

```
local_file.abc[0]
local_file.abc[1]
local_file.abc[2]
```

### 중간 값 삭제 시 동작 비교

**count — 삭제 후 인덱스 밀림 → 파괴적 재생성**

```
# 중간 index 1 삭제 시 plan 결과
# local_file.abc[1] must be replaced  ← index 2의 내용이 1로 밀림
# local_file.abc[2] will be destroyed

Plan: 1 to add, 0 to change, 2 to destroy.   ← 불필요한 재생성!
```

**for_each — 해당 키만 정확히 삭제**

```
# "b" 키 삭제 시 plan 결과
# terraform_data.my_for_each["b"] will be destroyed

Plan: 0 to add, 0 to change, 1 to destroy.   ← 정확하게 1개만 삭제!
```

| 비교 항목 | `count` | `for_each` |
|----------|---------|-----------|
| 인덱스 방식 | 숫자 (0, 1, 2...) | 문자열 키 ("a", "b"...) |
| 중간 삭제 시 | 이후 인덱스 모두 재생성 ❌ | 해당 키만 삭제 ✅ |
| 참조 방식 | `resource.name[0]` | `resource.name["key"]` |
| 허용 입력 타입 | number | map, set |
| 권장 사용 시점 | 단순 고정 개수 | 구별 가능한 여러 복사본 |

> **결론**: 리소스의 구별되는 여러 복사본을 만들 때는 **`count` 대신 `for_each`를 사용**하는 것이 바람직합니다.

---

## 3.9 for Expressions

> `for_each`와 **다른** 개념으로, **복합 형식 값의 형태를 변환**하는 데 사용합니다.  
> 📖 공식 문서: https://developer.hashicorp.com/terraform/language/expressions/for

### 기본 문법

```hcl
# 결과가 tuple ([ ] 사용)
[for <변수> in <컬렉션> : <표현식>]

# 결과가 object ({ } 사용)
{for <변수> in <컬렉션> : <키> => <값>}
```

### list 타입 예시

```hcl
variable "names" {
  type    = list(string)
  default = ["a", "b"]
}

output "A_upper_value" {
  value = [for v in var.names : upper(v)]
}

output "B_index_and_value" {
  value = [for i, v in var.names : "${i} is ${v}"]
}

output "C_make_object" {
  value = { for v in var.names : v => upper(v) }
}

output "D_with_filter" {
  value = [for v in var.names : upper(v) if v != "a"]
}
```

**terraform output 결과**

```
A_upper_value = ["A", "B"]
B_index_and_value = ["0 is a", "1 is b"]
C_make_object = { "a" = "A", "b" = "B" }
D_with_filter = ["B"]
```

### map 타입 예시

```hcl
variable "members" {
  type = map(object({ role = string }))
  default = {
    ab = { role = "member", group = "dev" }
    cd = { role = "admin",  group = "dev" }
    ef = { role = "member", group = "ops" }
  }
}

output "A_to_tuple" {
  value = [for k, v in var.members : "${k} is ${v.role}"]
}

output "B_get_only_role" {
  value = {
    for name, user in var.members : name => user.role
    if user.role == "admin"
  }
}

output "C_group" {
  value = {
    for name, user in var.members : user.role => name...
    # ...은 그룹화 모드 심볼 (키 중복 방지)
  }
}
```

**terraform output 결과**

```
A_tuple = ["ab is member", "cd is admin", "ef is member"]
B_filter_admin = { "cd" = { "role" = "admin" } }
C_group_by_role = {
  "admin"  = ["cd"]
  "member" = ["ab", "ef"]
}
```

### for 구문 규칙 요약

| 컬렉션 | 반환값 1개 | 반환값 2개 |
|--------|-----------|-----------|
| list | 값(v) | 인덱스(i), 값(v) |
| map | 키(k) | 키(k), 값(v) |
| set | 키/값(동일) | — |

| 묶는 기호 | 반환 타입 |
|----------|---------|
| `[ ]` | tuple |
| `{ }` | object (키 ⇒ 값 형식, `...` 그룹화 가능) |

---

## 3.9 dynamic 블록

> 📖 공식 문서: https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks  
> 📖 참고: [Spacelift — Terraform Dynamic Blocks](https://spacelift.io/blog/terraform-dynamic-blocks)

### 개념

리소스 **내부의 반복되는 속성 블록**을 동적으로 생성합니다.  
`count`/`for_each`가 리소스 전체를 반복하는 반면, `dynamic`은 **리소스 내부 블록**을 반복합니다.

```
Input (Variable/Local)
        │
        ▼
   for_each Iterator ──► content 블록 정의
        │
        ├──► Generated Block 1
        ├──► Generated Block 2
        └──► Generated Block N
```

> 📷 참고 이미지: [Spacelift — Terraform Dynamic Blocks](https://spacelift.io/blog/terraform-dynamic-blocks) | [KodeKloud — Dynamic Block Examples](https://kodekloud.com/blog/terraform-dynamic-block/)

### 사용 전/후 비교

**동적 블록 없이 (반복 작성)** — DRY 원칙 위반

```hcl
resource "aws_security_group" "example" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**dynamic 블록 사용** — DRY 원칙 준수 ✅

```hcl
locals {
  ingress_rules = [
    { port = 22,  description = "SSH" },
    { port = 443, description = "HTTPS" },
    { port = 80,  description = "HTTP" },
  ]
}

resource "aws_security_group" "example" {
  dynamic "ingress" {
    for_each = local.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = ingress.value.description
    }
  }
}
```

> `each.key` / `each.value` 대신 **블록명.key** / **블록명.value**를 사용합니다.

### 실습 — archive_file dynamic source

**static 방식**

```hcl
data "archive_file" "dotfiles" {
  type        = "zip"
  output_path = "${path.module}/dotfiles.zip"

  source { content = "hello a" filename = "${path.module}/a.txt" }
  source { content = "hello b" filename = "${path.module}/b.txt" }
  source { content = "hello c" filename = "${path.module}/c.txt" }
}
```

**dynamic 방식 (동일한 결과)**

```hcl
variable "names" {
  default = {
    a = "hello a"
    b = "hello b"
    c = "hello c"
  }
}

data "archive_file" "dotfiles" {
  type        = "zip"
  output_path = "${path.module}/dotfiles.zip"

  dynamic "source" {
    for_each = var.names
    content {
      content  = source.value
      filename = "${source.key}.txt"
    }
  }
}
```

**실행 결과**

```
$ terraform init -upgrade && terraform apply -auto-approve

data.archive_file.dotfiles: Read complete after 0s

No changes. Your infrastructure matches the configuration.
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

$ unzip -o dotfiles.zip -d /tmp/dotfiles_test
  inflating: /tmp/dotfiles_test/a.txt
  inflating: /tmp/dotfiles_test/b.txt
  inflating: /tmp/dotfiles_test/c.txt

$ cat /tmp/dotfiles_test/a.txt
hello a
```

> ⚠️ **주의**: dynamic 블록 남용은 코드 가독성을 떨어뜨립니다. 재사용 가능한 모듈 작성 시에만 사용 권장 ([공식 문서](https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks))

---

## 3.10 조건문

> 📖 공식 문서: https://developer.hashicorp.com/terraform/language/expressions/conditionals

### 개념 — 삼항 연산자

```hcl
# <조건> ? <true일 때 값> : <false일 때 값>
var.a != "" ? var.a : "default-a"
```

Java/JavaScript의 삼항 연산자와 동일한 구조입니다.

### 타입 일치 권장

```hcl
var.example ? 12 : "hello"            # ❌ 비권장 (타입 불일치)
var.example ? "12" : "hello"          # ✅ 권장
var.example ? tostring(12) : "hello"  # ✅ 권장
```

### 실습 — count와 결합한 리소스 생성 조건

```hcl
variable "enable_file" {
  type    = bool
  default = true
}

resource "local_file" "conditional" {
  count    = var.enable_file ? 1 : 0    # true면 생성, false면 미생성
  content  = "This file is conditionally created"
  filename = "${path.module}/conditional.txt"
}

output "file_created" {
  value = var.enable_file ? "File was created" : "File was NOT created"
}
```

**false 환경 변수 설정 후 실행**

```bash
export TF_VAR_enable_file=false
terraform apply -auto-approve
```

```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:
file_created = "File was NOT created"
```

**true(기본값)로 실행**

```bash
unset TF_VAR_enable_file
terraform apply -auto-approve
```

```
local_file.conditional[0]: Creating...
local_file.conditional[0]: Creation complete after 0s

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:
file_created = "File was created"
```

---

## 3.11 함수

> 📖 공식 문서: https://developer.hashicorp.com/terraform/language/functions

### 개념

테라폼에는 값의 유형을 변경하거나 조합하는 **내장 함수**가 제공됩니다.  
단, **사용자 정의 함수는 지원하지 않습니다.**

### 함수 카테고리

| 카테고리 | 대표 함수 |
|---------|---------|
| 문자열 | `upper()`, `lower()`, `title()`, `replace()`, `trimspace()` |
| 숫자 | `max()`, `min()`, `abs()`, `ceil()`, `floor()` |
| 컬렉션 | `toset()`, `tolist()`, `tomap()`, `length()`, `flatten()`, `merge()` |
| 인코딩 | `jsonencode()`, `jsondecode()`, `base64encode()` |
| IP 네트워크 | `cidrsubnet()`, `cidrnetmask()`, `cidrsubnets()`, `cidrhost()` |
| 파일 시스템 | `file()`, `filebase64()`, `templatefile()` |
| 날짜/시간 | `timestamp()`, `formatdate()` |
| 타입 변환 | `tostring()`, `tonumber()`, `tobool()` |

### 실습

```hcl
resource "local_file" "foo" {
  content  = upper("hello terraform")   # → "HELLO TERRAFORM"
  filename = "${path.module}/upper.txt"
}
```

**terraform console 확인**

```bash
terraform console
# ------------------------------------------
upper("hello")                         # → "HELLO"
lower("HELLO")                         # → "hello"
max(1, 5, 3)                          # → 5

cidrsubnet("10.0.0.0/16", 8, 1)       # → "10.0.1.0/24"
cidrsubnet("10.0.0.0/16", 8, 2)       # → "10.0.2.0/24"
cidrnetmask("10.0.0.0/16")            # → "255.255.0.0"
cidrsubnets("10.0.0.0/16", 8, 8, 8)  # → ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]

jsonencode({"hello"="world"})         # → "{\"hello\":\"world\"}"
# ------------------------------------------
```

---

## 3.12 프로비저너

> 📖 공식 문서: https://developer.hashicorp.com/terraform/language/resources/provisioners/syntax  
> 📖 참고: [Spacelift — Terraform Provisioners](https://spacelift.io/blog/terraform-provisioners)

### 개념

> ⚠️ **Provisioners are a Last Resort** — 프로비저너는 최후의 수단으로만 사용

- 프로바이더로 처리할 수 없는 **명령 실행, 파일 복사** 등을 담당
- **프로비저너 실행 결과는 tfstate에 저장되지 않음** → 선언적 보장 안됨
- 가능하면 `userdata`, `cloud-init`, `Packer` 등 대안 사용 권장

### 프로비저너 종류

| 종류 | 실행 위치 | 설명 |
|------|---------|------|
| `local-exec` | 로컬 (Terraform 실행 머신) | 로컬 명령 실행 |
| `remote-exec` | 원격 서버 | SSH/WinRM으로 원격 명령 실행 |
| `file` | 로컬 → 원격 | 파일/디렉터리 복사 |

> `file`과 `remote-exec`는 `connection` 블록이 반드시 필요합니다.

### local-exec

```hcl
resource "local_file" "provisioner_example" {
  content  = "Hello Provisioner"
  filename = "${path.module}/provisioner.txt"

  # 생성 시 실행
  provisioner "local-exec" {
    command = "echo 'Content: ${self.content}'"
  }

  # 실패해도 계속 진행
  provisioner "local-exec" {
    command    = "abc"
    on_failure = continue
  }

  # 삭제 시 실행
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Destroying file: ${self.filename}'"
  }
}
```

**apply 실행 결과**

```
local_file.provisioner_example: Provisioning with 'local-exec'...
local_file.provisioner_example (local-exec): Content: Hello Provisioner
local_file.provisioner_example: Provisioning with 'local-exec'...
local_file.provisioner_example (local-exec): /bin/sh: abc: command not found
local_file.provisioner_example: Creation complete after 0s

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**state show — 프로비저너 정보 없음 확인**

```bash
terraform state show local_file.provisioner_example
# → 프로비저너 실행 결과가 state에 전혀 없음!
```

**destroy 실행 결과**

```
local_file.provisioner_example: Provisioning with 'local-exec'...
local_file.provisioner_example (local-exec): Destroying file: ./provisioner.txt
local_file.provisioner_example: Destruction complete after 0s
```

**sensitive 변수 설정 시**

```hcl
variable "sensitive_content" {
  default   = "secret"
  sensitive = true    # 민감 정보 마킹
}
```

```
local_file.foo (local-exec): (output suppressed due to sensitive value in config)
```

### local-exec 고급 옵션

```hcl
resource "null_resource" "advanced_exec" {
  provisioner "local-exec" {
    interpreter = ["bash", "-c"]    # 실행 인터프리터
    working_dir = "/tmp"            # 실행 디렉터리
    environment = {
      ENV = "world!!"              # 환경변수
    }
    command = "echo Hello!! > file.txt && echo $ENV >> file.txt"
  }
}
```

**실행 결과**

```
null_resource.advanced_exec (local-exec): Executing: ["bash" "-c" "..."]
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

$ cat /tmp/file.txt
Hello!!
world!!
```

### remote-exec + file

```hcl
resource "aws_instance" "web" {
  # ...

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/keypair.pem")
    host        = self.public_ip
  }

  # 파일 복사
  provisioner "file" {
    source      = "script.sh"
    destination = "/tmp/script.sh"
  }

  # 원격 명령 실행
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/script.sh",
      "/tmp/script.sh args",
    ]
  }
}
```

---

## 3.13 null_resource와 terraform_data

> 📖 참고: [Spacelift — null_resource](https://spacelift.io/blog/terraform-null-resource)

### null_resource

**아무 작업도 수행하지 않는 가상 리소스**로, 프로비저닝 순서 조율이나 외부 명령 실행에 활용합니다.

**주요 시나리오**

- 프로비저닝 과정에서 명령어 실행
- 모듈, 반복문, 데이터 소스와 함께 사용
- 리소스 간 **상호 참조(Cycle)** 문제 해소

**Cycle 에러 예시**

```hcl
# ❌ 상호 참조 → Cycle 에러
resource "aws_instance" "example" {
  provisioner "remote-exec" {
    inline = ["echo ${aws_eip.myeip.public_ip}"]  # aws_eip 참조
  }
}
resource "aws_eip" "myeip" {
  instance = aws_instance.example.id              # aws_instance 참조
}
```

```
Error: Cycle: aws_eip.myeip, aws_instance.example
```

**null_resource로 해결 — 실행 순서 분리**

```hcl
resource "aws_instance" "example" { ... }  # eip 참조 제거

resource "aws_eip" "myeip" {
  instance = aws_instance.example.id
}

resource "null_resource" "echomyeip" {     # 별도 null_resource로 분리
  provisioner "remote-exec" {
    connection {
      host        = aws_eip.myeip.public_ip
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/keypair.pem")
    }
    inline = ["echo ${aws_eip.myeip.public_ip}"]
  }
}
```

**trigger — 강제 재실행**

```hcl
resource "null_resource" "always_run" {
  triggers = {
    always = timestamp()    # 매번 apply마다 재실행
  }
  provisioner "local-exec" {
    command = "echo 'Current time: ${timestamp()}'"
  }
}
```

**실행 결과 (2회 apply)**

```
=== 1st apply ===
null_resource.always_run (local-exec): Current time: 2026-04-05T10:46:26Z
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

=== 2nd apply ===
# null_resource.always_run must be replaced  ← timestamp() 변경됨
null_resource.always_run (local-exec): Current time: 2026-04-05T10:47:10Z
Apply complete! Resources: 1 added, 0 changed, 1 destroyed.
```

---

### terraform_data (Terraform 1.4+)

> 📖 공식 문서: https://developer.hashicorp.com/terraform/language/resources/terraform-data

`null_resource`의 대체재로, **별도 프로바이더 없이 테라폼 기본 내장** 리소스입니다.

| 비교 | null_resource | terraform_data |
|------|--------------|---------------|
| 프로바이더 필요 | ✅ (hashicorp/null) | ❌ (기본 내장) |
| 강제 재실행 인수 | `triggers` (map) | `triggers_replace` (tuple) |
| 상태 저장 인수 | ❌ | ✅ (`input` / `output`) |

```hcl
resource "terraform_data" "foo" {
  triggers_replace = [timestamp()]
  input = "hello world"
}

output "terraform_data_output" {
  value = terraform_data.foo.output   # → "hello world"
}
```

**실행 결과**

```
terraform_data.foo: Creating...
terraform_data.foo: Creation complete after 0s

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:
terraform_data_output = "hello world"
```

---

## 3.14 moved 블록

> 📖 공식 문서: https://developer.hashicorp.com/terraform/language/modules/develop/refactoring#moved-block-syntax  
> 📖 참고: [Refactor Terraform with Moved Blocks](https://lexdsolutions.com/2022/09/refactoring-terraform-code-using-moved-block-or-state-mv/)

```
[ Before ]                        [ After ]
local_file.a  ──── moved ────►  local_file.b
(State 주소)                     (State 주소)
     │                                │
     └── 실제 리소스는 그대로 ──────────┘
         (삭제/재생성 없음)
```

> 📷 참고 이미지: [Refactoring with Moved Blocks](https://developer.hashicorp.com/terraform/tutorials/configuration-language/move-config)

### 개념

리소스 이름이 변경될 때 **기존 인프라를 삭제/재생성하지 않고** State의 주소만 변경합니다.

**사용 시나리오**

- 리소스 이름 변경
- `count` → `for_each` 마이그레이션
- 리소스를 모듈로 이동

### 실습

**초기 상태**

```hcl
resource "local_file" "a" {
  content  = "Hello moved block"
  filename = "${path.module}/moved.txt"
}
```

```
local_file.a: Creation complete after 0s [id=c6589f449b7e59391108acad7ae1694a89fd2add]
```

**moved 블록 없이 이름만 변경 (a → b)**

```bash
terraform plan
```

```
# local_file.a will be destroyed
# local_file.b will be created

Plan: 1 to add, 0 to change, 1 to destroy.   ← 삭제 후 재생성!
```

**moved 블록 사용**

```hcl
resource "local_file" "b" {
  content  = "Hello moved block"
  filename = "${path.module}/moved.txt"
}

moved {
  from = local_file.a
  to   = local_file.b
}
```

```bash
terraform plan
```

```
# local_file.a has moved to local_file.b
  resource "local_file" "b" {
      id = "c6589f449b7e59391108acad7ae1694a89fd2add"  ← ID 동일!
  }

Plan: 0 to add, 0 to change, 0 to destroy.   ← 변경 없음!
```

> ID가 동일하게 유지됩니다 — 리소스가 재생성되지 않고 State 주소만 변경됩니다.

**리팩터링 완료 후 moved 블록 제거**

```hcl
# moved 블록 삭제
# moved {
#   from = local_file.a
#   to   = local_file.b
# }
```

---

## 3.15 CLI를 위한 시스템 환경 변수

> 📖 공식 문서: https://developer.hashicorp.com/terraform/cli/config/environment-variables

### 주요 환경 변수

| 변수 | 설명 | 예시 |
|------|------|------|
| `TF_LOG` | 로그 레벨 (`trace` `debug` `info` `warn` `error` `off`) | `TF_LOG=info terraform plan` |
| `TF_LOG_PATH` | 로그 파일 출력 경로 | `TF_LOG_PATH=./terraform.log` |
| `TF_LOG_CORE` | 코어 전용 로그 레벨 | `TF_LOG_CORE=debug` |
| `TF_LOG_PROVIDER` | 프로바이더 전용 로그 레벨 | `TF_LOG_PROVIDER=info` |
| `TF_INPUT` | `false`/`0` → 대화형 입력 비활성화 | `TF_INPUT=0 terraform plan` |
| `TF_VAR_<변수명>` | 입력 변수 값 오버라이드 | `TF_VAR_region=us-east-1` |
| `TF_CLI_ARGS` | 모든 서브커맨드에 인수 추가 | `TF_CLI_ARGS="-input=false"` |
| `TF_CLI_ARGS_<subcommand>` | 특정 서브커맨드에만 인수 추가 | `TF_CLI_ARGS_apply="-auto-approve"` |
| `TF_DATA_DIR` | `.terraform` 디렉터리 위치 변경 | `TF_DATA_DIR=./.terraform_tmp` |

### 사용 예시

```bash
# 로그 레벨 설정
TF_LOG=info terraform plan

# 대화형 입력 비활성화 (CI/CD 환경에서 유용)
TF_INPUT=0 terraform plan
# → Error: No value for required variable

# 변수 오버라이드
export TF_VAR_enable_file=false
terraform apply -auto-approve

# 모든 커맨드에 인수 추가
TF_CLI_ARGS="-input=false" terraform apply -auto-approve
# ↑ terraform apply -input=false -auto-approve 와 동일

# apply에만 인수 추가 (plan에는 적용 안됨)
export TF_CLI_ARGS_apply="-input=false"
terraform apply   # input=false 적용
terraform plan    # input=false 미적용

# Jenkins 등 CI 환경에서 color 제거
terraform plan -no-color

# .terraform 위치 변경
TF_DATA_DIR=./.terraform_tmp terraform plan
```

---

## 핵심 요약

| 문법 | 목적 | 핵심 포인트 |
|------|------|------------|
| `for_each` | map/set 기반 리소스 반복 | 키 기반 → 중간 삭제 안전 |
| `count` | 숫자 기반 리소스 반복 | 인덱스 기반 → 중간 삭제 위험 |
| `for` | 컬렉션 값 변환/필터링 | `[]`→tuple, `{}`→object |
| `dynamic` | 리소스 내부 블록 반복 | `each` 대신 블록명 사용 |
| 조건문 | 리소스 생성 조건 제어 | `condition ? true : false` |
| 내장 함수 | 값 변환 및 연산 | `upper/cidrsubnet/jsonencode` 등 |
| 프로비저너 | 명령 실행/파일 복사 | 최후 수단, state 미추적 |
| `null_resource` | 순서 조율, 강제 재실행 | `triggers`로 재실행 제어 |
| `terraform_data` | null_resource 대체 (1.4+) | 프로바이더 불필요, input/output 지원 |
| `moved` | State 주소 변경 | 삭제/재생성 없이 이름 변경 |

---

## 참고 자료

| 주제 | 링크 |
|------|------|
| for_each 공식 문서 | https://developer.hashicorp.com/terraform/language/meta-arguments/for_each |
| count 공식 문서 | https://developer.hashicorp.com/terraform/language/meta-arguments/count |
| count vs for_each (HashiCorp) | https://support.hashicorp.com/hc/en-us/articles/31348158569363 |
| for expression 공식 문서 | https://developer.hashicorp.com/terraform/language/expressions/for |
| dynamic block 공식 문서 | https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks |
| 조건문 공식 문서 | https://developer.hashicorp.com/terraform/language/expressions/conditionals |
| 내장 함수 공식 문서 | https://developer.hashicorp.com/terraform/language/functions |
| 프로비저너 공식 문서 | https://developer.hashicorp.com/terraform/language/resources/provisioners/syntax |
| local-exec 상세 | https://spacelift.io/blog/terraform-local-exec |
| terraform_data 공식 문서 | https://developer.hashicorp.com/terraform/language/resources/terraform-data |
| moved 블록 공식 문서 | https://developer.hashicorp.com/terraform/language/modules/develop/refactoring |
| 환경 변수 공식 문서 | https://developer.hashicorp.com/terraform/cli/config/environment-variables |
