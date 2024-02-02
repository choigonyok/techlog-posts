[ID: 51]
[Tags: ISTIO]
[Title: Envoy xDS & gRPC & 프로토콜 버퍼]
[WriteTime: 2024-01-05]
[ImageNames: 3c55c8e2-e27e-4dd7-b841-4b70bfb23e34.png]

## Content

1. Preamble
2. xDS
3. xDS의 장점
4. xDS APIs
5. gRPC & Protocol Buffer
6. Reference

## 1. Preamble


`Istio`에서는 사이드카 프록시로 `Envoy`를 사용한다. 대부분의 Istio 기능은 Envoy를 추상화해서 한 단계 하이 레벨로 제공하지만, EnvoyFilter 등의 Istio CR(Custom Resource)을 통해 엔보이 프록시 기능 자체 또한 사용할 수 있도록 제공하고 있다.

따라서 Envoy에 대해서도 깊은 이해가 필요하고,  Istio 깃허브 이슈에도 `EnvoyFilter`와 관련된 이슈들이 많기 때문에, 컨트리뷰션을 위해 깊게 학습해봐야겠다고 생각했다.

## xDS

엔보이는 Istio에서 서비스 메시를 구현하기 위한 사이드가 프록시로 사용된다. Pod마다 엔보이 프록시가 주입되어 수많은 엔보이 프록시가 배포되는데, 설정 관리를 용이하게 하기 위해 엔보이 프록시는 자체적으로 구성관리를 하지 않고 중앙화된 외부 서버나 파일시스템을 통해 구성을 관리하게된다.

엔보이 프록시는 쿠버네티스의 파드가 쿠버네티스 클러스터에 배치될 때 파드에 주입되어 컨테이너로 배포된다. 이 때 엔보이 프록시는 해당 인스턴스(Envoy Proxy)가 어떤 다른 인스턴스로 트래픽을 전달해야하는지에 대한 라우팅 룰이나 적용해야하는 정책 등에 대한 정보를 필요로 한다.

이 때 Istio 컨트롤 플레인인 Istiod에 요청해서 자신 인스턴스에 적용해야하는 구성(configuration)을 받아오는데, 요청할 때 사용하는 API를 xDS API, 사용하는 프로토콜을 xDS 프로토콜이라고 한다.

즉, 엔보이 프록시가 관리 서버(Istiod)에 구성(Config) 데이터를 요청하는 프로토콜이라고 정의할 수 있다.

xDS 프로토콜의 API는 네 가지로 구분된다.

- ADS (Aggregated Discovery Service)
- CDS (Cluster Discovery Service)
- LDS (Listner Discovery Service)
- RDS (Route Discovery Service)
- EDS (EndPoint Discovery Service)

해당 API들에 대해서는 아래에서 자세히 설명한다.

## xDS의 장점

일반적인 HTTP 프로토콜이 아닌 xDS 프로토콜을 사용하는 이유는 아래와 같다.


- 동적 구성

실시간으로 업데이트된 엔보이 프록시 구성 정보를 가져올 수 있다. 

엔보이에서는 세 가지 방식으로 구성관리를 할 수 있다.

* 파일 시스템

파일 시스템을 watch해서, 마치 핫리로딩처럼 구성 파일이 업데이트되면 업데이트를 감지하고, 업데이트된 구성 내용을 적용시키는 방식이다.

* 관리 서버에 gRPC 질의

Istiod에서 사용하는 방식인데, 엔보이 프록시가 gRPC로 관리 서버에 업데이트된 내용이 있는지를 질의하며, 프로토콜 버퍼로 요청/응답을 주고받는다.

* REST-JSON URL 폴링

엔보이 프록시가 HTTP로 관리 서버에 업데이트된 내용이 있는지를 주기적으로 질의하며, JSON 형식으로 요청/응답을 주고받는다.

기본적으로 이 세 가지 방식 중 xDS 프로토콜은 gRPC를 기반으로 한다. gRPC는 프로토콜 버퍼를 통해 메시지를 직렬화하기 때문에 가볍고 빠르다는 장점이 있다. 프로토콜 버퍼와 gRPC에 대해서는 아래에서 자세히 설명한다.


- 표준화

이 xDS 프로토콜을 통해 여러 다른 서비스 메시 구현 도구들에서도 동일하게 엔보이 프록시가 동작할 수 있다. 이건 xDS 프로토콜만의 장점이라기 보다는 프로토콜 자체의 장점에 가까운 것 같다.


- 확장성

xDS는 푸시 기반 업데이트를 지원한다. 

예를 들어 xDS가 아닌 HTTP로 동일한 구성 데이터를 여러 엔보이 프록시에 전달한다고 가정해보자. 첫 번째 엔보이 프록시에 업데이트된 구성 정보를 보내고, 200 상태코드를 응답받았다. 그 다음 두 번째 엔보이 프록시에 구성 데이터를 보냈는데 마침 두 번째 엔보이 프록시 인스턴스가 배치된 파드가 재시작 중이라면 여기서 많은 시간을 잡아먹게 될 것이다.

xDS에서는 푸시기반이기 때문에 업데이트된 구성 데이터를 푸시만 해두면, 푸시된 구성 데이터를 확인하고 읽어들이는 건 각 엔보이 프록시의 몫이 된다. 따라서 컨트롤 플레인에서는 엔보이 인스턴스가 어떤 상태인지를 구태여 확인할 필요가 없고, 즉 데이터 플레인인 엔보이 프록시가 자유롭게 확장/축소 될 수 있어진다는 것을 의미한다.


- 유연성
- 디버깅/모니터링

xDS API 중 데이터플레인의 디버깅과 모니터링 정보를 조회하기 위한 API가 존재한다. 이 API로 엔보이 프록시의 헬스 체크나 디버깅, 모니터링이 더 간편하다.

##  xDS APIs

- CDS (Cluster Discovery Service)

특정 엔보이 프록시에 대한 업스트림 클러스터를 찾는 API이다. 업스트림 클러스터는 트래픽이 전달되어야하는 목적지 서비스를 의미하고, Istio는 쿠버네티스를 기반으로 하기 때문에, 이 때 업스트림 클러스터는 쿠버네티스 Service 오브젝트라고 할 수 있다.


- LDS (Listner Discovery Service)

각 엔보이 프록시 인스턴스에 구성되어야하는 리스너들을 찾는 API이다. 리스너는 엔보이 프록시로 라우팅된 트래픽을 listen할 네트워크 인터페이스를 의미한다.


- EDS (EndPoint Discovery Service)

각 업스트림 클러스터의 엔드포인드를 찾는 API이다. 엔드포인트는 트래픽이 전달되어야하는 특정 엔보이 인스턴스를 의미한다. 예를 들어 파드1에 컨테이너 A1, 엔보이 A1가 배치되어있고, 파드2에 컨테이너 B, 엔보이 B, 파드 3에 컨테이너 A2, 엔보이 A2가 배치되어있다면 서비스 A에 대한 엔드포인트는 엔보이 A1, A2가 될 것이고, 서비스 B에 대한 엔드포인트는 엔보이 B가 될 것이다.


- RDS (Route Discovery Service)

각 엔보이 프록시에 구성되어야하는 라우트를 찾는 API이다. 라우트는 업스트림 클러스터와 엔드포인트 간에 어떻게 트래픽이 라우팅되어야하는지에 대한 라우팅 규칙을 의미한다. 위 예시를 이어 사용하면, 서비스 A로 트래픽이 전달되었을 때, 해당 트래픽이 엔드포인트인 엔보이 A1, A2에 어떻게 전달되어야하는지에 대한 라우팅 룰을 의미하는 것이다.


- ADS (Aggregated Discovery Service)

CDS, LDS, EDS, RDS로 다양하게 업데이트된 API들을 한 번에 모아서 컨트롤 플레인으로 전달하는 API이다.

## gRPC & Protocol Buffer

### gRPC

`gRPC`는 google Remote Procedure Call의 준말로, HTTP/2를 기반으로하는 원격 프로시저 호출 시스템 프레임워크이다. REST API는 `리소스`에 대한 행위를 기반으로한다. GET /posts 라면, 게시글 전체를 조회한다는 의미가 된다.

이에 반해 gRPC는 리소스 중심이 아니라 메서드 중심이다. 

로컬이 아닌 외부 서비스의 프로시저를 마치 자신의 프로시저인 것처럼 호출해서 사용할 수 있게해주는 프레임워크라고 볼 수 있다.

REST API를 사용해서 MSA를 구성하면 다른 마이크로 서비스에 요청을 보내기 위해서 일반적으로 HTTP 요청을 보내야한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D8BCBE9-3784-4E2D-BC0D-5EBCA2DBF0F4/BD8CA06D-79E9-4C2E-9ADA-BCC6058A9319_2/1OHdMJvNHAXHipGcU5U1ZAKyFcD5HeX1r5AshF5XeHIz/Image.png)

gRPC를 사용하면 다른 마이크로 서비스의 getAlbum 프로시저를 자신의 함수인 것 마냥 호출할 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D8BCBE9-3784-4E2D-BC0D-5EBCA2DBF0F4/8C0848EC-6B31-4832-92DC-118ACDF46367_2/t8oQ8THUZk0PsPJHubmgXPwD9BDWuNrjCFBJBMJAHAUz/Image.png)

gRPC를 활용한 통신의 흐름은 아래와 같다.


1. 프로토콜 버퍼를 생성한다. 
2. 서버와 클라이언트에서 각각 언어에 맞는 프로토콜 버퍼 소스코드와 스텁을 생성한다.
3. 클라이언트에서 서버 스텁을 사용해서 요청 로직 코드를 작성한다.
4. 서버에서 클라이언트 스텁을 사용해서 응답 로직 코드를 작성한다.

### Protocol Buffer


프로토콜 버퍼는 데이터를 바이너리 형식으로 변환해서 직렬화를 가능하게하는 포맷(형식)이다. gRPC에는 컴파일러가 존재한다. 이 컴파일러는 프로토콜 버퍼에 정의된 내용을 컴파일해서 사용하는 언어에 맞게 메서드를 사용할 수 있는 소스 파일을 생성해준다.

다른 언어로 작성된 서버끼리 REST API로 통신을 할 때는 json이라는 통일된 형식을 통해 메시지 해석이 가능한 것처럼, gRPC에서는 메시지 형식을 특정시킨 하나의 프로토콜 버퍼를 공유하게 됨으로써 다른 언어로 작성된 서버끼리 메시지 해석이 가능하게 되는 것이다.

gRPC도 HTTP 처럼 TLS 암호화도 가능하고, 클라이언트 사이드 로드밸런싱이 가능하며, 바이너리 파일로 메시지가 전송되기 때문에 문자열을 json 형식으로 전송하는 HTTP에 비해서 훨씬 가볍고 성능이 좋다.

## Reference


[Introduction to envoy’s Dynamic Resource Discovery (xDS) protocol.](https://medium.com/@rajithacharith/introduction-to-envoys-dynamic-resource-discovery-xds-protocol-d340032a63b4)

[Why gRPC?](https://grpc.io/)

[gRPC - Istio By Example](https://istiobyexample.dev/grpc/)

[Language Guide (proto 3)](https://protobuf.dev/programming-guides/proto3/)
