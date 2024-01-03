[ID: 49]
[Tags: istio]
[Title: [ #2 ] Istiod Architecture 이해하기]
[WriteTime: 2024-01-03]
[ImageNames: 65b1f072-d85a-478e-9eed-b0e1c98b2efa.png]

## Content


## 1. Preamble

> 이 글은 Istio Github 레포의 Istio/architecture/networking/pilot.md를 기반으로 작성되었다.


Istio의 아키텍처를 파악해보려고 한다. 아키텍처를 논리적으로 알고나서 코드를 보면 전체적으로 더 이해하기가 수월할 것이다.

기존 Istio의 아키텍처는 현재보다 훨씬 더 복잡했다. 이 아키텍처의 복잡성이 Istio의 러닝커브를 높이는데 한 몫 했다. 지금은 추상화를 통해 사용자 입장에서의 복잡성은 줄었지만, Istio에 기여하고싶은 나로써는 기존 아키텍처에 한 겹이 더 쌓인 형태이기 때문에 더 복잡하다고 말할 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/B42D6074-F301-41F1-B751-647A5DDE2B3C_2/xb9lQFSjF0TqY9yJ2BK4OuyBZmat9MhY9tZT2UiHpoUz/Image.png)


- **Pilot** - Responsible for configuring the proxies at runtime.
- **Citadel** - Responsible for certificate issuance and rotation.
- **Galley** - Responsible for validating, ingesting, aggregating, transforming and distributing config within Istio.

### 2. Control Plane


Istio는 크게 Control Plane과 Data Plane으로 나뉘어진다. 

컨트롤 플레인은 쿠버네티스의 컨트롤 플레인처럼 여러 역할을 담당하는 Istio 데몬 서버라고 할 수 있다.

위 사진에서 볼 수 있는 Pilot, Galley, Citadel, Mixer는 현재 Istiod라는 하나의 컴포넌트로 통합되었다. 즉 Istio의 컨트롤 플레인 = Istiod 라고도 말할 수 있다.

Istiod는 모듈러 모놀로식 구조를 가지고있다. MSA가 마이크로서비스간의 종속성을 완전히 삭제시킨 것과 다르게, 모놀로식 형태이지만 모듈 간 종속성을 최소화시키는 모놀로식 아키텍처와 MSA의 중간 단계라고 볼 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/74D2A44D-3627-4D6D-A2E4-708ED041DDC3_2/brFjPqAufVWeMDkgt367Dox6Qz8DTQkNX1I8otFZdh0z/Image.png)

Golang의 언어적 특성을 살려서 패키지를 활용해 모듈들을 구현한 모놀로식 구조로 구성이 되어있는 것 같다.

Istiod는 인증서 관리, 프록시 구성(XDS), 쿠버네티스 컨트롤러 등의 기능을 수행한다. 이 중 가장 중요한 역할은 서비스 메시 오픈소스인 만큼 프록시 구성이다.

Istiod는 오픈소스 프록시인 Envoy를 활용해 구현되어있다. 프록시 구성은 Envoy 뿐만 아니라 gRPC, ztunnel, ingress 등을 사용한다.

프록시 구성은 크게 3가지 파트로 나뉜다.


- 프록시 설정 동적 ingestion
- 프록시 설정 translation
- 프록시 설정 제공 (XDS)

## Config Ingestion


Config Ingestion은 설정 소화이다. 설정을 잘 소화시킨다는 맥락인 것 같다. Istio를 사용하는 개발자는 config 파일, 환경변수, 쿠버네티스 manifest 등 다양한 방식으로 Istio에 대한 설정들을 적용할 것이다. 이 설정들을 잘 수집해서 실질적으로 적용시키도록 하는 기능이다.

Istio는 프록시를 설정하기 위해 쿠버네티스, 파일, xDS 등 무려 20개가 넘는 리소스 타입들로부터 데이터를 읽어낸다고 한다.

이 Config Ingestion은 크게 네 가지의 컴포넌트로 분류되어있다.

### Config Store


여러 리소스 타입들로부터 Config 데이터를 읽어서 하나의 인터페이스로 모으는 컴포넌트이다. 앞서 말한 20개가 넘는 리소스 타입들에서 받아진 데이터들은 config.Config Struct로 래핑되어서 하나의 통일된 인터페이스를 제공한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/E46695DA-19CB-43EF-8EF3-7DBB042BDCCA_2/CNsKIfNKqYzzCRryRStmTgkC4xisvvIgnTOoGCw6Og8z/Image.png)

Golang은 다형성을 위해 DuckTyping 방식의 인터페이스를 지원한다. 이렇게 하나의 인터페이스를 통해 메서드를 통일시켜서 가독성을 높이고 코드 재사용성을 높일 수 있게 될 것이다.

config.Config의 구조를 살펴보기 위해 우선 Istio의 기본 module 명을 확인해보았다.

Golang에서는 모듈 의존성을 관리하기 위해 go.mod와 go.sum 파일을 사용한다.

이 두 파일은 **go mod init 모듈명,** **go mod tidy** 를 통해 생성되고 업데이트되는데, 이 때 입력한 모듈명이 이후 이 프로젝트의 패키지를 호출할 때 접두사로 활용된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/D6C168C0-9C96-4058-9912-1073F62C6427_2/jDDs7uwKdlVfTPVUbvEaSO86rQpTItNKq3M6xWjjrFsz/Image.png)

aggregate 패키지에서 config.Config가 사용되는 것을 확인하였고, import된 패키지 목록을 확인해보니 아래와 같이 istio.io/istio/pkg/config 경로로 패키지가 지정되어있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/DC6B1EFE-3347-4B3F-AEEB-41890106BAA6_2/9aBKxcyuU3o3K9o2ViY1Xn7Q1iIGpsfSUG1HyHivyMgz/Image.png)

그래서 접두사인 istio.io/istio를 제외하고 루트 디렉토리에서 pkg/config 경로로 접근해보니 model.go 소스파일에서 config.Config의 출처를 찾을 수 있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/C26545AB-887D-4FB8-BF3A-989D2650AD8B_2/McU4pDkHPZIxFmU7COJA66XcWzqubxdjPkxI4ixNI6cz/Image.png)

Config Struct 안에 Meta라는 Struct가 포함되어있어서 확인해보니 Meta는 아래와 같은 필드들을 가지고있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/E120460E-576A-4CA2-B7DA-44792CF3CCC3_2/cuIyjyvGDiGpMkrDiNAulM8mAVxxUWNkhtd3UDtWgEMz/Image.png)

###  ServiceDiscovery


서비스 디스커버리는 쿠버네티스에서 서비스 이름만으로 클러스터 내부에 존재하는 해당 서비스의 ClusterIP를 응답해주는 DNS이다.

서비스 디스커버리도 ConfigStore처럼 여러 리소스타입으로부터 오는 설정 데이터들을 모으는(aggregate) 역할을 한다. 다른 점은 리소스에 대한 접근을 제공하지 않고, 대신 서비스 기반의 내부 리소스 변화를 먼저 컴퓨팅한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/9F4FB915-21A5-4118-9287-E5F7F66D15F0_2/SytrXNQGNwzsPIdMgAxjFLstAYyO7pEJcGysFtg1MgYz/Image.png)

예시로는 /pilot/pkg/model/service.go 경로 Service/ServiceInstance Struct가 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/D844AD9C-BAFC-4A15-9A7F-4E4B757AC0CF_2/1yNWg2D5VAMZfDVPVIn75c4iGrhq9xKgLI1lUe0Ju0cz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/3D9B666E-D47B-4B26-B41D-C2EFA2D5CE4C_2/FomgxEyPUqHf8IkvZYR9Z99jgsQvx2NY8vnGnih9Q8Yz/Image.png)

###  PushContext


PushContext는 Config가 적용(Push)될 때마다 적용된 current state에 대한 스냅샷을 저장하는 컴포넌트이다. 이 PushContext 덕분에 Config에 대한 조회를 할 때 대부분 lock-free가 가능하다고 한다.

### Endpoints


Endpoints는 최적화된 코드 경로를 가지고있다. 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/641A455A-F81D-4CF5-8257-AC002C0D09C8_2/D2idTjW3CziyHKknNzhhKBMZ3dx9ZGfTNRDYxigVS7gz/Image.png)

##  Config Translation


Config Translation은 Config Ingestion에서 인풋으로 받은 설정 데이터들을 연결된 XDS 클라이언트인 Envoy가 사용할 수 있는 타입으로 변환시킨다.

Generator들에 의해서 이 작업이 이루어진다. 예를 들어 Routes를 빌드하기 위해서 RouteGenerator가 이 역할을 하는 식이다.

Envoy XDS 타입 이외에도, DNS에 사용되는 NameTable같은 Istio 커스텀 타입들도 포함된다.

Config Translation 단계에서, 프록시 별로 적용된 label에 따라서 다른 ingress 정책을 적용하는 등의 기능이 있기 때문에, 정적으로 번역하는 것이 불가능하다.

### Caching


Config Translation 과정에서, 특히 protobuf 인코딩에서 Istiod가 가장 많은 리소스를 사용한다. 그래서 캐싱이 필요하고, 이미 인코딩된 protobuf.Any를 저장한다.

이 캐싱은 generators로 가는 모든 설정 인풋들을 캐시 키로 선언하는 것에 의존하게된다. 즉, 만약 키가 아닌 인풋이 generator로 가게 되면 잘못된 설정이 적용될 것이다.

이를 막기 위해서 

generator로 가기 전 자체적인 로직을 통과하도록 구현해서 키인 인풋들만 generator로 보내도록 하는 방식

조심 또 조심하는 방식

키가 다른 값을 가지고 있으면 CI 단계에서 에러를 잡도록 하는 방식

이 있다.

### Partial Computations


Partial Computations도 캐싱처럼 퍼포먼스를 최적화하는데 중요한 요소이다. Partial Computations은 변경사항이 있을 때마다 매번 모든 리소스를 모든 프록시에 보낼 필요가 없도록 해준다. 

##  Config Serving


Config Serving은 사이드카 프록시와 gRPC 커넥션을 맺고 인풋으로 받은 Config를 적용시키는 레이어이다.

###  Request


리퀘스트는 리소스를 요청하는 프록시 클라이언트로부터 리퀘스트가 온다. 

클라이언트는 request, 이전 푸시의 ACK, 이전 푸시의 NACK 메시지를 보낼 수 있다. 이 메시지들은shouldRespond 로직을 통해 분리된다.

###  Pushes


푸시는 Istiod가 설정을 업데이트 해야한다는 걸 감지할 때 발생한다. Request와 같은 결과를 일으키긴하는데, 다른 리소스에 의해 트리거된다.

Config Ingestion 단계에서 다양한 리소스 타입의 인풋들이 생기면 과한 오버헤드를 줄이기 위해 Push 큐에 넣어두고 한 번에 일괄적으로 푸시해서 적용시킨다.

프록시 클라이언트마다 개별적으로 푸시 큐를 가지고있는게 아니고 하나의 푸시 큐를 사용하기 때문에, 어떤 푸시에는 특정 프록시 클라이언트에 설정 업데이트가 안될 수도 있고, 어떤 푸시에는 특정 프록시 클라이언트에 수많은 설정 업데이트가 적용될 수도 있다.

Pusher는 개별적으로 Push Job을 실행해서 큐잉되어있는 설정들을 각 프록시 클라이언트에 푸시한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/D0DCD174-0F5B-4272-8C6F-CC0397815DFA_2/kw2sTxzjJyhIyiGTLxmFQJ4PHByhNlPv9XlB2sSeYNAz/Image.png)

### Optimizations


Istio에는 Full 푸시라는 개념이 있다. Full 푸시에만 PushContext를 다시 계산한다. Full 푸시가 아니면 지난 번 PushContext를 재사용한다.

Full 푸시이더라도, 최대한 지난 PushContext에서 많은 것들을 카피하도록 구현되어있다. 

해당 설정이 업데이트 되었을 때 각 프록시들이 영향을 받는지를 체크한다. 예를 들어 Gateway 설정을 업데이트하는 것과 프록시는 관련이 없다.

만약 프록시가 영향을 받는 설정 업데이트면, 어떤 타입이 영향을 받는지를 확인한다. 필요한 타입만 재생성할 수 있도록 하기 위해서이다.

마지막으로 영향을 받는 타입 중 어떤 서브셋이 생성되어야하는지를 확인한다. XDS는 SotW(State of the World) 모드와 Delta 모드가 있다. SotW 모드에서는 업데이트가 하나만 있어도 타입의 모든 리소스들을 생성해야하는데 반해서, Delta 모드에서는 변경된 리소스만 업데이트가 가능하다.

Istio는 SotW / Delta 프로토콜을 모두 지원하긴 하는데, 델타모드 구현이 아직 최적화가 잘 되어있지 않아서 SotW와 퍼포먼스가 비슷하다.

##  Controllers


Istiod에는 컨트롤러들도 구현되어있다. Istiod에서 컨트롤러는 여러 쿠버네티스 클러스터들과 외부 리소스들의 current state를 watch하고 있다가 필요할 때 요청을 보내는 역할을 한다.

Istio에서는 여러 라이브러리들을 통해서 컨트롤러를 구현했는데, 아직 완벽한 컨트롤러를 구현하진 못했다고 한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/728A5605-8F84-481F-9620-E71E5DFE9405_2/PfkQYy9FZ5ChSWeR0f45pSCx5io1gp7XqCeK70Kt5xAz/Image.png)

### Mesh Config


Mesh Config는 쿠버네티스 컨피그맵에서 설정 데이터를 읽어서 프로세싱하고 특정 타입의 MeshConfig에 머지하는 역할을 하는 컨트롤러이다.

그리고 MeshConfig의 current state를 체크하는 mesh.Watcher가 데이터 변경을 감지한다.

###  Ingress


Ingress 리소스는 빠르게 Istio의 커스텀 리소스 타입인 VirtualService와 Gateway로 변경되고, 컨트롤러가 Ingress 리소스를 읽어도 

### Gateway


게이트웨이는 Istio에서 Ingress 리소스같은 역할을 한다. 게이트웨이 컨트롤러는 게이트웨이 API 타입을 ConfigStore 인터페이스를 만족시키는 VirtualService와 Gateway로 변환시킨다.

Gateway (referring to the [Kubernetes API](http://gateway-api.org/), not the same-named Istio type) works very similarly to [Ingress](https://github.com/istio/istio/blob/master/architecture/networking/pilot.md#ingress). The Gateway controller also coverts Gateway API types into `VirtualService` and `Gateway`, implementing the `ConfigStore` interface.

However, there is also a bit of additional logic. Gateway types have extensive status reporting. Unlike Ingress, this is status reporting is done inline in the main controller, allowing status generation to be done directly in the logic processing the resources.

Additionally, Gateway involves two components writing to the cluster:


- The Gateway Class controller is a simple controller that just writes a default `GatewayClass` object describing our implementation.
- The Gateway Deployment controller enables users to create a Gateway which actually provisions the underlying resources for the implementation (Deployment and Service). This is more like a traditional "operator". Part of this logic is determining which Istiod revision should handle the resource based on `istio.io/rev` labeling (mirroring sidecar injection); as a result, this takes a dependency on the "Tag Watcher" controller.

### CRD Watcher


Istio에서는 기본 CRD 리소스 타입과 다르게 설정에 빠진 타입들이 있더라도 나이브하게, graceful하게 처리할 수 있도록 CRD Watcher 컴포넌트를 도입했다.

This is consumed in two ways:


- Some components just block on `watcher.WaitForCRD(...)` before doing the work they need.
- `kclient.NewDelayedInformer` can also fully abstract this away, by providing a client that handles this behind the scenes.

###  Credentials Controller


쿠버네티스 Secret 오브젝트에 저장되어있는 TLS 인증서 정보에 접근시켜주는 컨트롤러이다. 단순히 접근만 시켜주는 게 아니라, 요청자가 해당 namespace 안에서 secret에 접근이 가능한 권한이 있는지도 체크하는 인증 로직이 포함되어있다.

### Discovery Filter


discoverySelectors는 아래 예시 코드철머 네임스페이스마다 서비스 디스커버리를 활성화/비활성화 시킬 수 있게하는 MeshConfig의 설정이다. 

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
  meshConfig:
    defaultConfig:
      discoverySelectors:
        namespaceA: true
        namespaceB: false
```


Discovery Filter 컨트롤러는 MeshConfig의 discoverySelectors를 구현하기 위한 컨트롤러이다. 서비스 디스커버리가 수행될 때 어떤 네임스페이스에서만 수행되어야하는지를 결정한다.

###  Multicluster


멀티클러스터 환경에서 Secret 오브젝트로 저장되어있는 kubeconfig 파일을 읽거나, 각 클러스터 별로 쿠버네티스 클라이언트를 생성하는 역할을 하는 컨트롤러이다. 클러스터를 추가/업데이트/삭제할 수 있는 핸들러 등록이 가능하다.

Multicluster Secret 컨트롤러를 기반으로, 앞서 언급한 크리덴셜 컨트롤러와 쿠버네티스 서비스디스커버리 컨트롤러가 구현되어있는데, 크리덴셜 컨트롤러는 말한 것처럼 TLS 인증서 접근과 접근 권한 확인을 담당하고, 쿠버네티스 서비스디스커버리 컨트롤러는 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D3FA7853-6B09-431F-92DA-5CB37C1896C0/4D65F82F-577C-4DC8-9D03-41B205B98590_2/1BxHVEyNBzbQHQXVWa4xWQRl8ywb3BQIB6IY2vxcFh0z/Image.png)

### VMs


### Reference


[modular monolothic](https://giljae.com/2022/10/13/Moduler-Monolithic-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98.html)

[Official Readme: Architecture of Istiod](https://github.com/istio/istio/blob/master/architecture/networking/pilot.md)

[Discovery Selectors](https://istio.io/v1.14/blog/2021/discovery-selectors/)
