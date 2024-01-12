[ID: 55]
		[Tags: KUBERNETES ISTIO NETWORK]
		[Title: CNI에 대해 (Flannel & Calico)]
		[WriteTime: ]
		[ImageNames: ]
		
		## Content 

1. Preamble
2. CNI란
3. Flannel & Calico 비교
4. References

## 1. Preamble


Istio 깃허브 레포지토리에 CNI 관련 이슈들이 많이 생성되어있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/7CEBB326-C284-4E74-A858-05E92BC8C94E_2/NSaONxW8NRRJAiwccEo8fGUHHT2Av4hKzyxb1qSeBCkz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/D360C969-F8AA-4618-9E01-E16A0B352E57_2/W7VOatUR4jhtmBofpePo26Krc76lyqAYUHLRql2dCWsz/Image.png)

CNI는 이전에 kubeadm으로 쿠버네티스 클러스터를 구축할 때도 Calico를 사용해서 클러스터를 구축했었다. 구현을 했지만 기술부채로서 남아있었기 때문에, Istio-cni에 대해 공부하기 이전에 CNI 자체에 대해서 먼저 정리해보려고한다.

## 2. CNI란


CNI는 Container Network Interface의 준말로, Istio, Kubernetes처럼 CNCF 오픈소스 프로젝트이다. 컨테이너 환경에서 네트워킹을 제어할 수 있게하는 하나의 표준으로서 사용되고있다.

CNI는 리눅스 컨테이너에 네트워크 Config를 할 수 있는 여러 라이브러리들로 구성되어있다.

CNI는 캡슐화가 된 네트워크 모델인 VXLAN(Virtual Extensible LAN)이나, 캡슐화가 되지 않은 네트워크 모델인 BGP(Border Gateway Protocol)를 사용해서 네트워크를 구현한다.

### 캡슐화가 된 네트워크 모델


캡슐화는 데이터 패킷을 IP 헤더로 감싸는 방식이다. 

논리적인 L2 네트워크 캡슐화를 제공한다. 

이 모델을 통해서 분산 라우팅을 하지 않고도 컨테이너마다 분리된 L2 네트워크를 가질 수 있게된다. 프로세싱과, overlay 캡슐화로부터 생겨난 IP 헤더로부터 발생한 IP 패키지 사이즈의 증가 관점에서 오버헤드를 최소로 유지할 수 있다.

캡슐화된 정보는 각 쿠버네티스 워커노드의 UDP 포트로 분산된다. 그리고 어떻게 MAC 주소가 도달할 수 있는지에 대한 네트워크 컨트롤 플레인 정보를 교환한다. 일반적으로 캡슐화는 VXLAN, IPSec, IP-in-IP에서 사용된다.

간단히 하면, 이 네트워크 모델은 쿠버네티스 워커 노드 간의 네트워크 브릿지를 생성한다. 

이 모델은 확장된 L2 브릿지가 선호될 때 사용된다. 그리고 쿠버네티스 워커 노드의 L3 네트워크 레이턴시에 민감하다. 만약 데이터센터가 다른 리전에 위치해있다면 최대한 낮은 레이턴스를 유지하도록 신경써야한다. 아니면 결국에는 언젠가 네트워크 Segmentation이 발생하게 될 것이다.

이 모델을 사용하는 CNI 네트워크 프로바이더는 Flannel, Cilium 등이 있다. 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/A18FC5EE-0F47-4255-AF26-CB9342418B62_2/W3oV0NBjNEJLVdgIUBKjbYbHaBgPoC8ynywXtkBiVXQz/Image.png)

### 캡슐화가 되지 않은 네트워크 모델


이 모델은 컨테이너 간에 패킷을 라우팅할 수 있도록하는 L3 네트워크를 제공한다. 이 모델은 캡슐화 네트워크 모델과 달리 분리된 L2 네트워크를 생성하지도 않고, 따라서 오버헤드도 생성되지 않는다.

오버헤드가 생기지 않기 때문에 트래픽 여러 컨테이너로 트래픽을 분산해서 라우팅해야하는 쿠버네티스 워커 노드에서 비용을 아낄 수 있다.

캡슐화를 위해 IP 헤더를 사용하는 대신, 이 네트워크 모델은 각 파드에 트래픽을 분산시키기 위해 BGP 등의 네트워크 프로토콜을 사용한다.

간단히 요약하면, 쿠버네티스 워커노드 간 통신을 위해, 어떻게 파드에 접근해야하는지에 대한 정보를 제공하는 네트워크 라우터를 생성하는 모델이다.

L3 네트워크가 선호될 때 이 모델이 주로 사용되고, 이 모드는 OS 레벨에서 동적으로 루트(route)를 업데이트한다. 이 방식이 레이턴시가 덜 생긴다.

대표적으로 Calico가 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/AEF13B5D-D85F-454F-8E19-41A7393A9B5A_2/AucLdIpPyIlLUaZW1Uoq21HsQdvTCFkGHyBHcmW7y7kz/Image.png)

## 3. Flannel & Calico 비교


### Flannel


Flannel은 쿠버네티스에서 L3 네트워크를 구성하기 쉽다. flanneld 라는 바이너리 에이전트를 각 호스트에 실행시킨다.

Flannel은 Kube API나 etcd를 사용해서, 네트워크 설정이나 할당된 서브넷 정보나, 호스트 IP 등의 데이터들을 직접 저장한다.

패킷들이 포워딩되는 방식은 여러가지인데, Default로는 VXLAN 캡슐화를 통해 포워딩된다.

보통 캡슐화된 트래픽은 암호화되어있지않다. 그래서 flannel은 캡슐화된 패킷을 암호화하기위해


- 쿠버네티스 워커노드 간 암호화된 IPSec 터널을 생성하기 위해 strongSwan을 사용하거나,
- strongSwan보다 조금 더 높은 퍼포먼스의 WireGuard를 사용한다.

쿠버네티스 워커노드들은 8473번 UDP 포트를 열어야한다. 이 포트가 VXLAN에 사용된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/A65A28DB-E8D2-469D-BC00-52EBE4DC7054_2/NAfokxSxg6MjYJOBV6QyBV9AcxxHXfFCzc4gQcAC7O4z/Image.png)

### VXLAN


VXLAN은 VLAN(Virtual LAN)의 확장형(X: Extension)으로, 네트워크 가상화 표준이다.

VXLAN을 통해 하나의 물리적인 네트워크를 공유하면서 하나의 테넌트가 다른 테넌트의 네트워크 트래픽을 볼 수 없도록 멀티테넌시를 구축할 수 있다.

1600만개의 가상 네트워크를 만들 수 있고, 각 네트워크는 1~약 1600만 까지의 정수로 설정되어있는 세그먼트 ID로 구분된다. 데이터를 보내는 곳에서 캡슐화를 할 때 추가적인 레이어에서 세그먼트 ID, 송신자 IP, 수신자 IP가 들어있는VXLAN 헤더를 추가해서 Frame을 만든다.

이 프레임은 추가적인 캡슐화로 패킷이 되어 수신자 IP로 보내지고, 수신한 서버에서 `역캡슐화`를 통해 세그먼트 ID를 확인하게 되고, 해당하는 세그먼트 ID를 가진 가상 네트워크에 패킷을 라우팅하게 된다.

VXLAN은 어떻게 하나의 IP에서 1600만여개나 되는 가상 네트워크를 생성할 수 있는 것일까?

## Calico


Calico는 클라우드 환경에서 실행되는 쿠버네티스 클러스터에 대해 네트워크를 구성할 수 있고, 네트워크 정책을 설정할 수 있게 해준다.

기본적으로 Calico가 Default로 사용되면 캡슐화되지 않은 IP 네트워크와 정책이 제공된다.

왜 Calico는 클라우드 이야기를 하지? Flannel은 클라우드에서 사용을 못하나? 왜?

클라우드 뿐만 아니라 온프레미스 서버에서도 BGP를 통해 쿠버네티스 워크로드 간의 통신을 가능하게 할 수 있다.

그리고 Calico에서는 `stateless`한 IP-in-IP나 VXLAN 캡슐화 역시 지원하고있다.

쿠버네티스 워커 노드들은 BGP를 사용한다면 TCP 179번 포트를 열어야하고, VXLAN을 사용한다면 4789번 포트를 열어야한다. 그리고 Typha를 사용할 때 5473번 TCP 포트 역시 열어야한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/AEF13B5D-D85F-454F-8E19-41A7393A9B5A_2/AucLdIpPyIlLUaZW1Uoq21HsQdvTCFkGHyBHcmW7y7kz/Image.png)

사진의 `bgp routing`은 BGP 라우터를 의미하는데, Calico의 경우 `Calico Agent`가 된다. 이 BGP 라우터인 Calico 에이전트는 각 워커노드마다 배치된다.

서로 다른 워커 노드에서 파드가 생성될 때 파드의 사설 IP주소가 겹쳐지면 안된다. 당연한 말이지만 그렇게되면 원하는 파드에 패킷을 전달할 수 없게된다. 그래서 각 워커노드는 할당 가능한 `IP주소 Block`을 지정받는다. 이 데이터가 ETCD에 저장되고, Calico Agent는 자기 노드에 파드가 생성되면 ETCD에서 할당가능한 IP주소 블록을 참조해서 파드에 IP주소를 할당한다.

그럼 위 사진과 같이 서로 다른 노드에 있지만 IP주소가 겹치지 않는 4개의 워크로드(파드)가 생성될 수 있다.

Calico Agent는 라우터이기 때문에 자체적으로 라우팅 테이블을 가지고있다. 만약 같은 노드 내에서 파드A가 파드B에게 패킷을 보낸다면, A에서 출발한 패킷이 Calico 에이전트에 수신되고, Calico 에이전트는 라우팅 테이블을 확인해서 파드B에게 패킷을 라우팅하게된다.

만약 노드1의 파드A가 노드2의 파드B에게 트래픽을 전달한다면 어떻게될까?

노드1의 파드A에서 출발한 패킷은 노드1의 BGP 라우터인 Calico 에이전트1에 수신되고, Calico 에이전트1은 자신의 라우팅 테이블을 확인해서 해당 패킷이 Calico 에이전트2에게 전달해야한다는 것을 확인한다. 그리고 전달한다. Calico 에이전트2는 패킷을 수신하고, 마찬가지로 자신의 라우팅 테이블을 확인해서 자기 노드에 배치된 파드B에게 패킷을 전달한다.

에이전트1은 어떻게 에이전트2에게 트래픽을 전달해야한다는 것을 알았을까?

앞서 말했듯이 Calico 에이전트는 각 쿠버네티스 노드마다 하나씩 배치된다. 새로운 워커노드가 추가되면 Calico 에이전트를 새로 생성해서 새로운 노드에 배치한다. 이렇게 새로운 노드가 배치되어 새로운 에이전트가 생성되면, 기존에 존재하던 모든 Calico 에이전트들은 이 새로운 에이전트들과 `BGP 피어링`을 맺는다.

BGP 피어링을 통해서 각 에이전트들은 자신이 가지고있는 라우팅 테이블을 주기적으로 공유해서 자신의 라우팅 테이블을 업데이트할 수 있다. 그래서 앞선 ~~예시에서~~ 에이전트1이 파드B의 IP에 대한 루트(route)를 알고있었던 것이고, 이 루트대로 에이전트2에게 패킷을 전달하게 되었던 것이다.

정리하자면, 

```go
워커 노드  1에 배치된 파드 A가 워커노드2에 배치된 파드 B에게 요청을 보내는 상황

1. 워커노드1의 파드 A에서 출발한 패킷을 워커노드1의 Calico 에이전트가 수신한다.
2. 워커노드1은 BGP 피어링을 통해 워커노드2의 라우팅 테이블을 가져와서 업데이트한다.
3. 워커노드1은 패킷을 워커노드2의 Calico 에이전트에 전송한다. 
4. 워커노드2의 Calico 에이전트는 패킷을 받아서 자신의 라우팅 테이블을 확인한 후에 파드 B에게 패킷을 라우팅한다.
```


Calico 에이전트가 노드 별로 IP주소 블록을 사용해서 파드에 IP를 할당하기 때문에, 같은 노드에 있는 파드끼리는 같은 네트워크를 공유하게된다. 그럼 컨테이너 간 독립성이 유지되지 않는다. 이를 보완하기 위해 Calico에서는 ACL(Access Control List: 접근 제어 목록)을 제공하고, 이를 통해 같은 워커노드 및 네트워크를 공유하는 파드끼리도 격리가 정상적으로 이루어질 수 있게된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/B44ABA17-52D4-4B06-A73A-151A33EEC89B_2/rEcDiJbbBl2G7AfTl9K2TmwFbgoYSCcG0R5gWziGwf8z/Image.png)

### BGP


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/4F6F3A47-33B8-4EE2-85F2-99B41326421A_2/nf8qmyzcdgbDCnx7YOWfRXxIprAD6DnBi1KSEmuMKxoz/Image.png)

BGP는 Border Gateway Protocol의 준말로, 네트워크를 통해 목적지로 가는 최적의 경로를 찾아주는 경로 탐색 규칙을 의미한다.

BGP는 `피어링`을 통해 작동된다. 인접

BGP 라우터는 다른 BGP 라우터와 피어링 관계를 설정한다. 피어링을 통해 인접한 BGP 라우터와 라우팅 테이블을 교환한다.

BGP 라우터는 AS 경계를 감지하고, 목적지 AS를 식별한다. AS 경계에서 라우팅 정보를 교환하고 최적의 경로를 찾기위해 라우팅 업데이트를 수신한다.

BGP 라우터는 자신의 라우팅 테이블을 인접 BGP 라우터에 공유한다.

BGP 라우터가 최적의 경로를 계산한다.

최적 경로가 BGP 라우터의 라우팅 테이블에 업데이트된다.

BGP 라우터가 패킷을 목적지 AS로 전달한다.

하나의 네트워크 안에 여러 AS가 존재하고, 하나의 AS 안에 여러 BGP 라우터가 존재한다.

네트워크를 구성할 때 어떤 AS가 어떤 AS와 인접되어있는지를 설정하게된다.

AS 내부에서의 BGP 라우터 간 연결을 `iBGP` 연결이라고 하고, BGP 라우터를 통한 AS 간 연결을 `eBGP` 연결이라고 한다.

다른 AS로 가야하는 패킷이 있을 때, BGP 라우터는 자신의 라우팅 테이블을 참조해서 최적 경로가 무엇인지를 체크하고 출발지 AS(자신)와 목적지 AS 헤더를 추가한 패킷을 생성해서 알맞은 다음 AS로 패킷을 전송한다.

만약 라우팅 테이블에 목적지 AS에 대한 정보가 없으면, BGP 피어링(인접한 AS)을 통해 자신의 라우팅 테이블을 업데이트한다.

인접한 AS의 라우팅 테이블에서 목적지 AS 관련한 데이터를 가져와서 자신의 라우팅 테이블에 업데이트하고 패킷을 전달한다.

패킷은 라우팅 테이블에 따라 다음 AS로 전달되고, 다음 AS에서 또 그 자신의 라우팅 테이블을 참조해서 또 다음 AS로 패킷을 전달하는 과정이 반복된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1A94A87E-D016-42D3-A2C0-2FE8A082E868/72949939-4EBE-4C78-B8B6-70ACE8197620_2/PCAahMaDcx6TJ06iplyl4t38eOC4k4XVyAo5eF21dQgz/Image.png)

인터넷 네트워크는 여러 AS로 구성되어있다. AS는 `라우팅 정책`이 존재하는 특정 네트워크 그룹을 의미한다. 모든 엔드 디바이스들은 최소 하나의 AS와 연결되어있고, 패킷이 AS에서 다른 AS로 전달되면서 트래픽이 전송된다. AS에서 말하는 `라우팅 정책`은 다른 AS에 대한 정보를 의미한다. 이 라우팅 정책을 지속적으로 업데이트하면서 최적의 경로를 계산한다.

BGP는 AS 간 패킷을 라우팅하기 위한 프로토콜, 규칙이다. 

## Summary


캡슐화된 네트워크 모델은 L2에서 컨테이너 별로 분리된 네트워크를 생성한다. 대신 오버헤드가 발생한다. 최소화가 가능하긴 하다. L2 네트워크라서 파드 간 통신이 아니라 노드 간 통신 중심

캡슐화되지 않는 네트워크 모델은 L3에서 컨테이너 간 라우팅을 가능하게 한다. 네트워크가 분리되지 않고 대신 오버헤드도 생기지 않는다. L3 네트워크라서 노드 간 통신이 아니라 파드 간 통신 중심

쿠버네티스에서는 파드 간 네트워킹을 위해 CNI를 사용한다. 

## 4. References


[Container Network Interface Providers](https://ranchermanager.docs.rancher.com/faq/container-network-interface-providers)

[BGP란 무엇인가요? - AWS](https://aws.amazon.com/ko/what-is/border-gateway-protocol)

[Calico 모듈과 기능](https://coffeewhale.com/packet-network2)

[자율시스템이란? ASN이란? - Cloudflare](https://www.cloudflare.com/ko-kr/learning/network-layer/what-is-an-autonomous-system/)
