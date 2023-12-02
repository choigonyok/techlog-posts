[ID: 36]
[Tags: KUBERNETES AWS]
[Title: 내가 느낀 EKS와 Kubeadm의 차이점]
[WriteTime: 2023/11/20]
[ImageNames: ]

## Contents

1. Preamble
2. EKS vs Bare-Metal
3. kubectl
4. eksctl
5. IRSA w/ EBS CSI
6. Control Plane
7. Summary

## 1. Preamble


EKS를 언젠간 깊게 학습해야지 계획은 하고있었다. 블로그의 CI/CD 파이프라인을 구축하다 EKS에서 제공하는 IRSA를 사용하기 위해 결국 베어메탈로 구성되어있던 쿠버네티스를 EKS로 이전하게 되는 상황까지 일어났고, EKS에 대해 학습하게 되었다.

EKS를 이제 막 알아가고 있음에도 불구하고 장점이 상당히 많고, 베어메탈에 비해 훨씬 편리하다. 아직 내가 모르는 장점들 역시 많을 것이라고 생각한다.

내가 느낀 베어메탈 클러스터와의 차이점에 대해 정리해보려고 한다.

## 2. EKS vs Bare-Metal


### Loadbalancer type Service


쿠버네티스의 Service 오브젝트에는 ClusterIP / Loadbalancer / NodePort 세 가지 타입이 있다. ClusterIP는 쿠버네티스 클러스터 내부에서 서비스 간 통신을 위한 Private IP 주소이고, NodePort는 외부에서 해당 서비스에 접근할 수 있는 포트를 열어두는 서비스이다. Loadbalancer는 클라우드 벤더에서 제공하는 External-IP를 할당받을 수 있는 서비스이다.

난 kubeadm으로 첫 쿠버네티스 클러스터를 구성했었다. kubeadm은 베어메탈 쿠버네티스 클러스터를 구성할 수 있게 해주는 도구이다. 베어메탈 쿠버네티스 클러스터는 클라우드 기반의 VM, Continer 위에서 구성되지 않고 물리적인 서버에 구성되는 클러스터를 의미한다. kubeadm을 이용하면 직접 온프레미스 서버 위에 쿠버네티스를 구성할 수도 있고, 베어메탈의 의미와는 조금 다르지만 클라우드 서비스 위에도 쿠버네티스를 구성할 수 있다.

처음 어플리케이션을 배포하기 위해 외부에서 클러스터로 들어오는 접근을 받으려고 Loadbalancer type Service를 생성하려 했다. 그러나 External-IP가 할당되지 않는 문제를 겪었다. 이후 알고보니 Loadbalancer type Service는 클라우드 서비스에서만 사용가능한 타입이었고, kubeadm으로 클러스터를 구성한 나는 결국 Public Subnet을 구성한 뒤 NodePort를 열어 외부에서 오는 접근을 가능하게 했다.

EKS도 쓰기만 한다고 Loadbalancer 타입 서비스에 External-IP를 할당할 수 있는 것은 아니다. aws-load-balancer-controller를 배포하고, ServiceAccount와 IAM Role을 연결해서 AWS API에 요청을 보내 AWS ELB를 생성한 후 할당된 IP와 도메인을 Loadbalancer 타입 서비스에 연결하는 방식이다.

그냥 NodePort를 열면 되는데 왜 AWS ELB를 위한 비용을 지불해가면서까지 Loadbalancer 타입 서비스를 생성하는 걸까? 답은 보안에 있다.

NodePort를 열어 외부의 접근을 받는 방식은, node의 Public IP + NodePort의 조합을 통해 접근하는 방식이다. 즉 NodePort를 통한 외부 접근을 위해서는 기본적으로 node인 인스턴스가 Public Subnet에 생성되어야한다. NodePort가 아닌 Loadbalancer 타입 서비스를 사용하게 되면, Public Subnet에는 AWS ELB만 위치해서 외부에 접근을 받고, 쿠버네티스 클러스터 전체는 Private Subnet에 생성될 수 있어서 보안적으로 조금 더 안전하다고 볼 수 있다. 

다만 Loadbalancer를 사용하면 추가적인 비용이 든다는 것과 aws-load-balancer-controller에 대한 의존성이 생긴다는 것이 단점이다.

## 3. kubectl


kubeadm으로 클러스터를 구성했을 때는 클러스터를 관리하기 위해 직접 컨트롤 플레인에 접근해야 kubectl CLI를 사용할 수 있었다. EKS에 대해 가장 놀라웠던 것이 이 부분인데, EKS에서는 EKS context를 추가하면 로컬에서도 kubectl로 클라우드에 구성되어있는 쿠버네티스 클러스터를 관리할 수 있다.

```dockerfile
aws eks --region REGION update-kubeconfig --name CLUSTER_NAME
```


이 커맨드 한 줄이면 가능하다. 내부적으로 AWS CLI Configuration에 등록한 크리덴셜의 IAM 권한을 바탕으로 원격 관리가 가능한 것 같다. 사실 이 기능 하나만으로도 EKS를 쓸만한 이유가 충분히 되는 것 같다. 그 정도로 쿠버네티스 관리 복잡성이 상당히 줄어들었다.

## 4. eksctl


EKS 클러스터 생성 및 관리를 위해 eksctl를 제공한다. 베어메탈 클러스터를 구성할 때 일일이 컨테이너 런타임 설치하고, kubectl 설치하고, 컨테이너 간 격리를 위해 시스템 콜 호출 및 스왑 해제시키고, cni 설치하고, iptable과 네트워크 브릿지 설정하고, 워커노드들을 클러스터에 join 시키기 위해 일일이 접속해서 커맨드 실행했던 것과 달리 단 한 줄의 커맨드면 EKS 클러스터를 생성할 수 있다.

## 5. IRSA w/ EBS CSI


IRSA가 EKS로 어플리케이션을 이전하게 만든 가장 결정적인 원인이었다. IRSA는 IAM Role for Service Account의 준말로, 쿠버네티스 리소스인 서비스 어카운트와 AWS의 IAM Role을 연결할 수 있게 해준다. IAM Role 설정을 통해 권한이 부여된 AWS API를 호출할 수 있게되고, ServiceAccount 리소스와 연결된 Pod에서 AWS의 리소스들에 접근할 수 있게된다.

AWS의 Elastic Block Storage(EBS)을 생성해서 Pod의 PersistenceVolume(PV)로 사용하기 위해 aws-ebs-csi-driver를 사용해야했다. 쿠버네티스에서는 다양한 벤더의 스토리지를 사용하기 위해 기존의 플러그인 형식 대신 CSI(Container Storage Interface)를 채택했다. 이를 통해 쿠버네티스 코드베이스의 의존성을 갖지 않고, 각 벤더가 더 적은 비용으로 쿠버네티스를 위한 자사의 스토리지 서비스를 제공할 수 있게 했다.

EBS CSI Driver는 Pod 형태로 클러스터에 배포되는데, AWS EBS에 접근하기 위해서는 AWS API에 요청을 보낼 수 있는 권한이 필요하다. 그 권한을 얻기 위해서는 위에 언급한 것처럼 IAM Role에 EBS 스토리지 관련 정책을 부착한 뒤, 해당 IAM Role을 Pod가 가질 수 있게 해야하는데, 이것을 간편하게 가능하게 해주는 것이 IRSA이다. 

IRSA는 EKS에서만 사용이 가능한 기능이다. EKS가 아닌 쿠버네티스 클러스터에서 IAM Role을 Pod 또는 Node와 연결하기 위해서는 별도의 도구를 사용해야한다. kube2iam이나 kiam 같은 오픈소스가 그 예시인데, kiam은 IRSA가 생긴 이후로 Deprecated 되었고, kube2iam은 Pod가 Crash되는 이슈가 많아서 선택지가 IRSA 밖에 남아있지 않았다.

kube2iam의 방식을 살펴보면 우선 전체 클러스터에 동일한 IAM Role을 부여한다. 그리고 kube2iam을 DaemonSet으로 배포해서 모든 AWS API 호출을 인터셉트하게 된다. 특정 annotation이 존재하는 Pod만 AWS API로의 요청을 허용하는 방식으로 IAM Role을 관리한다.

kube2iam가 권한을 제어하는 방식이라 그냥 전체 클러스터에 IAM Role을 부여하면 EKS 없이도 구현이 가능할 것 같긴 한데, 보안적으로 옳지 않은 방향이라 고민 대상에서 제외되었다.

## 6. Control Plane


쿠버네티스 클러스터를 구성하는 API server, controller, etcd, scheduler 등의 Core Pod들은 대부분 마스터 노드에 모여있고, 이 Core Pod들의 논리적인 집합을 Control Plane이라고 한다.

EKS에서는 컨트롤 플레인의 HA(High Availability: 고가용성) 보장을 위해 Default로 3개의 마스터 노드를 각각 다른 AZ(Availability Zone: 가용 영역)에 배치해 운영해준다. 다른 가용영역에 위치하는 이유는 천재지변 등으로 인해 특정 가용지역 전체에 장애가 생겨도 다른 가용지역의 마스터 노드가 그 역할을 대신할 수 있게하기 위해서이다.

베어메탈 쿠버네티스에서 이런 고가용성 보장은 모두 직접 구현해야한다. RAFT 등의 리더 선출 알고리즘을 선정하고, 서로 다른 가용영역에 마스터 노드를 스핀업해야한다. EKS는 이 과정을 알아서 해준다. 

단점으로는 마스터 노드의 개수를 임의로 조정할 수가 없다. 하나만 쓰고 비용을 줄일 수도 없고, 더 많이 쓰고 비용을 많이 낼 수도 없다.

또 ETCD를 추가적으로 고가용성 운용하는 것이 불가능하다. 기존 kubeadm을 통한 쿠버네티스 클러스터를 구성했을 당시, HAProxy를 로드밸런서/리버스프록시로 사용하고, ETCD 3개를 RAFT 알고리즘으로 구성해 ETCD의 고가용성을 보장한 적이 있다. EKS의 경우에 ETCD만 따로 분리해 운용하는 것이 불가능하고, 컨트롤 플레인 전체가 Stacked 형태로 고가용성 운영되는 방식만 가능하다.

## 7. Summary


EKS는 참 좋고 편리한 서비스이다. 비용이 추가적으로 생기긴 하지만, 하나의 클러스터마다 시간당 0.1USD가 지불되는 것이 기업 입장에서 크게 부담되는 금액은 아니라고 생각하고, 여차하면 IAM를 적절히 사용해서 하나의 EKS 안에서 여러 서비스들을 구성할 수도 있다.

비용보다도 EKS, 더 나아가서 AWS에 대해 너무 커다란 의존성이 생기는 것이 불편했다. 안그래도 쿠버네티스로 서비스를 배포하기 위해서 수많은 오픈소스 소프트웨어의 의존성을 갖게되는데, 뿌리라고 할 수 있는 클러스터 구성에서부터 큰 의존성을 갖고 시작하는 것 같다.
