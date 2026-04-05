# Terraform 문법 정리 (3.9 ~ 3.15)
 
> 테라폼의 반복문, 조건문, 함수, 프로비저너, null_resource, moved 블록 등 핵심 문법 정리
 
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
# id = "..."
# content = "content a"
# filename = "./a.txt"
 
ls *.txt
# a.txt  b.txt
 
cat a.txt    # content a
cat b.txt    # content b
```
 
**terraform console 확인**
 
```bash
echo "local_file.abc" | terraform console
# 결과: map of all instances
 
echo 'local_file.abc["a"].content' | terraform console
# 결과: "content a"
```
 
**tfstate 파일 내부 구조**
 
```json
"instances": [
  {
    "index_key": "a",
    "schema_version": 0,
    ...
  },
  {
    "index_key": "b",
    "schema_version": 0,
    ...
  }
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
 
**state list 결과**
 
```
local_file.abc["a"]
local_file.abc["b"]
local_file.abc["c"]
local_file.def["a"]
local_file.def["b"]
local_file.def["c"]
```
 
> `local_file.abc`의 결과가 **map**으로 반환되므로, `local_file.def`에서도 `for_each`에 직접 전달할 수 있습니다.
 
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
type(["a","b"])           # → tuple
type(tolist(["a","b"]))  # → list(string)
type(toset(["a","b"]))   # → set(string)
 
type({a="a", b="b"})     # → object({a=string, b=string})
type(tomap({a="a", b="b"})) # → map(string)
 
var.list_set_tuple_b[0]  # list는 인덱스 접근 가능
var.map_object_a["a"]    # map은 키 접근 가능
# ------------------------------------------
```
 
---
 
## 3.9 for_each vs count 비교
 
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
 
### 중간 값 삭제 시 문제
 
| 방식 | 중간 삭제 시 |
|------|------------|
| `count` | 삭제된 인덱스 이후 **모든 리소스** 재생성 (파괴적) |
| `for_each` | 해당 **키에 해당하는 리소스만** 삭제 (안전) |
 
> 리소스의 구별되는 여러 복사본을 만들 때는 **`count` 대신 `for_each`를 사용하는 것이 바람직**합니다.
 
---
 
## 3.9 for Expressions
 
> `for_each`와 **다른** 개념으로, **복합 형식 값의 형태를 변환**하는 데 사용합니다.
 
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
  # → ["A", "B"]
}
 
output "B_index_and_value" {
  value = [for i, v in var.names : "${i} is ${v}"]
  # → ["0 is a", "1 is b"]
}
 
output "C_make_object" {
  value = { for v in var.names : v => upper(v) }
  # → { a = "A", b = "B" }
}
 
output "D_with_filter" {
  value = [for v in var.names : upper(v) if v != "a"]
  # → ["B"]
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
A_to_tuple = ["ab is member", "cd is admin", "ef is member"]
B_get_only_role = { "cd" = "admin" }
C_group = {
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
 
### 개념
 
리소스 **내부의 반복되는 속성 블록**을 동적으로 생성합니다.  
`count`/`for_each`가 리소스 전체를 반복하는 반면, `dynamic`은 **리소스 내부 블록**을 반복합니다.
 
### 사용 전/후 비교
 
**동적 블록 없이 (반복 작성)**
 
```hcl
resource "aws_security_group" "example" {
  ingress { from_port = 22  ... }
  ingress { from_port = 443 ... }
  ingress { from_port = 80  ... }
}
```
 
**dynamic 블록 사용**
 
```hcl
resource "provider_resource" "name" {
  dynamic "some_setting" {
    for_each = {
      a_key = a_value
      b_key = b_value
    }
    content {
      key = some_setting.value  # each.value 대신 블록명.value 사용
    }
  }
}
```
 
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
      filename = "${path.module}/${source.key}.txt"
    }
  }
}
```
 
**실행 결과**
 
```bash
terraform apply -auto-approve
# No changes. (동일한 결과이므로 변경 없음)
 
ls *.zip    # dotfiles.zip
unzip dotfiles.zip
ls *.txt    # a.txt  b.txt  c.txt
cat a.txt   # hello a
```
 
---
 
## 3.10 조건문
 
### 개념 — 삼항 연산자
 
```hcl
# <조건> ? <true일 때 값> : <false일 때 값>
var.a != "" ? var.a : "default-a"
```
 
> 자바스크립트/Java의 삼항 연산자와 동일한 구조입니다.
 
### 타입 일치 권장
 
```hcl
var.example ? 12 : "hello"            # ❌ 비권장 (타입 불일치)
var.example ? "12" : "hello"          # ✅ 권장
var.example ? tostring(12) : "hello"  # ✅ 권장
```
 
### 실습 — count와 결합한 리소스 생성 조건
 
```hcl
variable "enable_file" {
  default = true
}
 
resource "local_file" "foo" {
  count    = var.enable_file ? 1 : 0    # true면 생성, false면 미생성
  content  = "foo!"
  filename = "${path.module}/foo.bar"
}
 
output "content" {
  value = var.enable_file ? local_file.foo[0].content : ""
}
```
 
**false 환경 변수 설정 후 실행**
 
```bash
export TF_VAR_enable_file=false
terraform apply -auto-approve
 
# state list 결과
# (아무것도 없음 — 리소스 미생성)
 
echo "var.enable_file ? 1 : 0" | terraform console
# 결과: 0
```
 
**true(기본값)로 실행**
 
```bash
unset TF_VAR_enable_file
terraform apply -auto-approve
 
# state list 결과
# local_file.foo[0]
 
echo "local_file.foo[0].content" | terraform console
# 결과: "foo!"
```
 
---
 
## 3.11 함수
 
### 개념
 
테라폼에는 값의 유형을 변경하거나 조합하는 **내장 함수**가 제공됩니다.  
단, **사용자 정의 함수는 지원하지 않습니다.**
 
### 함수 카테고리
 
| 카테고리 | 예시 함수 |
|---------|---------|
| 문자열 | `upper()`, `lower()`, `title()`, `replace()` |
| 숫자 | `max()`, `min()`, `abs()` |
| 컬렉션 | `toset()`, `tolist()`, `tomap()`, `length()` |
| 인코딩 | `jsonencode()`, `jsondecode()`, `base64encode()` |
| IP 네트워크 | `cidrsubnet()`, `cidrnetmask()`, `cidrsubnets()` |
| 파일 시스템 | `file()`, `filebase64()` |
| 날짜/시간 | `timestamp()`, `formatdate()` |
| 타입 변환 | `tostring()`, `tonumber()`, `tobool()` |
 
### 실습
 
```hcl
resource "local_file" "foo" {
  content  = upper("foo! bar!")   # → "FOO! BAR!"
  filename = "${path.module}/foo.bar"
}
```
 
**terraform console 확인**
 
```bash
terraform console
# ------------------------------------------
upper("foo!")              # → "FOO!"
max(5, 12, 9)             # → 12
lower(local_file.foo.content)  # → "foo! bar!"
 
# IP 관련 함수
cidrnetmask("172.16.0.0/12")   # → "255.240.0.0"
cidrsubnet("1.1.1.0/24", 1, 0) # → "1.1.1.0/25"
cidrsubnet("1.1.1.0/24", 1, 1) # → "1.1.1.128/25"
cidrsubnets("10.1.0.0/16", 4, 4, 8, 4)
# → ["10.1.0.0/20", "10.1.16.0/20", "10.1.32.0/24", "10.1.33.0/20"]
# ------------------------------------------
```
 
---
 
## 3.12 프로비저너
 
### 개념
 
> ⚠️ **Provisioners are a Last Resort** — 프로비저너는 최후의 수단으로만 사용
 
- 프로바이더로 처리할 수 없는 **명령 실행, 파일 복사** 등을 담당
- **프로비저너 실행 결과는 tfstate에 저장되지 않음** → 선언적 보장 안됨
- 가능하면 `userdata`, `cloud-init`, `Packer` 등 대안 사용 권장
 
### 프로비저너 종류
 
| 종류 | 설명 |
|------|------|
| `local-exec` | 테라폼 실행 환경(로컬)에서 명령 실행 |
| `remote-exec` | 원격 서버에서 명령 실행 (SSH/WinRM) |
| `file` | 로컬 → 원격 파일/디렉터리 복사 |
 
### local-exec
 
```hcl
resource "local_file" "foo" {
  content  = upper(var.sensitive_content)
  filename = "${path.module}/foo.bar"
 
  # 생성 시 실행
  provisioner "local-exec" {
    command = "echo The content is ${self.content}"
  }
 
  # 실패해도 계속 진행
  provisioner "local-exec" {
    command    = "abc"
    on_failure = continue
  }
 
  # 삭제 시 실행
  provisioner "local-exec" {
    when    = destroy
    command = "echo The deleting filename is ${self.filename}"
  }
}
```
 
**실행 결과**
 
```
local_file.foo: Provisioning with 'local-exec'...
local_file.foo (local-exec): The content is SECRET
local_file.foo (local-exec): /bin/sh: abc: command not found  ← on_failure=continue로 무시
 
# destroy 시
local_file.foo (local-exec): The deleting filename is ./foo.bar
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
 
### local-exec 상세 인수
 
```hcl
resource "null_resource" "example1" {
  provisioner "local-exec" {
    command = <<EOF
      echo Hello!! > file.txt
      echo $ENV >> file.txt
      EOF
 
    interpreter = ["bash", "-c"]    # 실행 인터프리터
    working_dir = "/tmp"            # 실행 디렉터리
    environment = {
      ENV = "world!!"              # 환경변수
    }
  }
}
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
 
### null_resource
 
**아무 작업도 수행하지 않는 가상 리소스**로, 프로비저닝 순서 조율이나 외부 명령 실행에 활용합니다.
 
**주요 시나리오**
 
- 프로비저닝 과정에서 명령어 실행
- 모듈, 반복문, 데이터 소스와 함께 사용
- 리소스 간 **상호 참조(Cycle)** 문제 해소
 
**Cycle 에러 예시**
 
```hcl
# ❌ aws_eip가 aws_instance를, aws_instance가 aws_eip를 참조 → Cycle 에러
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
 
**null_resource로 해결**
 
```hcl
resource "aws_instance" "example" { ... }  # eip 참조 제거
 
resource "aws_eip" "myeip" {
  instance = aws_instance.example.id
}
 
resource "null_resource" "echomyeip" {       # 별도 null_resource로 분리
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
resource "null_resource" "foo" {
  triggers = {
    ec2_id = aws_instance.bar.id    # id 변경 시 재실행
  }
}
 
resource "null_resource" "bar" {
  triggers = {
    always = timestamp()            # 매번 재실행
  }
}
```
 
---
 
### terraform_data (Terraform 1.4+)
 
`null_resource`의 대체재로, **별도 프로바이더 없이 테라폼 기본 내장** 리소스입니다.
 
| 비교 | null_resource | terraform_data |
|------|--------------|---------------|
| 프로바이더 필요 | ✅ (hashicorp/null) | ❌ (기본 내장) |
| 강제 재실행 인수 | `triggers` (map) | `triggers_replace` (tuple) |
| 상태 저장 | ❌ | ✅ (`input` / `output`) |
 
```hcl
resource "terraform_data" "bootstrap" {
  triggers_replace = [
    aws_instance.web.id,
    aws_instance.database.id
  ]
 
  provisioner "local-exec" {
    command = "bootstrap-hosts.sh"
  }
}
 
resource "terraform_data" "foo" {
  triggers_replace = [aws_instance.foo.id]
  input = "world"
}
 
output "terraform_data_output" {
  value = terraform_data.foo.output   # → "world"
}
```
 
---
 
## 3.14 moved 블록
 
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
  content  = "foo!"
  filename = "${path.module}/foo.bar"
}
```
 
```bash
terraform apply -auto-approve
echo "local_file.a.id" | terraform console
# → "4bf3e335199107182c6f7638efaad377acc7f452"
```
 
**이름 변경만 하면? (a → b)**
 
```hcl
resource "local_file" "b" { ... }
```
 
```bash
terraform plan
# Plan: 1 to add, 0 to change, 1 to destroy.   ← 삭제 후 재생성!
```
 
**moved 블록 사용**
 
```hcl
resource "local_file" "b" {
  content  = "foo!"
  filename = "${path.module}/foo.bar"
}
 
moved {
  from = local_file.a
  to   = local_file.b
}
```
 
```bash
terraform plan
# local_file.a has moved to local_file.b
# Plan: 0 to add, 0 to change, 0 to destroy.   ← 변경 없음!
 
terraform apply -auto-approve
echo "local_file.b.id" | terraform console
# → "4bf3e335199107182c6f7638efaad377acc7f452"  (ID 동일 유지!)
```
 
**리팩터링 완료 후 moved 블록 제거**
 
```hcl
# moved 블록 삭제 (주석 처리 또는 제거)
# moved {
#   from = local_file.a
#   to   = local_file.b
# }
```
 
---
 
## 3.15 CLI를 위한 시스템 환경 변수
 
### 주요 환경 변수
 
| 변수 | 설명 |
|------|------|
| `TF_LOG` | 로그 레벨 설정 (`trace`, `debug`, `info`, `warn`, `error`, `off`) |
| `TF_LOG_PATH` | 로그 파일 출력 경로 |
| `TF_LOG_CORE` | 테라폼 코어 전용 로그 레벨 |
| `TF_LOG_PROVIDER` | 프로바이더 전용 로그 레벨 |
| `TF_INPUT` | `false`/`0`으로 설정 시 대화형 입력 비활성화 |
| `TF_VAR_<변수명>` | 입력 변수 값 오버라이드 |
| `TF_CLI_ARGS` | 모든 서브커맨드에 인수 추가 |
| `TF_CLI_ARGS_<subcommand>` | 특정 서브커맨드에만 인수 추가 |
| `TF_DATA_DIR` | `.terraform` 디렉터리 위치 변경 |
 
### 사용 예시
 
```bash
# 로그 레벨 설정
TF_LOG=info terraform plan
 
# 대화형 입력 비활성화 (CI/CD 환경)
TF_INPUT=0 terraform plan
# → Error: No value for required variable
 
# 변수 오버라이드
export TF_VAR_enable_file=false
terraform apply -auto-approve
 
# 인수 추가
TF_CLI_ARGS="-input=false" terraform apply -auto-approve
# ↑ terraform apply -input=false -auto-approve 와 동일
 
# apply에만 인수 추가
export TF_CLI_ARGS_apply="-input=false"
terraform apply   # input=false 적용
terraform plan    # input=false 미적용
 
# .terraform 위치 변경
TF_DATA_DIR=./.terraform_tmp terraform plan
```
 
---
 
## 핵심 요약
 
```
for_each   → map/set 기반 반복, 키로 리소스 식별 (count보다 안전)
for        → 컬렉션 값 변환/필터링 ([]→tuple, {}→object)
dynamic    → 리소스 내부 블록을 동적으로 반복 생성
조건문      → 삼항 연산자 (condition ? true_val : false_val)
내장 함수   → upper/lower/cidrsubnet/jsonencode 등
프로비저너  → 최후 수단, local-exec/remote-exec/file
null_resource / terraform_data → 순서 조율, 강제 재실행
moved      → 리소스 이름 변경 시 삭제/재생성 없이 State 주소 변경
```
 
