[ID: 0]
		[Tags: ISTIO]
		[Title: Envoy 아키텍처 & Request-Response Flow]
		[WriteTime: 2024-01-08]
		[ImageNames: 35d0b88d-37da-44bc-89e5-9914915bdeea.png fbf78fc3-5453-4a26-aadd-3b575e2a0c4b.png]
		
		
## Content

1.  Preamble
2. Terminology
3. Listener
4. Architecture
5. Request Flow
6. References

## 1. Preamble

> 이 글은 [Life of a Request - Envoy Official Docs](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request#listener-filter-chains-and-network-filter-chain-matching)를 기반으로 한 글이다.


Envoy에는 리스너, 필터 체인, 엔드포인트 등 다양한 컴포넌트들과 클러스터, 업/다운스트림, 루트 등 여러 용어들이 있다. 이 용어들에 대해 자세히 알아보고 엔보이를 통한 서비스 메시를 구현했을 때의 요청/응답 플로우를 확인해보려고 한다.

## 2. Terminology


**Cluster**: 엔보이 프록시가 요청을 포워딩하는 서비스

**Downstream**: 엔보이와 연결되어있는 엔티티, 엔보이가 실행되고있는 서버를 의미한다.

**Endpoints**: 서비스를 구성하는 네트워크 노드, 여러 엔드포인트가 모여서 클러스터를 구성한다. 클러스터의 엔드포인트들을 엔보이 프록시의 upstream이라고 말한다.

**Filter**: 요청에 대한 작업(확인, 조작 등)을 수행하는 모듈

**Filter chain**: Filter 파이프라인

**Listeners**: 엔보이 프록시에서 IP와 port 바인딩을 담당하는 모듈, 새로운 TCP/UDP 커넥션을 맺고 요청단계에서 다운스트림들을 관리한다.

**Upstream**: 엔보이가 목적지 서비스로 요청을 포워딩할 때 연결하는 엔드포인트

**HTTP/2 코덱**: 요청/응답 데이터를 frame 단위로 나누고 다중화시키는 프로토콜

**Transport 소켓**: 데이터 전송에 사용되는 소켓

## 3. Listener


기본적으로 리스너는 **Ingress**, **Egress** 리스너가 있다. 인그레스 리스너는 외부에서 업스트림으로 요청/응답을 할 때 바인딩을 하고, 이그레스 리스너는 업스트림에서 다운스트림으로 요청/응답을 할 때 바인딩을 한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6FCC0DD3-C585-412F-A551-0D75ECB1D06A/FEB3FFFC-7AA9-4F44-8F03-F45F0EDA9CB4_2/kw2Rr6F6CEl9I7vX60AkA5TVpPnTVRnvDfUusrnjDjkz/Image.png)

하나의 엔보이 인스턴스에는 여러 리스너가 존재할 수 있고, 꼭 사이드카 프록시가 아니더라도 아래 사진처럼 내부적인 로드밸런서로 활용될 수도 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6FCC0DD3-C585-412F-A551-0D75ECB1D06A/0E40656B-49FE-4216-ABBB-D66DCE5F79E4_2/Nc5BPJWsg24hDbFB4dDIUxZ9bWwNWkk6dBkHNyuTY60z/Image.png)

아래 사진은 사이드카 프록시, 내부 로드밸런서, 엣지 인그레스, 엣지 이그레스로 엔보이 프록시가 다양하게 활용되는 예시이다. Istio의 경우에 가장 왼쪽의 엣지 인그레스가 Istio-ingressgateway의 역할이다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6FCC0DD3-C585-412F-A551-0D75ECB1D06A/1D882928-70BC-4B0D-AF21-AFB35B32F6C9_2/oGJkmxJ7S3tAKMCf1fA23MNOr1CNt6sAOWTHaLdRnxgz/Image.png)

## 4. Architecture


엔보이의 아키텍처는 크게 두 개의 서브시스템으로 구분할 수 있다.


- Listener


>  다운스트림에서 오는 요청을 담당하며, 다운스트림에서의 요청 라이프사이클과 클라이언트로의 경로를 관리한다.

- Cluster


>  업스트림 커넥션에 대한 구성을 담당한다. 클러스터와 엔드포인트에 대한 헬스 체크, 로드밸런싱, 커넥션 여부 등을 관리한다.


이 두 서브시스템은 HTTP router 필터로 연결되어있다. 이 router 필터는 다운스트림에서 업스트림으로 요청을 포워딩한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6FCC0DD3-C585-412F-A551-0D75ECB1D06A/DB7114F5-63BF-4F8F-AE09-B6E2D5085748_2/S0KRMNMuyXGJ9TkEOSVZBDLxgJxcm1LvnvIu7ftwgZ8z/Image.png)

## 5. Request Flow


요청 흐름은 11단계로 설명할 수 있다.


1. 엔보이의 worker 쓰레드에서 실행되고있던 리스너에 의해 다운스트림과의 TCP 커넥션이 시작된다.
2. 리스너 필터체인이 생성되고 시작된다. SNI와 preTLS 정보를 제공할 수 있고, 이후에 리스너가 네트워크 필터체인과 매치시킨다. 각 리스너는 여러 필터체인을 가질 수 있고, transport 소켓은 이 필터체인과 관련되어있다.
3. TLS transport 소켓이 TCP 커넥션으로부터 읽어들인 데이터를 복호화한다.
4. 네트워크 필터체인이 생성되고 실행된다. 이 필터체인 중 가장 중요한 **HTTP connection manager**는 필터체인에서 마지막 네트워크 필터이다.
5. HTTP connection manager의 HTTP/2 코덱은 TLS 커넥션에서 복호화된 데이터스트림을 수많은 독립적인 스트림으로 분배한다. 각 스트림은 단일 요청/응답을 다루게된다.
6. 각 HTTP 스트림인 **Downstream HTTP 필터체인**이 생성되고 실행된다. 요청은 처음에 요청을 읽고 수정하는 CustomFilter를 거치고, 트래픽을 전달할 route와 클러스터가 선정되는 가장 중요한 HTTP 필터인 **router 필터**도 거친다. 스트림의 요청 헤더는 클러스터의 업스트림 엔드포인트로 포워딩된다. router 필터는 HTTP connection pool을 가지고있다.
7. 엔드포인트를 찾기위해 클러스터로의 로드밸런싱이 수행된다. 그리고 새로운 스트림이 허용되는지를 결정하기 위해 클러스터의 서킷 브레이커가 체킹된다. 만약 엔드포인트의 커넥션 풀이 비어있거나 capacity가 적으면 새로운 엔드포인트로의 커넥션이 생성된다.
8. 각 스트림에서 **Upstream HTTP 필터체인**이 생성되고 실행된다. 기본적으로 이 필터에는 CodecFilter만 포함되어있다. CodecFilter는 데이터를 적절한 코덱으로 보내는 필터이다. 만약 클러스터가 Upstream HTTP 필터체인 구성이 되어있으면 이 필터체인은 각 스트림에서 생성되고 실행되며 
9. 업스트림 엔드포인트 커넥션의 HTTP/2 코덱은 요청의 스트림을 단일 TCP 커넥션에서 업스트림으로 전달되는 다른 스트림들에 함께 전송된다.
10. 요청은 업스트림으로 프록시되고, 응답은 다운스트림으로 프록시된다. 응답은 요청과 반대되는 순서로 HTTP 필터들을 거친다. 다운스트림에 응답이 전송되기 전에 코덱 필터에서 시작해서, 다른 업스트림 HTTP 필터들과, router 필터, CustomFilter를 거치게 된다.
11. 응답이 완료되면, 스트림이 destroy된다. 요청 플로우가 끝난 이후에는 이후에는 상태를 업데이트하고 엑세스 로그를 작성하며, 트레이스 span을 완성시키는 등의 프로세싱을 한다.

### Listener TCP accept


ListenerManager가 리스너 인스턴스를 구성하고 생성한다. 리스너는 Warning / Active / Draining 상태 중 하나를 가진다. Warning은 의존성 구성을 기다리고 있는 상태를 의미하고, Draining은 더 이상 새로운 TCP 연결을 맺지 않는 리스너의 상태를 의미한다.

엔보이 프록시는 멀티 쓰레딩을 사용하는데, master 쓰레드는 엔보이 인스턴스 자체의 라이프사이클을 관리하고, 내부적으로 수많은 worker 쓰레드들에서 각 리스너 인스턴스들이 생성되고 유지된다. 여러 리스너가 같은 리스너 포트나 또는 같은 포트의 같은 소켓을 공유할 수도 있다.

TCP 커넥션이 도착하면 어떤 worker 쓰레드에서 커넥션을 받아들이고 리스너를 생성할지는 커널이 결정한다.

### 리스너 필터체인과 네트워크 필터체인 매칭


Worker 쓰레드에서 리스너가 생성되면 리스너가 **리스터 필터체인**을 생성하고 실행한다. 이 필터체인에는 **TLS inspector 필터**가 포함되어있는데, 이 필터는 TLS 핸드셰이크가 이루어졌는지 체크하고, 이루어졌다면 SNI을 추출한다.

SNI는 Server Name Indication의 준말로, 클라이언트가 서버에 어떤 호스트명으로 연결하려는지를 알려주는 역할을 한다. SNI는 하나의 IP로 여러 도메인을 관리할 때 유용하게 사용될 수 있는데, TLS 핸드셰이크 단계에서 SNI없이 클라이언트가 IP주소만 서버에 전달할 경우, 서버는 여러 도메인들 중 요청이 어느 도메인과 연관되어있는지를 알 수가 없게된다.

SNI

리스너 필터체인에 TLS inspector 필터가 존재하면, 언제든 리스터 필터체인에서 SNI/ALPN이 필요할 때 해당 데이터를 삽입해줄 수 있게된다. SNI/ALPN을 통해 어떤 필터체인에 요청을 매칭시켜야하는지를 알 수 있게해주고, 이 커넥션을 다루기위한 네트워크 필터체인과 transport 소켓을 전달할 수 있게된다.

엔보이에서 network filter chain matching이라는 건 TLS inspector 필터를 통해 추출된 SNI를 활용해서, 수많은 네트워크 필터체인 중 어느것과 해당 요청을 매칭시킬지 결정한다는거야? 만약 이 명제가 맞다면, 같은 IP와 포트로 오는 모든 요청은 하나의 리스너가 수신해서, 하나의 리스너 필터체인을 거치게 되고, 리스터 필터체인 마지막의 TLS inspector를 통해 추출된 SNI에 맞게 각각 알맞은 네트워크 필터체인으로 요청이 로드밸런싱된다는건가?

HTTP에서는 TLS 핸드셰이를 안하고, 따라서 SNI도 없고, 서브도메인처럼 하나의 IP에서 여러 도메인을 사용하고싶다면 추가적인 헤더를 넣어서 요청이 어느 도메인을 위한 것인지를 명시해야한다고 한다. 왜?

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6FCC0DD3-C585-412F-A551-0D75ECB1D06A/7D797B3A-E96E-485C-89CD-BB0908D6D868_2/O7YKjXCaDjoM40H8V7Iy6Pc0UCiiyfBkkcaLeyfTCwkz/Image.png)

### TLS Transport socket 복호화


엔보이 프록시는 TCP 커넥션 라이프사이클의 이벤트에 따라 네트워크 버퍼에 읽고 쓸 수 있는 transport socket을 제공한다.

TLS inspector 필터에서 TLS transport 소켓을 트리거시키고, 이 소켓이 TLS 핸드셰이크를 수행한다. 핸드셰이크가 완료되면 복호화된 요청 데이터(스트림)이 네트워크 필터체인을 관리하는 인스턴스에 보내진다.

### 네트워크 필터체인 프로세싱


리스너 필터체인에서 네트워크 필터들을 생성한다. 

네트워크 필터들은 세 가지로 구분된다.


- ReadFilter

데이터가 커넥션에서 사용가능해지면 호출된다.


- WriteFilter

데이터가 커넥션에 Write될 예정일 때 호출된다.


- Filter

ReadFilter와 WriteFilter를 구현한다.

네트워크 필터의 예시로는 RateLimit이나 RBAC 등을 예시로 들 수 있다.

네트워크 필터체인의 마지막 필터는 HCM(HTTP Connection Manager)이다. 이 필터는 HTTP/2 코덱을 생성하고 HTTP 필터체인을 관리한다. 

### HTTP/2 코덱 디코딩


요청 데이터는 TLS transport 소켓에서 복호화되기 때문에 HCM에 의해 생성된 HTTP/2 코덱에 요청이 전달될 때는 평문(Plaintext)로 전달된다. HTTP/2 코덱은 하나의 바이트 스트림 형태의 요청을 여러 HTTP/2 frame 형태로 디코딩하고, 각 HTTP 스트림으로 커넥션을 나눈다.

이 과정을 스트림 멀티플렉싱이라고 하고, 이 기능이 없는 HTTP/1에 비해 성능 향상이 많이 되기 때문에 HTTP/2의 주요 기능이라고 할 수 있다.

이렇게 나누어진 각 HTTP 스트림이 개별적으로 요청/응답을 수행하게된다.

### HTTP 필터체인 프로세싱


HCM은 각 HTTP 스트림에 **Downstream HTTP 필터 체인**을 생성한다. HTTP/2 코덱을 통해 스트림으로 나뉘어진 데이터는 우선 CustomFilter를 통과한 후 router 필터를 통과한다.

인코딩/디코딩 관련해서 세 가지 필터가 있다.


- encoder-decoder
- encoder
- decoder

요청 트래픽이 포워딩될 때는 encoder 필터는 bypass되고, 응답 트래픽이 포워딩될 때는 decoder 필터가 bypass 된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6FCC0DD3-C585-412F-A551-0D75ECB1D06A/4CFA0AC7-05ED-407C-8657-778B5433A4AC_2/G466CxVRxaEUcEVb153la83DZJco2eWqRygmGmN4ihIz/Image.png)

요청 트래픽 관점에서, router 필터에서 decoder가 호출되면 루트(route)와 cluster가 선정된다. HCM이 HTTP 필터체인 실행 시작단계에서 RouteConfiguration에 따라서 루트를 선정한다.이걸 cached route라고 한다. 필터들은 요청 헤더를 수정시킬 수 있고 새로운 루트를 선정할 수 있다.
>  캐시된 루트대로 HTTP 필터체인 시작단계에서 우선 루트를 선정해놓고, 중간에 필터를 거치면서 HCM에게 루트를 초기화하고 재선정하도록 요청할 수 있다.


router 필터가 호출되면 루트가 최종 결정된다. 이렇게 선정된 루트의 구성(configuration)은 업스트림 클러스터의 이름을 가리킨다. 그럼 라우터 필터가 ClusterManager에게 HTTP 커넥션 풀 할당을 요청한다. 여기서 로드밸런싱과 커넥션 풀 요청이 포함된다.

HTTP 커넥션 풀은 HTTP 요청에 대한 인코딩/디코딩을 캡슐화한다. 스트림이 HTTP 커넥션 풀을 할당받으면 요청 헤더가 업스트림 엔드포인트로 포워딩된다.

라우터 필터는 전체적인 업스트림 요청 라이프사이클 관리를 담당하고, 요청에 대한 타임아웃이나 재시도, 어피니티 등도 담당한다.

그리고 라우터 필터는 **Upstream HTTP 필터체인**을 담당한다. 기본적으로, 업스트림 HTTP 필터체인은 라우터 필터에 요청 헤더가 도착하자마자 즉시 실행된다. 근데 C++ 헤더는 업스트림 커넥션이 준비될 때까지 업스트림 HTTP 필터체인을 잠시 정지시킬 수 있다.

업스트림 HTTP 필터체인은 기본적으로 클러스터 구성을 통해 설정할 수 있다. 다운스트림 HTTP 필터와 다르게 업스트림 HTTP 필터는 루트를 대체할 수 없다.?

### 로드밸런싱


각 클러스터에는 새로운 요청이 도착했을 때 엔드포인트를 선정하는 **로드밸런서**가 존재한다. 엔보이는 다양한 로드밸런싱 알고리즘을 지원한다.

엔드포인트가 로드밸런서에 의해 선정되면, 요청을 포워딩할 커넥션을 찾기위해 커넥션 풀을 사용한다. 커넥션이 아예 없거나, 모든 커넥션이 과부하 상태이면 새로운 커넥션이 생성되고 커넥션 풀에 추가된다. 만약 서킷 브레이커가 필요한 경우, 해당 커넥션을 drain하고 새로운 커넥션을 생성하는 식으로 실행된다.

### HTTP/2 코덱 인코딩


선정된 커넥션의 HTTP/2 코덱은 같은 업스트림으로 가는 요청 스트림들을 하나의 TCP 커넥션으로 모은다. HTTP/2 코덱 디코딩이 스트림으로 나눈 것과 반대로, 나뉘어져있는 걸 다시 모으게된다.

### TLS transport 소켓 암호화


업스트림 엔드포인트 커넥션의 TLS transport 소켓은 HTTP/2 코덱 인코딩의 결과물로 출력된 바이트 데이터를 암호화한다. 그리고 암호화된 데이터를 업스트림 서비스의 TCP 소켓에 write한다.

### 응답


헤더, 본문 등을 포함한 요청은 업스트림으로 프록시되고, 응답은 다운스트림에 프록시된다.

응답은 HTTP와 네트워크 필터 등을 요청의 반대 순서대로 거치게된다.

### Post-request 프로세스


응답이 완료되면, 스트림이 destroy된다. 요청 플로우가 끝난 이후에는 이후에는 상태를 업데이트하고 엑세스 로그를 작성하며, 트레이스 span을 완성시키는 등의 프로세싱을 한다.

## 6. References


[Life of a Request - Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)
