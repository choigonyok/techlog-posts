[Tags: INFRA]
[Title: 기본적인 Terraform 사용법]
[WriteTime: 2023/11/07]
[ImageNames: ]

## Tags


TERRAFORM

## Contents

1. Preamble
2. Terraform이란?
3. Basic Terraform
4. Summary

## 1. Preamble


Terraform에 대해 어느 정도 이해하고 있다고 생각했다. 내 자만이었다. Terraform은 내 생각보다 훨씬 더 깊고 넓은 도구였다. Terraform에 대해 알고있던 기본 지식을 간단히 정리하고, 앞으로 Terraform에 대한 지식을 꾸준히 고도화해보려한다.

## 2. Terraform이란? 


Terraform은 **인프라스트럭처 프로비저닝 도구**이다. 공식 홈페이지에선 Terraform을 **Automate infrastructure on any cloud with Terraform** 라고 소개한다. 어떤 클라우드든 Terraform을 통해 인프라를 자동화할 수 있다는 것이다. 

provisioning의 사전적 정의는 공급이다. Ansible의 벤더인 RedHat에서는 Cloud Provisioning을 다음과 같이 정의한다.
> Cloud provisioning includes creating the underlying infrastructure for an organization’s cloud environment, like installing networking elements, services, and more.


클라우드 환경을 위한 인프라 구성을 high level에서 도와주는 툴이 infrastructure provisioning tool이라고 할 수 있겠다.

terraform은 코드로 인프라를 구축할 수 있게 해주는 IaC(Infrastructure as Code)의 대표주자이다. 사용하는 IDE에서 .tf extension의 파일을 생성하면 Terraform을 구성할 수 있다.

## 3. Basic Terraform


### Provider


Terraform엔 provider라는 개념이 있다. provider는 리소스를 제공하는 업체이다. 유명한 클라우드 컴퓨팅 서비스인 AWS, GCP, Azure부터 우리나라의 네이버클라우드도 official provider로 등록되어있다.

Terraform로 인프라 HCL 코드를 짜면 Terraform이 코드를 바탕으로 해당 provider에 API요청을 보내 인프라가 실질적으로 구성되는 방식이다. 참고로 HCL은 HashiCorp Configuration Language의 준말로, 그냥 Hashicorp사에서 개발한 언어라고 보면 된다.

Terraform official homepage의 registry에 들어가보면 수많은 provider를 확인해볼 수 있다. provider는 Official, Partner, Community로 나뉘어진다.

Official은 Terraform에서 직접 관리하는 provider, Partner는 해당 파트너가 직접 관리하는 provider, Community는 개인이나 단체 등이 관리하는 provider를 의미한다.

provider를 tf파일에 선언하는 문법은 다음과 같다.

```dockerfile
provider "PROVIDER NAME" {
...
}
```


만약 provider가 Official provider가 아닌 Partner/Community provider라면 위 문법은 적용되지 않는다.

```dockerfile
terraform {
  required_provider "PROVIDER NAME" {
  ...
  }
}
```


으로 선언해주어야 한다.

### Process


기본적으로 Terraform은 init -> plan -> apply 의 순서로 작업이 이루어진다. tf파일에 provider를 지정하고,

```dockerfile
terraform init
```


을 입력하면 Terraform은 .terraform 디렉토리를 생성해서 모든 tf파일의 provider에 인프라 구성 작업을 실행하기 위한 플러그인들을 다운로드한다.

```dockerfile
terraform plan
```


을 입력하면 지금 사용자가 짠 HCL 코드에 이상이 없는지, 실행되면 어떤 리소스가 생성/수정/삭제되는지에 대한 개요를 보여준다. 내용을 확인해보고 그대로 적용하기를 원하면

```dockerfile
terraform apply
```


으로 코드를 적용하게 되고, 실질적인 리소스 생성/수정/삭제가 이루어진다. 위의 예시는 기본적인 사용사례이고, 실제로는

```dockerfile
terraform plan -out FILENAME
```


으로 plan의 내용을 특정 파일로 저장한 후에

```dockerfile
terraform apply FILENAME
```


으로 plan으로 구성된 내용이 적용되도록 하는 방식이 더 업무상 효율적인 것 같다.

apply로 생성된 인프라를 삭제하는 방식은 두 가지가 있다.


1. 코드 수정 후 apply

terraform은 리소스의 상태를 **현상태**로 유지한다. 예를 들어 AWS의 EC2 instance를 하나 생성하는 tf 코드를 작성했다고 가정하자. 이 상황에서 terraform apply를 3번 반복하면 어떻게 될까? 인스턴스가 3개 생성될까? 답은 **그렇지 않다.** 앞서 언급한 것처럼 terraform은 리소스의 상태를 코드의 현상태(Desired State)로 유지하기 때문이다. 

그렇기에 이미 해당 코드로 인스턴스가 하나 생성되어있는 상태에선 다시 apply 한다고 해서 추가적으로 인스턴스가 생성되진 않는다. 추가적으로 인스턴스를 생성하려면 리소스 블록을 하나 더 추가해야한다.

그럼 인스턴스를 생성하는 리소스 코드 블럭을 삭제하고 apply를 한다면, terraform은 현상태를 적용하기 위해 생성되어있던 인스턴스를 destroy할 것이다.


2. terraform destroy

1번 방식처럼 하게되면 두 가지 문제가 발생한다. 첫쨰로 매번 코드를 일일이 수정해야한다는 점과, 둘쨰로 삭제된 코드를 되돌리려면(인스턴스를 재생성하려면) 추가적인 비용이 든다는 것이다.

Terraform은 destroy command를 지원한다. 생성된 리소스들을 destroy하게 해주는 명령어이다.

```dockerfile
terraform destroy
```


이 커맨드는 현재 디렉토리의 모든 tf파일에 선언된 리소스들을 삭제시킨다. 만약 한 디렉토리에 여러 tf파일과 리소스들이 선언되어있고, 그 중 특정 리소스만 destroy하고 싶다면 어떻게 해야할까?

```dockerfile
terraform destroy -target RESOURCE NAME.LOCAL NAME
```


이 명령어를 통해 원하는 리소스만 파괴시킬 수 있다. resource name과 local name에 대해 간략히 먼저 알아보기 위해 예시로 AWS의 EC2 인스턴스를 생성하는 리소스 블럭을 가정한다.

```dockerfile
resource "aws_instance" "chat-service" {
  ami = "ami-0c9c942bd7bf113a2"
  instance_type = "t3.micro"
  tags = {
  Name = "ChatService"
  }
}
```


리소스 블럭의 "aws_instance"가 바로 resource name이다. provider로 aws를 선언해두었기 때문에, instance를 생성하려면 미리 정의된 "aws_instance"를 이용해야한다. 

Provider 항목에서 설명한 것처럼, AWS는 Official provider이고, 리소스를 제공하는 업체이다. 즉, Terraform에서 자체적으로 AWS API와 연계되는 기능을 구현한 것이 아니고, Terraform에서 믿고 AWS가 알아서 Terraform resource를 관리하도록 권한을 넘겨준 것이다. 그래서 AWS가 정의한 이름대로 EC2 인스턴스를 생성하기 위해서는 resource name으로 "aws_instance"를 사용해야한다.

이어 나오는 "chat-service"는 local name이다. terraform 내부적으로 리소스들을 구별하기 위한 id이다.

다른 리소스들은 두고 이 리소스만 파괴하고 싶다면,

```dockerfile
terraform destroy -target aws_instance.chat-service
```


이런 식으로 target flag를 이용해 원하는 리소스를 파괴시킬 수 있다.

### Current state / Desired state


Terraform은 tf파일을 실행할 때마다 반복해서 리소스를 생성하지 않고 Desired state와 Current state를 일치시킨다고 말했다. 이 기능은 Terraform state를 통해 가능하다.

Terraform apply를 처음 하게되면 디렉토리에 .tfstate 파일이 하나 생성된다. 이 파일 내부엔 현재 terraform이 알고있는 리소스의 상태(Current state)가 기록되어있다.

"terraform destroy"를 실행한 직후라면 .tfstate 파일은 비어있을 것이고, 리소스가 생성되었다면 생성된 리소스에 대한 정보가 담겨있게된다. 이렇게, Terraform은 자체적으로 리소스 상태 변화를 기록하고있기 때문에, 반복해서 apply를 실행해도 
>  "아, tfstate파일을 보니까 이 리소스는 이미 생성되어있구나. 또 생성할 필요 없겠네"


하고 추가적으로 생성하지 않는 것이다. 그래서 이 tfstate파일을 함부로 수정하지 않는 것이 좋다. 건들다 리소스 정보를 지워버리면 Terraform은 리소스가 이미 생성되어있다는 것을 알 수 없으니 동일한 또 리소스를 생성하게 되고, Terraform을 믿고 직접 리소스 현황을 체크하지 않고있던 관리자는 사용되지 않은채 생성되어있는 버려진 리소스의 비용을 지불해야만 하는 상황이 생길 수 있다.

만약 Terraform으로 EC2 인스턴스를 생성했는데, 관리자가 콘솔에서 그 인스턴스의 이름을 변경했다고 가정하자. Terraform state에는 이름이 변경되었다는 기록이 없기 때문에 Terraform은 destory할 때 이름이 변경된 인스턴스를 삭제하지 못하게된다.

그럼 manually 수정은 금지이고 항상 Terraform으로만 모든 리소스를 관리해야하는 것일까? 그렇지 않다.

```dockerfile
terraform refresh
```


이 커맨드를 사용해서 .tfstate에 기록되어있는 current state를 최신화할 수 있다. 이 refresh는 "terraform plan" 커맨드에 내부적으로 포함되어있기 때문에 따로 이 커맨드를 사용할 일을 많지 않다고 한다.

### Version management


Terraform에선 provider의 버전을 관리할 수 있다. 일반적으로 versions.tf 파일에 따로 버전을 관리한다. 기본적으로 버전을 명시하지 않고 apply를 실행하면 가장 최신 버전으로 실행된다. 그러나 중간에 새로운 provider 버전이 업데이트 되는 경우 어떻게 해야할까?

처음 "terraform init" 커맨드로 provider의 리소스 제공을 위한 의존성 플러그인들을 설치하면 lock 파일이 생성된다.

해당 lock 파일 내부에는 provider의 버전이 명시되어있다. 만약 새로운 버전의 provider를 사용하고 싶다면 lock파일을 지우고 다시 terraform init을 실행해 최신버전의 prodiver plugin을 설치할 수도 있고, 아니면

```dockerfile
terraform init -upgrade
```


를 실행하면 lock파일을 지우지 않아도 lock파일의 버전 내용이 최신 버전으로 업데이트 된다.

운영 도중에 최신버전의 provider를 도입하는 것은 더 발전된 기술들을 적용하는 장점이 있을 수 있지만, 반대로 버전을 변경하면서 오늘 호환성 문제가 생길 수도 있다. 그렇기 때문에 새로운 버전을 도입할 떄는 테스트를 잘 거친 후에 버전 업데이트를 하는 것이 좋다.

### 의존성


Terraform이 강력한 도구로 평가받은 이유 중 하나는 Terraform이 리소스 간 의존성을 자체적으로 관리해준다는 것이다.

Terraform에서 다른 리소스 데이터를 참조할 때는 .(dot)을 이용한다.

```dockerfile
role = "${aws_iam_role.node.name}"
```


이 코드는 리소스 블럭의 role을 aws_iam이라는 resource name + node라는 local name을 가진 리소스의 메타데이터 중 name로 정의하겠다는 것을 의미한다.

그럼 의존성이 생기게된다. aws_iam_role.node 리소스가 먼저 생성이 되어야 이 리소스에서 aws_iam_role.node 리소스의 name을 정상적으로 참조할 수 있을 것이다.

Terraform은 자체적으로 
>  "아 이걸 참조하려면 저 리소스가 먼저 생성된 후에 이 리소스를 생성해야하는구나"


판단하게 되고, 작업을 오류없이 동기적으로 실행할 수 있게 된다.

### Output


Terraform은 apply를 통해 생성된 다양한 리소스들에 대한 결과를 output 블럭을 통해 콘솔에 출력할 수 있다.

```dockerfile
output "cluster_name" {
  description = "Kubernetes Cluster Name"
  value       = module.eks.cluster_name
}
```


만약 이 기능이 없다면, 생성한 EC2 인스턴스에 SSH 접속을 하기위해 Public IP를 필요로할 때마다 AWS 콘솔에 직접 들어가서 일일이 IP를 확인해야할 것이다.

Output을 통한 출력값은 단순히 콘솔 출력용 뿐만 아니라, 이 Output을 통해 다른 프로젝트의 tf파일에서 값을 참조해서 사용할 수 있다. 특히 스크립트로 자동화를 할 때 Output 값을 참조해서 사용할 수 있어 아주 유용한 것 같다. 예를 들어 위와 같은 Output이 생성되었다면

```dockerfile
#!bin/bash
echo "$(terraform output cluster_name)"
```


 이런 식으로 스크립트를 실행시키면 value값이 출력되게 된다.

### Variables


variable 블록을 이요해서 Terraform에서 반복적으로 사용되는 값을 쉽게 수정해 Terraform 코드의 재사용성을 높일 수 있다.

```dockerfile
variable "region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-2"
}
```


이런 식으로 변수를 설정하고,

```dockerfile
provider "aws" {
  region = var.region
}
```


이런 식으로 변수 값을 참조할 수 있게된다.

variable 블록만 선언하고 참조도 했는데, default 필드가 지정되어있지 않다면, 아래 이미지처럼 apply를 할 때 콘솔에서 해당 variable의 값을 관리자에게 입력받게 된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/F6E108F5-4923-400D-B04E-DDF31B53343D/F6C66B24-781C-4217-AD91-826A424E36A1_2/NVTNRAQzzYylg44WeyylsgxjyQyLTY0R2sBHRsyuSnkz/Image.png)

또는 아래 커맨드처럼도 변수 적용이 가능하다.

```dockerfile
terraform plan -var="VALUE"
```


variable 블럭 외에도 locals 블럭을 통해 해당 tf 파일 내부에서만 사용되는 변수를 생성할 수 있다.

variable과는 범위 말고도 약간 다른 부분이 variable은 default를 정의해두었더라도 -var 플래그를 사용해서 value를 덮어씌울 수가 있는데, locals 블럭에서는 그게 불가능하다.

```dockerfile
locals {
  cluster_name = "cluster"
}
```


이렇게 locals 블록을 선언할 수 있고, 변수의 사용은 locals.cluster_name과 같이 한다.

### 조건문


조건문도 사용이 가능하다. 자바스크립트의 문법과 동일하다.

```dockerfile
ami_type = var.ami_id != "" ? null : var.ami_type
```


### 함수


다른 프로그래밍 언어처럼 사용자 정의 함수를 생성할 수는 없다. 그러나 Terraform에서 내부적으로 정의해둔 몇몇 함수를 사용할 수 있다.

terraform console command로 함수가 어떻게 동작할 지 확인해볼 수 있음

### data 타입


ami같은 경우는 region에 따라 같은 ami(우분투, 윈도우 등)이더라도 ami id가 달라진다.  이를 해결하기 위해서 ami id를 하드코딩하는 게 아니라 참조하는 형식으로 구현하면 코드 재사용성을 높일 수 있다.

```dockerfile
data "aws_ami" "ami_test" {
  most_recent = true // 가장 최신 버전 ami를 사용하겠다는 뜻
  owners = [amazon] // 아마존 ami를 사용하겠다는 뜻
  filter {
    name = "name"
    values = ["amzn2-ami-hvm*"] // 아마존ami중에서도 뭘 선택할건지 리눅스 기반 등등 선택하는 것
  }
} 
```


### 반복문


Terraform에서는 dynamic 블록을 사용해서 반복문을 구현할 수 있다.  dynamic 블록 내부에서 "for_each"로 특정 variable의 element를 돌면서 반복문을 동적으로 실행할 수 있다.

```dockerfile
dynamic "ingress" {
  for_each = var.VARIABLE_NAME
  contents {
    port = ingress.value
  }
}
```


기본적으로 iterator의 이름은 dynamic 블럭의 이름과 동일하게 (위 예시 코드에서는 "ingress") 설정되지만, iterator의 이름을 특정하고 싶다면, 

```dockerfile
iterator = ""
```


iterator 필드를 명시해주어서 iterator 이름을 변경할 수 있다.

terraform 파일의 코드를 포맷팅하고 싶으면 terrform fmt command

terraform validate command

해당 리소스들이 가지고있는 속성들이 유효한 속성들인지 확인할 수 있음

aws_instance 리소스에 유효한 속성은 ami, instance_type 등인데

그 외 정의되지 않은 속성을 작성했다면 validate command로 오류를 확인 가능

물론 terraform plan에는 유효성 검사가 내장되어있음 refresh처럼!

### Current state 초기화


만약 manually 리소스를 수정했는데 변경사항을 다 기록해두지 못해서 마구잡이로 리소스 상태 관리가 안되는 상황이라면,

```dockerfile
terraform apply -replace "."
```


커맨드를 실행해서 현재 provider에 있는 모든 리소스를 다 삭제한 뒤 tf 파일의 desired state 로 리소스를 생성할 수 있다. "*"는 Wildcard로, 전체를 의미한다.

### 의존성 시각화


앞서 Terraform은 리소스 간 의존성을 알아서 관리한다고 했다. 리소스 간의 의존성을 그래프 형식으로 확인할 수 있게 해주는 기능을 지원하는데, 

```dockerfile
terraform graph > FILENAME.dot
```


커맨드를 통해 Terraform으로 생성한 리소스 간의 의존성을 그래프 형태로 볼 수 있는 파일을 생성할 수 있다. 이 파일은 graphviz 등의 도구를 통해 변환하면 이미지 파일로 확인이 가능하다.

### remote-exec / local-exec / when / taint


Terraform에서는 리소스를 생성한 이후에 구성 관리를 할 수 있도록 remote-exec, local-exec 기능을 제공한다.

- ### remote-exec


remote-exec은 생성한 리소스에서 적용시킬 액션을 정의할 수 있다. 예를 들어 EC2 인스턴스를 생성한 후 EC2 인스턴스에서 echo 커맨드를 이용해 문자열을 출력시키고 싶다면, EC2 인스턴스 리소스 내부에서

```dockerfile
  provisioner "remote-exec" {
    inline = [
      "echo 'HELLO, WORLD'",
    ]
  }
```


이런 식으로 HCL 코드를 구성해 커맨드를 실행시킬 수 있다.

- ### local-exec


local-exec은 리소스를 생성한 이후 Terraform을 실행시킨 호스트 머신에서 특정 액션을 자동화할 때 사용한다. 예로, EC2 인스턴스를 생성한 후 로컬에서 echo 커맨드를 사용해 문자열을 출력시키고 싶다면, EC2 인스턴스 리소스 내부에서

```dockerfile
  provisioner "local-exec" {
    inline = [
      "echo 'HELLO, WORLD'",
    ]
  }
```


이렇게 HCL을 구성하면, 리소스 생성 이후 로컬에 "HELLO, WORLD"가 출력되게 된다.

- ### when


이 local-exec / remote-exec는 따로 when 필드를 지정해주지 않으면 default는 creation time에 실행된다. 만약 리소스가 삭제될 때 특정 작업을 실행시키고 싶다면, when을 사용하면 된다.

```dockerfile
  provisioner "local-exec" {
    when = destroy
    inline = [
      "echo 'resource is removing'",
    ]
  }
```


- ### taint


creation time provisioner의 특징은 만약 리소스 생성시 오류로 해당 provisioner가 실행되지 못했다면 그 리소스는 taint된 것으로 표시된다는 것

taint된 리소스는 다음에 동일한 terraform apply시에 자동으로 파괴되고 재생성된다

taint된 리소스는 tfstate파일의 state의 값으로 "tainted"를 가진다

provisioner 블럭 안에 on_failure 필드를 continue로 선언하면, 해당 리소스가 생성에 실패해도 다른 리소스 생성을 이어갈 수 있다.

fail은 default로 provisioner가 실패하면 taint되면서 작업이 중지되고, 만약 continue로 설정해두면 오류가 생겨도 taint되지도 않고 작업이 중지되지도 않는다

### null / file resource


처음에 local-exec을 통해 로컬의 pem 파일을 마스터 노드에 scp로 복사하려고 했으나, Terraform에서 connection 블록을 사용한 원격 연결이 아닌 커맨드 + pem키를 활용한 연결은 금지되어있다. 따라서 null resource를 통해 파일을 리소스에 복사해주어야한다.

```dockerfile
resource "null_resource" "pemkey" {
  connection {
    host     = "${aws_instance.master_node.public_ip}"
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/pemkey.pem")
  }

  provisioner "file" {
    source      = "~/pemkey.pem"    
    destination = "pemkey.pem"
  }
}
```


null resource를 사용해서 Terraform으로 생성한 리소스와 ssh 연결을 하고, file 타입 provisoner를 사용해 해당 파일을 리소스로 복사해줄 수 있다.

### Module


모듈은 테라폼에서 중요하고 또 많이 사용되는 블록이다. module 블록을 통해 Terraform 코드의 재사용성을 높여 여러 프로젝트에서 참조하여 리소스를 생성할 수 있다.

```dockerfile
module "LOCAL_NAME" {
  source = "RELATIVE_PATH"
}
```


이런 식으로 작성하면 source 경로에 있는 tf 파일의 리소스를 참조해서 사용할 수 있다. 일반적으로 직접 리소스를 정의하는 방식보다 각 Provider에서 제공하는 모듈을 활용해 리소스를 생성하는 방식이 조금 더 일반적인 것 같다.

module에서 참조하는 source는 Terraform repository에 존재하는 공식 모듈 뿐만 아니라, 직접 공유저장소(깃허브 등)에 보관하고있는 tf 파일도 참조가 가능하다.

### Workspace


마치 Kubernetes에서 namespace로 논리적인 분리를 하는 것처럼, Terraform도 유사한 기능을 지원한다. workspace를 분리하면 workspace마다 생성한 리소스를 별도로 관리하게되며, dev/stage/prod처럼 서버(리소스)를 분리해서 운영할 수 있다.

### Backend


Terraform을 활용해서 리소스를 생성하려면 당연히 인증/인가를 필요로한다. AWS의 경우 AWS 계정과 비밀번호를 통해 인증/인가를 할 수 있다. 이 코드가 공유저장소에 올라간다면 보안적으로 큰 위협이 될 것이다. 그래서 .gitignore에 해당 tf 파일을 명시해서 공유저장소에 크리덴셜이 노출되는 것을 막을 수 있는데, 문제가 하나 생긴다.

크리덴셜을 공유저장소에 올리지 않으면 협업하는 팀 안에서 Terraform이 공유되지 않는데 어떻게 해야할까?

Terraform 중앙 백엔드에 tfstate파일을 저장하는 방식을 사용해야한다. 중앙 백엔드는 s3 bucket이 될수도, 혹은 다른 스토리지가 될 수도 있다. 관리자가 직접 팀의 상황에 맞게 구성해야한다.

이를 통해서 terraform을 이용한 협업을 할 때, 공유저장소에서 tf파일들을 가져오고, 작업할 때 중앙 백엔드의 tfstate파일을 참조/수정하며 협업이 가능하도록 구성할 수 있다.

```dockerfile
terrform {
  backend "" {
    ...
  }
}
```


근데 백엔드로 사용하는 서비스에 따라 Terraform이 Locking을 지원하지 않을 수 있다.

### Locking


Terraform이 리소스를 생성(apply) 또는 삭제(destroy)할 때, 자체적으로 lock 파일을 생성한다. 그리고 작업이 완료되면 lock파일을 삭제시킨다. 이 lock 파일은 mutex lock처럼 동작해서, terraform이 작업하는 도중 다른 terraform 작업이 실행되어서 race condition이 생기는 것을 막기위한 기능이다.

백엔드에서 locking이 지원되지 않으면 한명이 Terraform이 작업을 수행하고 백엔드의 tfstate가 수정중인 상황에서 다른 Terraform 프로세스가 끼어들어서 tfstate가 엉킬 수 있게되고, 두 Terraform 프로세스 모두 정상적인 결과를 얻지 못할 수 있다.

그래서 크리덴셜 외에 또 별도로 tfstate만을 위한, lock을 지원하는 백엔드를 구성하는 것이 좋다. 데이터베이스 등을 이용하면 효과적이다. 방식은 Terraform이 먼저 작업중일때 데이터베이스 테이블에 레코드를 추가하고, 끝나면 삭제하는 방식으로 구성할 수 있다.

만약 중간에 다른 Terraform이 실행되면 데이터베이스 테이블에 레코드가 존재하는 것을 확인하고, 레코드가 삭제된 후 프로세스가 실행되도록 구성해 Lock을 구현할 수 있다.

## 4. Summary


Terraform은 보통 전체 프로젝트의 Terraform 코드가 하나의 레포지토리에서 통합되어 관리된다. Module을 통한 재사용성을 높히기 때문이고, 그만큼 가독성과 올바른 레포지토리 관리를 위해 Terraform도 하나의 프로그래밍 언어처럼 PR과 코드리뷰를 진행한다는 데브옵스 팀의 기술 블로그도 읽은 적이 있다.

Terraform도 깊게 파면 배울 것이 참 많은 도구인 것 같고, 그만큼 유용하게 사용되는 도구인 것 같다.

기본적인 Terraform 문법과 개념에 대해 알아보았고, 관련해서 AWS EKS를 Terraform으로 프로비저닝하도록 도와주는 eks module을 코드리뷰하며 EKS와 테라폼에 대해 더 깊게 학습해보려고 한다. 
