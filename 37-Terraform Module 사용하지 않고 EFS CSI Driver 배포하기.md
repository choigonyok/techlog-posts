[Tags: CLOUD INFRA KUBERNETES OPS]
[Title: Terraform Module 사용하지 않고 EFS CSI Driver 배포하기]
[WriteTime: 2023/11/24]
[ImageNames: ]

## Content

1. Preamble
2. EFS / EBS / S3 비교
3. EFS CSI Driver & IRSA
4. AWS IAM Policies 종류
5. Terraform으로 CSI Driver 배포
6. Subject & Audience란?
7. STS란?
8. [TroubleShooting] WebIdentityErr
9. 전체 테라폼 코드
10. Summary

## 1. Preamble


지난 글에서 AWS EBS를 Pod에 볼륨으로 사용하면서 고가용성을 보장하기 위해 Multi-AZ 솔루션들과 Service Toplogy 및 몇몇 트러블 슈팅에 대해 작성했다.

결론을 요약하자면 EBS를 사용하면서 고가용성을 보장하는 방법은 없다라는 결론이 나왔고, 따라서 Cross-AZ를 지원하는 EFS를 볼륨으로 사용하기로 결정했다.

EFS가 Cross-AZ를 보장한다면 당연히 EFS를 써야지 AWS에서는 왜 굳이 EBS를 만들었을까?

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/44F3C823-1AD2-405F-8A72-8A1D9EB01A7A_2/HuhbFNr2bH4OguBHGg2JwpUIHIlEVLMwKZdT9RLQmrUz/Image.png)

EBS는 One Zone에서만 사용가능하기 때문에 외부적으로 Race Condition이 발생할 확률이 적다. 그래서 루트볼륨이나 데이터베이스 스토리지로 많이 활용된다.

EFS는 Multi-AZ에서 사용 가능한만큼, 데이터 공유나 백업 용도로 자주 사용된다. 더 활용성이 높은만큼 더 비싸다. EBS 볼륨이 GB당 월 0.1USD인 것에 비해 EFS는 Stand 타입 기준으로 3배가 더 비싸다.

AWS에서 사용 목적에 따라 스토리지를 분리해 더 많은 사용자들이 용도에 맞게 사용할 수 있도록 전략을 세웠다는 생각을 해볼 수 있다.

지난 글에서 Driver와 AWS IAM Role 연결 관련한 내용을 생략했어서, 이 내용을 포함해서 Terraform으로 EFS CSI Driver를 클러스터에 배포하는 과정을 정리해보려고 한다.

## 2. EFS / EBS / S3 비교


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/C4076207-D124-4BE2-8EDE-12EDA332EE83_2/lOEbLO6wHppWMIUg3xpp6BHE22WRxtVrdJQZBAcIsd4z/Image.png)

### Block / File / Object


먼저 Block, File, Object 스토리지에 대해 알아보자.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/162A4019-D104-43DD-8FB3-FDF7EFB0BD22_2/K1xRypJOnQk6Jh5JVEIXXNXNpxzaXxpcmuThXZ5Gapsz/Image.png)

Block Storage는 빠르다. 데이터틑 정해진 블록 단위로 쪼개서 저장하기 때문이다. 이게 성능과 무슨 상관이 있는 것일까? 텍스트 파일을 예로 들어보자. 파일 / 오브젝트 스토리지의 경우에 텍스트 파일의 전체 내용 중 한 글자만을 수정했다면, 전체 텍스트 파일을 다시 저장해야한다. 반대로 블록 스토리지는 수정하는 한 글자의 데이터가 저장되어있는 블록들만 변경하면 된다.

또 하나의 블록을 처리하는 것이 다른 블록에 영향을 주지 않기 때문에 동시에 여러 블록을 병렬적으로 처리할 수 있다.

Object Storage는 데이터를 분산해서 저장한다. 때문에 Write은 분산으로 인해 느리고, Read는 분산된 데이터들을 병렬로 읽어올 수 있기 때문에 빠르다.

File Storage는 내부적으로 Locking을 구현할 수 있다. 다수의 사용자가 하나의 데이터를 읽고 써도 File Storage가 데이터의 안정성을 보장한다.

### S3


S3는 Simple Storage Service의 약자로, Object Storage이다. 세 단어 모두 S로 시작해 S3로 이름붙었다. S3는 오브젝트 스토리지이다.

Write에 비해 Read 수가 많을 때 적합하다. 그래서 백업이나 빌드파일 서빙 용도로 많이 사용된다. 백업은 자주 Write가 일어나지 않기 때문에 적합하다. 빌드파일 서빙은 한 번 빌드파일을 Write해두면 Read가 훨씬 빈번히 일어나기 때문에 적합하다.

예전부터 S3가 운영환경에 웹 어플리케이션을 배포할 때 Frontend BuildFile을 서빙하는 용도로 많이 사용되었던 걸 봤었다.

S3는 API를 통해 데이터를 관리한다. 데이터는 메타데이터와 함께 저장되기 때문에 Retrieve시에 메타데이터를 통한 효율적인 검색이 가능하다.

## EBS


EBS는 Elastic Block Storage의 준말로, Block Storage이다. 빠른 성능이 장점이다. 대신 Multi-AZ를 지원하지 않는다는 단점이 있다.

높은 성능이 장점이기 때문에 EC2 인스턴스의 Default 루트 볼륨으로 사용되고, 특히 데이터베이스는 높은 성능이 중요하고 EBS에서 스냅샷을 통한 백업 및 롤백을 지원하기 때문에 데이터베이스의 스토리지로도 많이 활용된다.

EBS는 말한 것처럼 Multi-AZ를 지원하지 않는데, 데이터베이스의 고가용성을 보장하려면 EBS 대신 다른 스토리지를 써야하는 것일까? 그렇지 않다.

나중에 또 다루겠지만, 이 문제는 ReadDB / WriteDB 데이터베이스를 분리하는 방식을 사용한다.

보통 데이터베이스는 Write보다 Read가 훨씬 많기 때문에 Leader DB가 혼자 Write를 담당하고, 나머지 DB들이 Read를 담당한다. Write DB가 새로운 데이터를 쓰면 다른 EBS들의 데이터를 동기화(Replication)시키게 되고, 정의된 알고리즘을 통해 Leader DB의 장애를 판단하고 Read DB 중 하나를 Leader로 선정하는 방식이 사용된다.

EBS는 부착(Attach)되는 방식이기 때문에 대상에 장애가 생긴다고 해도 데이터가 손실되지 않고, 원할 때마다 뗐다 붙였다할 수 있다.

## EFS


이 글에서 쿠버네티스에 프로비저닝 될 EFS는 Elastic File System의 준말로 File Storage이다. EBS와 달리 여러 인스턴스가 공유할 수 있고, 더 나아가서 Multi-AZ에서도 공유될 수 있는 스토리지이다.

이로 인해 컨테이너화된 어플리케이션에서 유용하게 사용된다. 쿠버네티스에서 Pod는 Affinity를 설정하지 않는 한 매번 다른 노드(인스턴스)에 배포되는데, 하나의 EC2에만 부착이 가능한 EBS를 사용한다면 Pod가 다른 노드에 배포될 때마다 기존의 스토리지와의 연결이 끊어지게된다. EFS를 사용하면 어느 노드에 배치되던지 동일한 스토리지에 접근할 수 있게된다.

이 것이 내가 Jenkins Pod의 PV로 EBS를 사용하려다 EFS로 옮기게 된 가장 큰 이유이다. 

### EFS가 매력적인 이유

1. Cross-AZ 사용이 가능하다.
2. fileSystemId를 통해 멱등성을 유지한다. 매번 반복해서 파일시스템을 생성하는 것이 아니라 같은 네트워크 안에 해당 ID로 생성되어있는 EFS가 있으면 새로 생성하지 않고 그 EFS를 가져다가 사용한다.
3. Cross-AZ가 아닌 One Zone mode로도 사용이 가능하다.
4. basepath와 subpath 이용해서 연관성과 독립성을 동시에 구성할 수 있다.
5. subpath는 Pod명, Namespace명 등 동적으로도 생성이 가능하다.

## 3. EFS CSI Driver & IRSA


먼저 efs-csi-driver와 IRSA에 대해 설명하자면, 내가 kubeadm 쿠버네티스 클러스터에서 EKS 클러스터로 블로그 어플리케이션을 이전하게 된 큰 원인 중 하나라고 할 수 있다. CSI Driver와 IRSA에 관한 내용은 

**내가 느낀 EKS와 Bare-Metal K8S의 차이점 (23.11.12)**

의 **5. IRSA w/ EBS CSI** 섹션에서 확인할 수 있다.

## 4. AWS IAM Policies 종류


구현에 앞서서 AWS IAM Policy에 대해 살펴보자. Policy는 Managed / Customer Managed / Inline /  Trusted 총 네 종류가 있다. 

이 중 Trusted는 인증(Authorization)의 개념에 가깝고, 나머지 세 정책은 인가(Authentication)의 개념에 가깝다.

예를 들어 역할과 역할에 할당되는 정책, 그리고 그 역할을 위임 받고싶어하는 서비스가 있다고 가정하자. 이 때 두 가지를 확인해야한다.


1. 이 서비스가 이 역할을 위임받아도 되는 서비스인가?
2. 이 역할을 가지면 어떤 일까지 할 수 있는 것인가?

1번이 인증, Trust Policy와 관련되어있고, 2번이 인가, 나머지 세 정책과 관련되어있다.

### Managed Policy


Managed Policy는 AWS에서 직접 관리하는 정책들을 의미한다. ARN(Amazon Resource Number)으로 유니크하게 구분되는 정책들이며, 따로 정책을 JSON 등으로 수정할 수 없다.

### Customer Managed Policy


Customer Managed Policies는 사용자가 직접 생성/관리하는 정책을 의미한다. 기존 Managed Policy 여럿을 포함해 Inline Policy를 생성할 수도 있고, JSON 형식으로 커스텀한 정책을 생성할 수도 있다.

### Inline Policy


Inline Policy 역시 사용자가 직접 생성/관리하는 정책이다. Custom Managed Policy와는 다르게 인라인 정책은 특정 사용자 / 역할 / 그룹에서만 적용되는 정책이다. 해당 사용자 / 역할 / 그룹이 사라지면 함께 사라진다.

### Trust Policy


신뢰관계(Trusted Policy)를 정의하는 정책이다. 역할에 부착되는 정책이고, 해당 역할을 누가 위임받을 수 있는지를 정의하는 정책이다. 

아래 두 가지 조건이 충족되면 신뢰관계를 통해 권한을 받을 수 있다.


1. 권한을 위임해주는 Role의 신뢰관계의 Principle에 권한을 위임받고자 하는 서비스가 정의되어있어야한다.
2. 권한을 위임해주는 Role의 신뢰관계의 Action에 sts:AssumeRole이 정의되어있어야한다.

액션 중 AssumeRole은 AWS 리소스(IAM, EC2 등)가 다른 리소스에 접근할 수 있도록 역할을 위임할 때 사용하고, AssumeRoleWebIdentify는 OIDC Provider를 통해 AWS 리소스가 아닌 외부에서 AWS 리소스에 접근할 수 있도록 역할을 위임할 때 사용한다.

## 5. Terraform으로 CSI Driver 배포


CSI Driver가 EFS 관련한 AWS API에 접근할 수 있도록 EFS 관련 정책이 설정된 IAM Role을 생성하고 이를 CSI Driver에 부여해주어야한다.

### IAM Role + Policy 생성


### IRSA Module


AWS에서 많은 module을 제공한다. 상대적으로 편리하게 EFS CSI Driver를 위한 IRSA를 구현할 수 있는 iam-assumable-role-with-oidc 모듈도 있다. 이 글에서는 기본기를 더 다지기 위해 IRSA를 테라폼 AWS resource를 통해 구현할 예정이다. IRSA 모듈을 통한 구현은 아래 테라폼 코드를 IAM Role과 Policy 관련된 리소스들 대신 사용하면 구현할 수 있다.

```yaml
module "irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
  version = "4.7.0"
  create_role                   = true
  role_name                     = "AmazonEKSTFEFSCSIRole-${module.eks.cluster_name}"
  provider_url                  = module.eks.oidc_provider
  role_policy_arns              = [data.aws_iam_policy.efs_csi_policy.arn]
  oidc_fully_qualified_subjects = ["system:serviceaccount:kube-system:efs-csi-controller-sa"]
}
```


### Without Module


Terraform의 AWS 리소스 중에 aws_iam_policy와 aws_iam_role_policy가 있다. 전자는 정책만을 따로 생성해서 attachment 리소스를 통해 부착해주어야하고, 후자는 Role을 생성하면서 적용될 Policy를 한 번에 정의할 수 있다.

우선 efs의 file system과 mount target을 직접 생성해야한다. AmazonEBSCSIDriverPolicy에서는 EBS 볼륨의 프로비저닝을 허용하는 권한이 정의되어있는데, AmazonEFSCSIDriverPolicy에는 fileSystem과 mountTarget에 대한 프로비저닝 대신 access point에 대한 접근만이 정의되어있다.

```yaml
resource "aws_efs_file_system" "jenkins_volume" {
  creation_token = "${local.cluster_name}-jenkins_volume"
  tags = {
    Name = "${local.cluster_name}-jenkins_volume"
  }
}

resource "aws_efs_mount_target" "private_subnet" {
  count          = 3
  file_system_id = aws_efs_file_system.jenkins_volume.id
  subnet_id = module.vpc.private_subnets[count.index]
  security_groups = [module.eks.node_security_group_id]
}
```


### creation_token


fileSystem의 creation_token 속성은 멱등성을 위한 필드이다. 임의의 문자열을 지정할 수 있다.

멱등성이란 desired state를 current state와 동일하게 유지시키는 성질을 의미한다. AWS는 fileSystem에 대한 프로비저닝 AWS API 요청이 왔을 때 creation token을 참조해서 추가로 생성해야하는지에 대한 여부를 판단하게된다.

만약 같은 creation token을 가진 fileSystem이 이미 존재한다면 AWS는 fileSystem 프로비저닝 요청을 받아도 추가로 fileSystem을 생성하지 않는다.

fileSystem의 creation token이 ID와 같은 개념이기 때문에, 다른 creation token을 가지지만 이름은 같은 fileSystem들을 여러개 프로비저닝할 수 있다.

### mount target


mount target으로 특정 파일시스템이 볼륨으로 마운트될 수 있는 서브넷을 지정해야한다.

count를 통해 마운트 될 수 있는 최대 서브넷 개수를 설정할 수 있고, Jenkins Pod가 어느 가용영역에 배치되든 사용할 수 있게 하기 위해 EFS를 마운트하는 것이기 때문에, 워커노드들이 배치될 수 있는 전체 Private Subnet들을 target으로 지정해준다.

count.index는 count가 3이기 때문에, 0, 1, 2를 돌면서 id를 적용시키게 된다. 어떤 파일시스템을 쓸 것인지에 대한 file_system_id을 지정해주고, 이 파일시스템이 적용될 보안그룹을 지정해준다.

### EFS CSI Driver Add On


```yaml
resource "aws_eks_addon" "efs-csi" {
  cluster_name  = module.eks.cluster_name
  addon_name    = "aws-efs-csi-driver"
  addon_version = "v1.7.0-eksbuild.1"
  service_account_role_arn = aws_iam_role.efs_csi_driver.arn
  tags = {
    "eks_addon" = "efs-csi"
    "terraform" = "true"
  }
}
```


EKS에서 간편하게 efs-csi-driver를 클러스터에 배포할 수 있도록 추가기능(add-on)을 제공한다.

addon의 버전을 지정할 때, 쿠버네티스 클러스터의 버전이 무엇인지에 따라 사용할 수 있는 add on 버전이 달라진다.

```yaml
aws eks describe-addon-versions --kubernetes-version 1.27 --addon-name aws-efs-csi-driver
```


AWS CLI를 통해 자신의 클러스터 버전에 사용가능한 add on 버전을 확인한 후에 명시해야한다.

### IAM Role


다음으로는 efs-csi-driver가 AWS EFS의 access point에 대한 관리 권한을 가질 수 있도록 IAM Role과 Policy, 그리고 둘을 연결시켜주기 위한 attachment 리소스를 정의한다.

```yaml
resource "aws_iam_role" "efs_csi_driver" {
  name = "efs_csi_driver"
  assume_role_policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Effect" : "Allow",
        "Principal" : {
          "Federated" : "${module.eks.oidc_provider_arn}"
        },
        "Action" : "sts:AssumeRoleWithWebIdentity",
        "Condition" : {
          "StringLike" : {
            "${module.eks.oidc_provider}:sub" : "system:serviceaccount:kube-system:efs-csi*",
            "${module.eks.oidc_provider}:aud" : "sts.amazonaws.com"
          }
        }
      }
    ]
    }
  )
}

data "aws_iam_policy" "efs_csi_driver" {
  arn = "arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy"
}

resource "aws_iam_role_policy_attachment" "efs_csi_driver" {
  policy_arn = data.aws_iam_policy.efs_csi_driver.arn
  role       = aws_iam_role.efs_csi_driver.name
}
```


data 블록의 aws_iam_policy를 통해 managed policy를 직접 json 형식으로 테라폼에 작성하지 않아도 참조해서 정책을 정의할 수 있다.

aws_iam_role 리소스에서는 inline_policy, policy, assume_role_policy 속성을 통해 inline, custom managed, trust policy를 정의할 수 있다.

EKS는 AWS 리소스이지만, EKS가 아닌 EKS에 배포된 efs-csi-driver Pod가 EFS에 접근할 수 있는 권한을 얻어야하기 때문에 AssumeRole이 아닌 AssumeRoleWebIdentify 액션으로 신뢰관계를 정의한다.

Principal은 권한을 위임받을 수 있는 Peer를 지정하는 필드인데, Federated를 통해 EKS에서 제공하는 OIDC Provider를 지정했다.

Condition 필드는 권한을 더 최소화하기 위해 사용한다. 정말 권한이 필요한 대상만 권한을 가질 수 있게 조건을 걸 수 있다. EKS addon을 통해 efs-csi-driver가 배포될 때 서비스 어카운트 오브젝트인 efs-csi-controller-sa와 efs-csi-node-sa가 함께 배포된다.

```yaml
"${module.eks.oidc_provider}:sub" : "system:serviceaccount:kube-system:efs-csi*",
```


이 코드를 통해 kube-system 네임스페이스의 efs-csi prefix로 시작하는 서비스 어카운트만 이 역할을 위임받을 수 있도록 설정해준다.

```yaml
"${module.eks.oidc_provider}:aud" : "sts.amazonaws.com"
```


추가로 이런 코드도 아래에 작성해준다.

## 6. Subject & Audience란?


sub과 aud, sts라는 키워드를 확인할 수 있다. 무슨 의미일까? 

sub은 Subject, 즉 OIDC Provider를 통해 인증될 대상을 의미한다.

aud는 Audience, OIDC Provider를 통해 인증할 대상을 의미한다.

즉, OIDC Provider를 통해 "system:serviceaccount:kube-system:efs-csi*"를 인증할 것이고 , 그 인증으로 sts.amazon.com에 대한 권한을 얻겠다는 것을 의미한다.

## 7. STS란?


AWS의 STS는 Security Token Service의 준말이다. AWS의 리소스에 접근하기 위한 권한을 토큰으로 생성해주는 임시 보안 자격 증명 서비스이다. 

권한을 위임받고 싶은 대상(Role 또는 OIDC Provider)가 STS에 API 요청을 보내면 STS에서 임시 토큰을 발급해준다. 이 토큰을 통해 Role을 위임받게 되고, 해당 사용자가 적합한 Role과 Policy가 정의되어있는지에 따라 특정 리소스에 접근할 수 있게된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/712CE3AA-89D8-4756-9EEF-65D4E523DF11/9363D3C5-46E9-4118-80E2-10DDF3B6E723_2/ZJy6CW8qD6wWvA7vxi1jdgZVDmS7YJ3vB3npeGcoW9gz/Image.png)

## 8. [TroubleShooting] WebIdentityErr


처음 sub을 efs-csi*로 지정하지 않고, efs-csi-controller-sa로 지정했었다. 그러다 아래와 같은 에러를 만났다.

```yaml
Failed to fetch File System info: Describe File System failed: WebIdentityErr: failed to retrieve credentials
caused by: AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity
```


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/F669DD27-D2DE-4CE8-B675-F712917AF990_2/QsBKZj723FnDN3PKyGxPgmn1gz2Cd8L8A3zg1skOy2Yz/Image.png)

IAM Role에 AssumeRoleWithWebIdentity를 수행하기 위한 권한이 없다는 에러메시지가 출력되었다.

위에서 이야기한 것처럼 addon을 통해 CSI Driver를 배포했는데, 버전 문제가 있어서 SA가 제대로 생성되지 않아 이런 오류가 발생한 것인가 생각했다. 그래서 SA 리스트를 확인해봤지만 정상적으로 생성되어있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/52FDE7E1-C9BB-40D2-99DA-8696DFE521ED_2/upQhO0xy7WZTCmlIY0iWodQyybb59jSa0Yjvs7x6aDYz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/9CC20C5B-7457-4289-885B-940D6C16883E_2/kzHG90eTWiMR1sCmNogsyKA8QRA5P9dI26We28vH89sz/Image.png)

위 사진처럼 ServiceAccount는 잘 생성되어있었다. 그럼 EKS에서 SA에 IAM Role을 할당해주도록 선언하는 annotation이 없거나 잘못되어있나해서 확인해보았다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/CA133D29-E923-438B-916E-2EE4C1F4F33A_2/igy9IzR905hL5o6DWnoLiUuBA0rW8XbPsVsBySktbu8z/Image.png)

그것 역시 아니었다. 콘솔에서 확인해보니 Role은 잘 생성되어있었고, arn이 일치한다. 정책도 아래 사진처럼 Role에 잘 attach 되어있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/18EBDD7B-EA0B-4A3D-A6CD-E5172433BC74_2/EhLHmdMVfcjqXu9O0GKmLyRCwGf9q6xEJY0E2tajm5Uz/Image.png)

그럼 신뢰관계를 Terraform으로 정의한 부분이 잘못되었나 생각해 콘솔에서 확인해보았다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/E118DE00-22F2-49D1-B0C7-E2C32ECFBFE2_2/yy982uCmrq93l6jx3tSPo5T8rwnjqEy4JtXcxZWLwUsz/Image.png)

신뢰관계 역시 Role에 잘 설정되어있었다. 관련해서 서칭을 하다가 AWS에서 정확히 내가 겪은 에러에 대해 게시한 포스트가 있었다. [AWS Post](https://repost.aws/ko/knowledge-center/eks-load-balancer-webidentityerr)

오류는 내가 겪은 오류와 토씨 하나 다르지 않고 같았지만, 이미 포스트의 내용은 알고있었고, 다 적용해두고 확인해둔 상태였다. 그러다 무언가 이상한 걸 발견했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/62244E61-94FC-486D-BF80-EC44CFCE69FC_2/gEbIa1AxeRPSSM5XtI7LgOJbCW4z0vAzAt0MsvCSkyoz/Image.png)

클러스터 전체 ServiceAccount 오브젝트를 살펴보던 중 efs-csi라는 prefix를 가진 ServiceAccount가 하나 더 있는 것을 확인했다.

EFS CSI Driver 관련 AWS 공식문서를 다시 정독했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/D67FE887-3121-4A9C-8CF1-68FC3F3FFC99_2/CbxvJHSTxBEH4DhxfoneqlfmmIjD7pAnaxIstz5kay8z/Image.png)

IAM Role과 SA를 연결해주는 신뢰관계 정책 설정에서 wildcard가 사용되는 것을 보게되었다. efs-csi-controller-sa에만 권한을 주어야하는 것이 아니고, efs-csi-node-sa에게도 권한을 줘야한다는 것을 알게되어 아래와 같이 신뢰관계를 변경해주었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/7E1CA1E0-4387-4129-996F-0AA974F04F05/D00548D8-38F5-4F99-B9EF-59907D331EF0_2/0LjfDzgaIy9BlOBwoNI1P2mAefgNUyyujuW8XML4VDMz/Image.png)

이렇게 문제를 해결할 수 있었다.

그럼 efs-csi-node-sa는 뭐하는 SA일까? 왜 controller SA와 분리해둔 것일까?

###  efs-csi-node-sa & efs-csi-controller-sa


efs-csi-driver의 깃허브 레포지토리에서 관련 항목을 찾아보았다.


- Controller Service: CreateVolume, DeleteVolume, ControllerGetCapabilities, ValidateVolumeCapabilities
- Node Service: NodePublishVolume, NodeUnpublishVolume, NodeGetCapabilities, NodeGetInfo, NodeGetId, NodeGetVolumeStats

Node SA는 볼륨을 마운트/언마운트하는 기능을 하고, Controller SA는 볼륨을 프로비저닝하는 기능을 한다.

## 9. 전체 테라폼 코드


```yaml
data "aws_iam_policy" "efs_csi_policy" {
  arn = "arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy"
}

resource "aws_iam_role_policy_attachment" "policy_attachment" {
  policy_arn = data.aws_iam_policy.efs_csi_policy.arn
  role       = aws_iam_role.iamrole.name
}

resource "aws_efs_file_system" "example" {
  creation_token = "${local.cluster_name}-example"
  tags = {
    Name = "${local.cluster_name}-example"
  }
}

resource "aws_efs_mount_target" "example" {
  count          = 3
  file_system_id = aws_efs_file_system.example.id
  subnet_id = module.vpc.private_subnets[count.index]
  security_groups = [module.eks.node_security_group_id]
}

resource "aws_eks_addon" "efs-csi" {
  cluster_name  = module.eks.cluster_name
  addon_name    = "aws-efs-csi-driver"
  addon_version = "v1.7.0-eksbuild.1"
  service_account_role_arn = aws_iam_role.iamrole.arn
  tags = {
    "eks_addon" = "efs-csi"
    "terraform" = "true"
  }
}

resource "aws_iam_role" "iamrole" {
  name = "efs"
  assume_role_policy = jsonencode({
    "Version" : "2012-10-17",
    "Statement" : [
      {
        "Effect" : "Allow",
        "Principal" : {
          "Federated" : "${module.eks.oidc_provider_arn}"
        },
        "Action" : "sts:AssumeRoleWithWebIdentity",
        "Condition" : {
          "StringLike" : {
            "${module.eks.oidc_provider}:sub" : "system:serviceaccount:kube-system:efs-csi*",
            "${module.eks.oidc_provider}:aud" : "sts.amazonaws.com"
          }
        }
      }
    ]
    }
  )
}
```


## 10. Summary


트러블 슈팅 과정에서 module 없이 resource 만으로 efs csi driver를 프로비저닝하고 권한을 부여해주는 경험을 해보길 잘 했다는 생각을 했다.

애를 많이 먹었지만 이 과정을 통해 EFS, CSI, IAM에 대해 더 깊게 이해할 수 있었던 것 같다. 이런 scratch한 지식을 가진 상태에서 module 등의 편한 방식을 사용하는 것과, 그냥 처음부터 원리는 모르고 바로 편하고 간단한 길로 가는 것은 다르다고 생각한다.

## References


[AWS Official Docs](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/efs-csi.html) : Amazon EFS CSI 드라이버

[Github](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/dynamic_provisioning/README.md) : aws-efs-csi-driver 레포지토리

[StackOverflow](https://stackoverflow.com/questions/60609820/how-to-use-amazon-efs-with-eks-in-terraform) : file system & mount target

[AWS Official Docs](https://docs.aws.amazon.com/ko_kr/efs/latest/ug/performance.html#performancemodes) : EFS Preformace & Throughput mode

[Devsisters Technical Blog](https://tech.devsisters.com/posts/pod-iam-role/) : IRSA

[Terraform Official Docs](https://registry.terraform.io/providers/hashicorp/external/latest/docs/data-sources/external) : Terraform data.external 리소스

[Github](https://gist.github.com/riccardomc/a71e14bf9c9a45632185a1445ef1ee03) : thumbprint script

[Terraform Official Docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_addon#service_account_role_arn) : Terraform aws_eks_addon 리소스

[Terraform Official Docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy#policy) : Terraform import 블록을 사용한 IAM Policy 설정

[Medium](https://vipulvyas.medium.com/a-comparison-of-aws-storage-s3-vs-ebs-vs-efs-c5cc95834034) : S3 vs EBS vs EFS
