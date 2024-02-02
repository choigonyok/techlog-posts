[ID: 35]
[Tags: TERRAFORM, AWS, KUBERNETES]
[Title: Terraform으로 EKS에 AWS Loadbalancer Controller 배포하기]
[WriteTime: 2023/11/17]
[ImageNames: d3ef3b3d-e78b-4e99-a76d-e64a3b3bbdsa.png]

## Contents

1. Preamble
2. 굳이 Loadbalancer 타입 서비스를 사용하는 이유
3. aws-load-balancer-controller란?
4. Terraform 구현

## 1. Preamble

기술 블로그 어플리케이션을 위해 kubeadm으로 구성되어있던 쿠버네티스 클러스트를 EKS로 이전하는 과정에서 Loadbalancer 타입 서비스 오브젝트를 사용하기로 했다. 그 과정에서 IRSA 설정과 aws-load-balancer-controller를 배포해야했는데, 이를 테라폼으로 구현한 예시를 찾아보기가 힘들었고 그나마 참고할만한 글들도 테라폼이 업그레이드 됨에 따라 대부분 적용되지 않는 문제가 있어 꽤나 애써서 해결한 결과물을 공유하려고한다.

## 2. 굳이 Loadbalancer 타입 서비스를 사용하는 이유


EKS처럼 클라우드 벤더에서 제공하는 쿠버네티스 서비스를 활용해 클러스터를 구축하면 Loadbalancer 타입 서비스를 생성할 때 클라우드 벤더에서 자동으로 Public IP가 할당된 클라우드 로드밸런서를 생성해서 이 서비스와 연결시켜준다.

다시 말하면, 클라우드 벤더에서 제공하는 쿠버네티스 클러스터 구성 서비스(AWS EKS 등)를 이용하지 않으면 LoadBalancer 타입 서비스를 생성해도 External-IP가 할당되지 않는다는 말과도 같다.

### 기존 블로그 아키텍처


나의 경우 최대한 운영 비용을 아낄 겸, 쿠버네티스에 대한 기초지식도 쌓을 겸 kubeadm을 활용해 쿠버네티스 클러스터를 구축해 블로그 서비스를 운영해왔다. Loadbalancer 타입 서비스를 사용하지 못하다보니, 외부에서의 접근을 가능하게 하기위해 전체 노드를 Public 서브넷에 배치하고, HAProxy를 Pod로 배포해 L7 로드밸런서로 활용하고 NodePort를 열었다.

그리고 AWS NLB를 사용해, NLB의 80포트로 listen된 트래픽을 target  group인 HAProxy가 배치된 워커노드 IP에 HAProxy 서비스 NodePort로 전달해 외부에서의 클러스터 접근을 가능하게 했다.

### 기존 방식의 단점


우선 클러스터의 외부 노출을 위해 전체 클러스터를 모두 Public 서브넷에 배치한 것은 보안적으로 좋지 않은 설계였다. 될 수 있다면 HAProxy는 Affinity 설정으로 Public 서브넷에 속한 특정 노드에서만 배포되게 하고, 나머지 Pod들은 모두 Private 서브넷에 배포되도록 하는 것이 조금 더 안전한 방법이었을 것이다.

근데 이렇게 되면 Public 서브넷에 HAProxy를 위해 배치된 노드는 오로지 HAProxy의 로드밸런싱만을 위해 사용되게 된다. 노드의 리소스를 효율적으로 사용하지 못하는 방법이라고 생각된다.

이 지점에서, Loadbalancer 타입 서비스가 사용가능해지면 많은 문제와 복잡성이 해결된다.

### Loadbalancer 타입 서비스의 장점


Loadbalancer 타입 서비스를 사용하게 되면 Loadbalancer 서비스를 포함한 클러스터 전체를 Private 서브넷에 배치할 수 있게된다. 외부에서는 Loadbalancer 타입 서비스와 연결된 AWS의 ELB가 Public 서브넷에 배치되어서 인터넷 게이트웨이를 통해 VPC로 들어온 외부 접근을 받아 Private 서브넷에 위치한 쿠버네티스 클러스터에 트래픽을 전달해줄 수 있게된다.

ALB는 L7 로드밸런서이기에 ALB를 사용하게되면 HAProxy를 사용하지 않을 수도 있다.

이 방식이 보안적으로 조금 더 바람직한 방식이라고 볼 수 있다.

## 3. aws-load-balancer-controller란?


aws-load-balancer-controller(이하 컨트롤러)는 Loadbalancer 타입 서비스를 실질적으로 구현한다. 이 컨트롤러가 Loadbalancer 타입 서비스의 생성을 감지하고 AWS API를 통해 AWS ELB를 생성하게 된다.

이게 컨트롤러에서 가능하려면 두 가지 조건이 필요하다.


1.  ELB 관련한 AWS API 요청을 보낼 수 있는 권한이 컨트롤러에게 있어야햔다.
2. 컨트롤러가 어떤 설정의 ELB를 생성할 것인지를 알아야한다.

### IRSA (IAM Role for Service Account)


1번을 만족시키기 위해 EKS에서는 IRSA을 사용할 수 있다.

EKS에서는 쿠버네티스 클러스터와 AWS IAM Role을 연결시킬 수 있는 기능을 제공한다. EKS에서 제공되는 OIDC Provider와 IAM Role, 그리고 그 IAM Role을 연결할 쿠버네티스의 ServiceAccount 리소스만 지정하면 쿠버네티스에서 ServiceAccount에 지정된 Pod들은 IAM Role에 할당된 권한을 가질 수 있게된다.

### Annotation


2번을 만족시키기 위해서는 쿠버네티스 서비스 리소스에 정의되는 annotation을 사용할 수 있다.

annotation은 메타데이터를 관리하기 위해 사용된다. label과 유사하지만 label은 관리자에 의해 사용되는 것, annotation은 그 외에 의해 사용되는 것으로 구분할 수 있다. 

예를 들어, label은 쿠버네티스 관리자가 직접 원하는 대로 키-값을 설정할 수 있고, 이 label을 통해 affinity, 서비스의 selector, 그룹화 등이 가능하다. 반대로 annotation은 키가 정해져있고, 값도 때로는 정해진 규칙 안에서 작성되어야한다.

Loadbalancer 타입으로 서비스를 생성할 때 이 annotation을 알맞게 지정해주게되면, 컨트롤러가 annotation을 확인하고 ALB 또는 NLB를 생성하고, Internal / External 설정 등을 할 수 있게된다.

## 4. Terraform 구현


### IAM Role 생


```dockerfile
module "loadbalancer_role" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name = "AWSLoadBalancerControllerIAMRole"
  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
    provider_arn               = "${module.eks.oidc_provider_arn}"
    namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}
```


우선 aws-load-balancer-controller가 AWS ELB를 생성할 수 있도록, 해당 권한을 가진 IAM Role을 생성해야한다.

AWS에서는 IAM 관련한 모듈들을 제공하고있고, IAM 모듈의 서브 모듈 중 iam-role-for-service-accounts-eks 모듈을 활용하면 쉽게 IAM Role을 테라폼으로 생성할 수 있다.

source 필드는 어떤 모듈을 쓸 지 명시하는 부분이다.

role_name은 생성될 IAM Role의 이름을 지정하는 것이고, 예시와 다른 이름을 지정해도 무관하다.

attach_load_balancer_controller_policy는 알아서 ELB 관련한 권한 정책들을 이 IAM Role에 붙여주는 설정이다. 이 필드가 없었다면 아래와 같이 따로 Policy를 생성해 Role에 부착해주어야하는데, 다행이 복잡성을 줄일 수 있다.

```dockerfile
resource "aws_iam_role_policy" "policy" {
  name = "AWSLoadBalancerControllerIAMRolePolicy"
  role = "${module.loadbalancer_role.role_name}"

  policy =  jsonencode({
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": [
          ...
        ],
        "Effect": "Allow",
        "Resource": "*"
      },
    ]
  })
}
```


### Kubernetes Provider 설정


```dockerfile
provider "kubernetes" {
  host = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    args        = ["eks", "get-token", "--cluster-name", local.cluster_name]
    command     = "aws"
  }
}
```


### Helm Provider 설정


```dockerfile
provider "helm" {
  kubernetes {
    host = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      args = ["eks", "get-token", "--cluster-name", local.cluster_name]
      command = "aws"
    }
  }
}
```


### ServiceAccount 생성 및 IAM Role 연결


```dockerfile
resource "kubernetes_service_account" "service-account" {
  metadata {
    name = "aws-load-balancer-controller"
    namespace = "kube-system"
    labels = {
        "app.kubernetes.io/name"= "aws-load-balancer-controller"
        "app.kubernetes.io/component"= "controller"
    }
    annotations = {
      "eks.amazonaws.com/role-arn" = "${module.lb_role.iam_role_arn}"
      "eks.amazonaws.com/sts-regional-endpoints" = "true"
    }
  }
}
```


### aws-load-balancer-controller 배포


```dockerfile
resource "helm_release" "nlb-pod" {
    name       = "aws-load-balancer-controller"
    repository = "https://aws.github.io/eks-charts"
    chart      = "aws-load-balancer-controller"
    namespace  = "kube-system"
    depends_on = [
        kubernetes_service_account.service-account
    ]

    set {
        name  = "serviceAccount.create"
        value = "false"
    }

    set {
        name  = "serviceAccount.name"
        value = "aws-load-balancer-controller"
    }

    set {
        name  = "clusterName"
        value = "${module.eks.cluster_name}"
    }
}
```