[ID: 30]
[Tags: OPS]
[Title: Jaeger 아키텍처 분석]
[WriteTime: 2023/10/30]
[ImageNames: ]

## Contents

1. Preamble
2. Jaeger Architecture
3. Sampling
4. Reference

1. ## Preamble


오픈 소스 프로젝트를 기획 및 설계하고 있다. 관련한 내용은 따로 포스트를 추가적으로 작성할 예정이다. 오픈소스 프로젝트를 성공적으로  설게 / 구현 / 배포하기 위해, 내가 개발하려는 소프트웨어와 공통점이 많고, 또 배울 점이 많은 오픈소스들을 찾아서, 서비스 아키텍처와 코드 구조를 분석하기로 했다.

Kiali, RBAC-tool, FluxDB, Vertical Pod Autoscaler, Istio, etcd 등 다양한 오픈소스 프로젝트들을 찾아보다가 Jaeger을 소스 코드를 보게되었고, 아래와 같은 이유들로 Jaeger에 대해 깊이 알아보기로 했다.


1. 상세한 아키텍처 구조를 설명해주는 공식문서
2. Go standard directory structure 
3. 많이 사용되는 트레이싱 도구
4. CNCF Graduated
5. 개발할 소프트웨어와 비슷한 Visualizer 기능

2. ## Jaeger Architecture


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/62CA6542-E0E2-427F-9137-EE5A1DE9FC26/95C3F2AB-B363-40C0-B093-AA8A99C36852_2/UDaJWV6byg34lihPR9yQji4Rq5dVIVH4HHX4r8WkM3Ez/Image.png)

Jaeger는 바이너리 파일 하나 안에서 전체가 동작하는 all-in-one 방식과 각 컴포넌트가 독립적으로 확장될 수 있는 MSA 방식으로 각각 설치가 가능하다.

Jaeger github 레포지토리에서 all-in-one 방식과 MSA 방식은 하나의 레포지토리에 통합되어있다. all-in-one 방식은 각 마이크로 서비스들을 하나의 all-in-one 소스 파일에 import해서 통합하는 방식으로 구성되어있다.

### Client


jaeger-client는 OpenTracing-API를 사용한다. OpenTracing-API는 분산 시스템에서 트레이싱을 수행하기 위한 표준 API이다.

OpenTracing-API에 의해 측정되고있는 어플리케이션에 요청이 들어오면 Client 컴포넌트는 Span을 생성하고, 요청이 다른 어플리케이션으로 나갈 때 요청에  spanID, traceID, baggage 등의 context information을 추가해서 요청을 보낸다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/62CA6542-E0E2-427F-9137-EE5A1DE9FC26/7472FAAC-F734-4AC5-943F-8C06BB21E736_2/Rd58DnKg1UlDrlvST0zeic1uDgjlUxy8mbNwVGyQgEkz/Image.png)

요청이 다른 어플리케이션(서비스)로 플로우를 따라 요청될 때마다 받은 context information에 추가로 context information을 덧붙여서 요청을 전달 하게된다.

예를 들어, 서비스 A가 B에게 C에 대한 데이터를 요청했다고 가정하자, 그럼 B는 C에게 요청을 보내고, C가 B에게 응답, B가 다시 A에게 응답하는 식으로 요청 흐름이 앞으로 쭉 갔다가 다시 뒤로 쭉 돌아오게 된다.

이 과정에서 추가적인 요청을 보낼 때 context information이 덧붙여지게 된다.

오퍼레이션 이름, 시간, 태그, 로그 등의 profiling data는 계속 이어서 전달되지 않고, 전달은 가벼운 id들과 baggage 등의 context information만 추가된다. Profiling Data는 요청에 추가되지 않고, 따로 비동기적으로 jaeger 백엔드에 전달된다.

트레이스가 샘플링이 될 때마다 해당하는 profiling data는 jaeger 백엔드로 전송되고, 샘플링되지 않은 트레이스면 profiling data를 전송하지 않는다. jaeger는 기본적으로 운영환경을 default로 생각하고 설계되어있기 때문에, 만약 개발단계 등에서 필요에 따라 profiling data 전송에서 발생하는 오버헤드를 줄이고 싶다면 샘플링 전략을 변경해서 샘플링되는 비율을 더 낮게 설정할 수 있다.

## Sampling


샘플링은 전체 트레이스 중 백엔드로 전송될 **일부** 트레이를을 선정하는 행위를 의미한다.

모든 트레이스마다 하나도 빼놓지 않고 모두 백엔드로 profiling data를 전송한다면, 대규모 서비스의 경우에 트레이싱 백엔드로 데이터를 전송하는 오버헤드가 과하게 커지게 되고, 서비스에 성능저하가 발생할 수 있다. 그래서 모든 트레이스들 중 일부만을 골라서 백엔드로 전송하는 방식이 일반적으로 사용된다. Jaeger는 0.1%만 샘플링한다. 서비스로 요청이 1000개 들어와서 1000개의 트레이스가 생성된다면 그 중에 하나만 백엔드로 전송한다는 의미이다.

샘플링 비율은 높을 수록 더 fine-grained한 트레이싱이 가능하자만 앞에서 언급한 부작용들이 있기 때문에, 적절한 비율을 설정하는 것이 중요하다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/62CA6542-E0E2-427F-9137-EE5A1DE9FC26/ED4A5EF3-41FA-4D4E-88BE-C3AD86DD9D30_2/3aiXTbddiKcDNyxCGmB3hOUxnQcok7n4vunahjn3L2Mz/Image.png)

MSA를 대상으로 Jaeger를 트레이싱 도구로 사용한다고 가정했을 때, 데이터 흐름을 추적하려면 결국 모든 마이크로 서비스들로 들어가고, 모든 마이크로 서비스에서 나가는 모든 요청을 Jaeger가 파악하고 있어야하고, 그 요청들을 Intercept해서 요청에 Trace ID 등의 데이터를 추가하며, Jaeger 백엔드로 Profiling Data를 전달할 수 있어야한다. 

이 로직은 두 방식으로 구현될 수 있다.


1.  서비스의 백엔드 코드에 트레이스 기능을 직접 또는 라이브러리등을 통해 구현하는 방식

이 방식은 이미 개발되어 운영중인 서비스에 적용하기엔 까다롭다. 기존 코드의 수정이 필요하다. 


2.  사이드카 프록시를 사용하는 방식

이 방식은 추가적으로 사이드카 프록시를 구현해야하는 복잡성이 있다

그래서 Istio는 Istio 엔보이 프록시와 Jaeger를 미리 통합해두어서 개발자가 쉽게 사용할 수 있도록 제공하고있다.

### Jaeger Agent


jaeger agent는 네트워크 데몬이다. jaeger client가 UDP로 전달하는 span(트레이스 데이터)를 listen하고, span을 묶어서 jaeger collector에게 전송하는 역할을 한다. Jaeger Agent는 Jaeger Client와 같이 서비스(호스트 또는 컨테이너)마다 개별적으로 배포되는 컴포넌트이고, 굳이 Client와 Agent를 분리한 것은 마치 리버스 프록시처럼 Client에게 Collector의 위치를 숨기고, Client가 라우팅과 관련된 복잡한 일들을 처리하지 않도록 기능대로 서비스를 분리한 것이다. 이를 통해 유지보수와 확장도 용이해지게 된다.

### Collector


collector는 각 Jaeger Agent들로부터 받은 모든 트레이스들을 수집해서, 유효성검사->인덱싱->변환->저장의 프로세싱 파이프라인을 실행시킨다.

Jaeger는 트레이스 데이터를 저장하기 위한 스토리지로  플러그인 패턴으로 카산드라, Elastic Search, 카프카를 지원한다고 한다. 플러그인 패턴은 다른 것으로 변경된다고 해도 전혀 문제가 되지 않는, 기존 코드를 수정할 필요가 없는, 독립적인 형태로 구성하는 패턴이고, 즉  카산드라를 사용하다가 카프카로 스토리지를 변경해도 아무런 문제가 되지 않는다는 것을 의미한다.

### Query


Query는 사용자(개발자)가 dashboard UI에 검색한 트레이스 데이터를 스토리지에서 조회해서 전달하는 역할을 한다. 말그대로 Query라는 이름이 딱 알맞다.

## Summary


Jaeger의 아키텍처에 대해 이해했으니 코드 리뷰를 진행하면서 Jaeger는 어떤 아키텍처로 구현을 했는지, 어떤 디자인 패턴들이 사용되었는지, 어떤 모델을 가지고있는지 등에 대해 알아볼 예정이다.

또 추가적으로 빌드 프로세스 자동화를 위한 Makefile, hack 스크립트, e2e를 포함한 테스트 코드에 대해서도 깊이 알아볼 예정이다.

##  Reference


[Jaeger Architecture](https://www.jaegertracing.io/docs/1.23/architecture)