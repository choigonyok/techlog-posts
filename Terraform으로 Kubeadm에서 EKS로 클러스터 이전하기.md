[ID: 34]
[Tags: INFRA KUBERNETES CLOUD]
[Title: Terraform으로 Kubeadm에서 EKS로 클러스터 이전하기]
[WriteTime: 2023/11/12]
[ImageNames: ]

## Tags

INFRA KUBERNETES TERRAFORM

1. ## Preamble


**블로그에 젠킨스 CI/CD 파이프라인 구성하기 시리즈**


- (현재 글) Terraform으로 BareMetal K8S 클러스터를 EKS로 이전하기
- EKS IRSA + CSI로 AWS EBS를 Pod의 PV로 사용하기
- 젠킨스 + ArgoCD CI/CD 파이프라인 구축하기

블로그의 CI/CD 파이프라인 구축을 위한 리팩토링을 진행하던 중, 클러스터 내부의 젠킨스 마스터의 스토리지를 루트 볼륨이 아닌 별도의 EBS로 운영하면 좋겠다는 생각을 했다.

별도의 EBS로 구축하게되면 비용 상 문제로 서버를 종료시켰다가 재시작해도 파이프라인 관련 데이터가 삭제되지 않고, 언제든지 원할 때마다 뗏다 붙였다 하며 다수 프로젝트의 파이프라인을 하나의 젠킨스 스토리지를 통해 관리하면 편리할 것 같다고 생각했다.

##  2. AWS EBS CSI


쿠버네티스에서 어떻게 AWS의 리소스를 사용할 수 있을까? 원래 쿠버네티스에서는 스토리지를 제공하는 벤더마다 직접 쿠버네티스 코드 베이스 기반에서 자신들의 스토리지를 사용할 수 있도록 플러그인을 개발해서 제공했다.

이 방식은 두 가지 문제점이 있었다.


1.  쿠버네티스가 버전 업 될 때마다 플러그인 역시 정상적으로 동작하기 위해 매번 업데이트를 해야했다.
2. 플러그인은 쿠버네티스 안에서 많은 권한을 갖게되기 때문에 벤더에서 제공하는 플러그인에 문제가 생기면 전체 클러스터에 위협이 될 수 있었다.

이러한 문제점을 기반으로 쿠버네티스는 CSI라는 것을 도입헀다. CSI는 Container Storage Interface의 준말이다. 외부 스토리지 서비스와의 인터페이스 역할을 해주는 기능을 제공하고, 각 벤더는 쿠버네티스 코드 베이스 기반이 아닌 CSI 형식에 맞게 자신들의 스토리지 서비스를 제공할 수 있도록 Driver만 적절히 개발하면 되는 방식으로 변경되었다.

인터페이스가 생겨남에 따라 의존성이 줄어들어 쿠버네티스가 버전 업 될 때마다 함께 플러그인을 업데이트해야했던 비용도 줄어들게되고, 각 벤더에서는 Driver만 제공하면 되기에 더 많고 다양한 벤더에서 CSI Driver를 제공할 수 있게 되었다.

그럼 CSI가 정확히 어떤 역할을 하는걸까?

CSI는 앞서 말했던 것처럼, 각 벤더에서 개발 및 제공하는 CSI driver와 컨테이너화된 쿠버네티스 오브젝트들을 API로 연결시켜주는 기능을 한다.

각 벤더(AWS, GCP 등)에서 CSI 드라이버를 개발해서 제공하면, 쿠버네티스에 어플리케이션을 배포하는 개발자가 해당 드라이버를 쿠버네티스 클러스터 내부에 Pod 형태로 배포하고(ex. aws-ebs-csi-driver Pod), 개발자가 CSI를 통해 CSI 드라이버의 스토리지 프로비저너로 스토리지를 생성하는 StorageClass 리소스를 정의해서, 각 벤더의 스토리지를 사용할 수 있게된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/0B8FF69D-11ED-47F7-897F-D2C4521D2B75/E1D65F41-EB70-4EED-898D-96453BCDC0EA_2/5P5qRVUg0FN1vmymYo424Ci01x8auqzeBi3EbDJLhBAz/Image.png)

##  3. 인증과 인가


예를 들어 AWS의 EBS를 프로비저닝하고 쿠버네티스의 Pod PV로 사용하려고 한다고 가정해보자. AWS는 이 계정에서 AWS 리소스를 생성/삭제 등의 작업을 하려고하는 쿠버네티스 클러스터/Pod가 그럴 권한이 있는지를 체크해야한다. 

Terraform 같은 경우는 로컬에 설치된 AWS CLI를 통해서 등록된 IAM 계정과 비밀번호를 이용해 인증/인가를 수행한다. 쿠버네티스는 로컬이 아닌 운영 서버에 구성되기 때문에 AWS CLI를 통해 인증/인가를 수행했다가는 보안적으로 크게 위험할 수 있고, 적절한 설정이 없다면 클러스터 내의 모든 Pod가 권한을 갖게되어 하나만 공격당해도 클러스터 및 인프라 전체가 위험해질 수 있다.

그럼 AWS는 어떤 방식으로 인증/인가를 구현할까? 두 가지 방식이 있다.


1.  kube2iam, kiam 등의 AWS IAM Role과 클러스터/Pod를 연결해주는 오픈소스 소프트웨어 사용
2. IRSA 사용

4. ## kube2iam, kiam


kube2iam과 kiam은 AWS IAM Role과 클러스터/Pod를 연결해주는 도구이다. 현재 리팩토링 중인 블로그에 적용하기 위해 두 도구에 대해 알아보았다. 

kube2iam은 쿠버네티스 데몬셋 리소스(모든 워커노드에 기본적으로 배포되는 Base Pod) 형태로 클러스터에 배포된다. 우선 전체 클러스터 Pod 들에게 AWS API를 호출할 수 있는 권한은 다 부여하고,  iptable 설정을 통해 호출 요청을 인터셉트해서 이 Pod가 AWS API를 호출할 수 있는 자격이 있는지 annotation을 통해 검증하게 된다.

kube2iam은 Pod가 CrashBack 되는 이슈가 많다고 하여 후보군에서 제외되었다.

kiam은 현재 Deprecated 상태여서 후보군에서 제외되었다. Kiam은 공식 깃허브 리드미에서 IRSA로 대체되었다고 소개하고있다.

그럼 IRSA는 뭘까?

## 5. IRSA


IRSA는 IAM Role for Service Account의 준말로, AWS에서 공식적으로 AWS의 IAM Role과 서비스 어카운트 리소스를 연결할 수 있도록 제공하는 기능이다. 이 기능은 기존의 kube2iam, kiam 같은 도구들과 다르게 AWS에서 제공하는 서비스인만큼 간편하게 IAM Role과 ServiceAccount를 통한 Pod간의 연결을 구성할 수 있다.

대신 IRSA는 EKS에서만 사용이 가능하다는 단점이 있다. EKS는 Elastic Kubernetes Service의 준말로, AWS에서 AWS 클라우드 환경 위에 쿠버네티스 클러스터를 간편하게 구축하고 관리할 수 있도록 제공하는 서비스이다.

6. ##  EKS로 서비스 이전 결정


아래 두 가지 이유로 인해 kube2iam이나 kiam 같은 기능을 하는 더 훌륭한 완성도의 도구가 없다는 것이 너무 아쉬웠다.


1.  현재 나의 블로그 서비스는 kubeadm을 통해 구성된 bare metal 클러스터를 기반으로 하고있다. 

IRSA를 사용하기 위해서는 EKS로 서비스를 이전하는 비용을 감수해야한다.


2.  AWS에 대한 종속성이 증가한다. 세상에는 많은 클라우드 플랫폼이 있다. 

아무리 AWS가 가장 널리 쓰이는 클라우드 플랫폼이라고는 하지만, 더 큰 규모의 서비스를 운영할 때, GCP, Azure, NCP 뿐만 아리나 온프레미스 서버를 활용할 수도 있을텐데 AWS 리소스를 관리하기 위해 전체 쿠버네티스 클러스터를 AWS에 배포해야한다는 종속성이 마음에 걸렸다.

그래도 kubeadm을 통한 베어메탈 클러스터 구축은 한 번 경험해봤고, EKS를 활용한 클러스터 구축 방법을 몰라서 못하는 것보단 할 줄은 알고있는 것이 여러모로 유익하겠다는 생각이 들어 EKS로 서비스를 이전하기로 결정했다.

## 7. EKS


사실 이전의 베어메탈 클러스터 역시 Terraform을 통해 구현해두었어서 복잡한 수많은 AWS 리소스들을 삭제하고 EKS 클러스터로 새롭게 구축하는데 큰 어려움은 없었다.

사실 EKS가 간편하게 클러스터를 구성하는데 도움을 주기위한 서비스인만큼, 오히려 많은 복잡성과 tf 코드가 줄었다고도 말할 수 있다.

이 김에 EKS를 통해 클러스터를 구성할 때와 Baremetal로 구성할 때의 Terraform 리소스 차이를 비교해보자.

### 기존 BareMetal 방식


기존 방식에서 기본적인 쿠버네티스 클러스터를 구성하기위해 아래 Terraform 리소스들이 필요했다.


1.  VPC
2. Subnet
3. Internet Gateway
4. Route table
5. Route table association
6. Security Group
7. NLB
8. Loadbalancer Target Group
9. Loadbalancer Target Group Attachment
10. Loadbalancer Listener
11. Null Resource
12. EC2 Instances
13. Output

### EKS

> main.tf


```dockerfile
provider "aws" {
  region = var.region
}

data "aws_availability_zones" "available" {
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

locals {
  cluster_name = "eks-${random_string.suffix.result}"
}

resource "random_string" "suffix" {
  length  = 8
  special = false
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "blog-vpc"

  cidr = "10.0.0.0/16"
  azs  = slice(data.aws_availability_zones.available.names, 0, 3)

  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                      = 1
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${local.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"             = 1
  }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "19.15.3"

  cluster_name    = local.cluster_name
  cluster_version = "1.27"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_group_defaults = {
    ami_type = "ami-0c9c942bd7bf113a2"

  }

  eks_managed_node_groups = {
    one = {
      name = "node-group-1"

      instance_types = ["t3.small"]

      min_size     = 1
      max_size     = 3
      desired_size = 2
    }

    two = {
      name = "node-group-2"

      instance_types = ["t3.small"]

      min_size     = 1
      max_size     = 2
      desired_size = 1
    }
  }
}


# https://aws.amazon.com/blogs/containers/amazon-ebs-csi-driver-is-now-generally-available-in-amazon-eks-add-ons/ 
data "aws_iam_policy" "ebs_csi_policy" {
  arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
}

module "irsa-ebs-csi" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
  version = "4.7.0"

  create_role                   = true
  role_name                     = "AmazonEKSTFEBSCSIRole-${module.eks.cluster_name}"
  provider_url                  = module.eks.oidc_provider
  role_policy_arns              = [data.aws_iam_policy.ebs_csi_policy.arn]
  oidc_fully_qualified_subjects = ["system:serviceaccount:kube-system:ebs-csi-controller-sa"]
}

resource "aws_eks_addon" "ebs-csi" {
  cluster_name             = module.eks.cluster_name
  addon_name               = "aws-ebs-csi-driver"
  addon_version            = "v1.20.0-eksbuild.1"
  service_account_role_arn = module.irsa-ebs-csi.iam_role_arn
  tags = {
    "eks_addon" = "ebs-csi"
    "terraform" = "true"
  }
}
```


## EKS 클러스터 관리


EKS를 사용하면서 가장 놀라웠던 기능 중 하나는 EKS 클러스터를 로컬에서 관리할 수 있다는 것이었다.

이전에 kubeadm으로 클러스터를 구성했을 때는 CI/CD 파이프라인을 통해 배포를 자동화하는 것이 아니라면 직접 마스터노드에 접근해서 kubectl CLI를 통해 파드를 배포해야했었다.

EKS가 마스터 노드들을 관리해주고 IAM Role과도 쉽게 연결할 수 있는 기능을 지원하기 때문에 아래 커맨드를 통해 로컬의 AWS CLI configuration으로 권한이 확인되면 로컬에서 manifest를 작성해 배포하거나 파드 및 노드의 상태를 확인하는 것이 가능하다.

```dockerfile
aws eks --region REGION_NAME update-kubeconfig --name CLUSTER_NAME
```


REGION_NAME에는 ap-northeast-2 등 AWS provider에서 설정한 region을 입력하면되고, CLUSTER_NAME에는 eks 모듈 블록의 cluster_name 필드에 지정한 이름을 입력하면 된다.

## eksctl


## Baremetal 방식와 EKS 방식의 클러스터 관리 차이


AWS의 Elastic Block Storage(EBS)를 쿠버네티스 Pod의 볼륨으로 사용하려면,  


1. AWS에서 제공하는 aws-ebs-driver를 Pod로 배포
2. aws-ebs-csi-driver가 EBS 관리를 k AWS API를 사용할 수 있도록 IAM 권한, 역할 및 정책을 설정
3. aws-ebs-csi-driver의 쿠버네티스 ServiceAccount 리소스를 생성하고 IAM Role과 연결
3. aws-ebs-csi-driver를 사용해서 AWS EBS와 연결되는 쿠버네티스 PV 리소스를 설정 후 배포
4. 해당 PV를 사용하도록 하는 PVC를 설정 후 배포
5. EBS를 볼륨으로 사용할 Pod에서 해당 PVC로 볼륨을 설정 후 배포

이 다섯 단계를 거친다.

2번이 가장 복잡하다.

## IAM OIDC_PROVIDER 생성


eks를 통해 구성한 클러스터에서는 IRSA(IAM Role for Service Account)를 통해 쿠버네티스의 서비스 어카운트 리소스와 aws의 IAM Role을 연결해서 클러스터 내부의 Pod들이 IAM 권한대로 aws 리소스들에 접근하는 것이 가능하다.

eks가 아닌 베어메탈 클러스터의 경우에 동적으로 aws의 리소스를 생성/연결/관리하기 위한 권한을 얻기 위해서는 OIDC Provider를 필요로 한다. 특히 kube2iam과 kiam

terraform은 aws cli를 통해 aws에 로그인을 해서 aws 리소스를 생성하는 방식을 사용하고있다.

클러스터 내부의 aws-ebs-csi-driver Pod에서 EBS 볼륨에 대한 요청(생성, 삭제, attach 등)을 하기 위해 AWS API를 사용해야하는데, 이 특정 AWS API를 사용하기 위한 권한을 부여하기 위해 aws cli 로그인과 같이 인증 및 인가가 필요하다.

이걸 가능하게 해주는 것이 OIDC provider이다.

RBAC으로 AWS IAM role에 권한을 부여받은 pod가 oidc provider를 통해 AWS STS `AssumeRoleWithWebIdentity` API를 호출할 권한을 얻게되고, 드라이버가 aws의 ebs 스토리지를 관리할 수 있게 된다.

이 OIDC provider를 통해 특정 권한을 가진 IAM과 연결을 인증해주는 토큰을 발급받아서 .kube/config 에 넣어주면, 드라이버가 이 토큰을 통해 AWS API 요청을 보낼 수 있게 된다.


- 드라이버의 권한 설정을 위한 IAM Policy 생성

## Summary


EKS로 블로그를 이전하게 되면서 새로운 기술을 학습하고, 특히 나의 Terraform에 대한 지식이 무지에 가까웠다는 것을 깨닫는 기회가 되어서 좋았다. 다만 EKS는 시간 당 0.1USD의 요금이 부과되는데, 일개 학부생인 나에게는 너무나도 비싼 금액이다. 현재 환율로 달에 약 10만원 가까운 비용이 청구되고 워커노드용 EC2는 또 별도로 청구된다. IAM Policy와 namespace를 활용해서 EKS를 최대한 뽕 뽑는 방식으로 프로젝트를 진행해나가야겠다.

## References


[https://marcincuber.medium.com/amazon-eks-with-oidc-provider-iam-roles-for-kubernetes-services-accounts-59015d15cb0c](https://marcincuber.medium.com/amazon-eks-with-oidc-provider-iam-roles-for-kubernetes-services-accounts-59015d15cb0c)

[https://medium.com/in-the-weeds/service-to-service-authentication-on-kubernetes-94dcb8216cdc](https://medium.com/in-the-weeds/service-to-service-authentication-on-kubernetes-94dcb8216cdc)

[https://developers.redhat.com/articles/2022/10/06/csi-drivers-essential-kubernetes-storage](https://developers.redhat.com/articles/2022/10/06/csi-drivers-essential-kubernetes-storage)

[https://towardsaws.com/ebs-csi-driver-amazon-eks-4eab8966dbb4](https://towardsaws.com/ebs-csi-driver-amazon-eks-4eab8966dbb4)

[https://dev.to/aws-builders/deploy-an-aws-eks-cluster-using-terraform-iac-5bjc](https://dev.to/aws-builders/deploy-an-aws-eks-cluster-using-terraform-iac-5bjc)
