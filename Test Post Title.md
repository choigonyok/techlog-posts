[ID: 46]
[Tags: test]
[Title: Test Post Title]
[WriteTime: ]
[ImageNames: ]

aesg

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