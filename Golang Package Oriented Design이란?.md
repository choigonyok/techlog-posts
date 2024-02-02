[ID: 31]
[Tags: GOLANG]
[Title: Golang Package Oriented Design이란?]
[WriteTime: 2023/11/02]
[ImageNames: 0ab42ef2-c3f1-481b-1234-f8ea04d64750.png]

## Contents

1. Preamble
2. 레퍼런스 프로젝트
3. Golang은 객체지향 언어가 아니다
4. Package Design
5. Summary

## 1. Preamble

`Go`에 한 번 더 깊이 빠졌다. Go는 무엇을 중요하게 생각하는지, Go에서 좋은 코드는 어떤 코드인지, Go에서는 어떤 디자인 패턴이 유용하게 사용되는지 등 Go에 대해 더 자세히 공부하고 싶어졌다. Go에 대해 조금씩 더 알아갈 수록 하나의 관통되는 가치관이 있는 것 같은 느낌이 강하게 든다.

이 글은 Go로 소프트웨어 디자인을 어떻게 해야하는가에 대해 살펴본다.

## 2. 레퍼런스 프로젝트

Go로 개발된 여러 훌륭한 오픈소스 프로젝트들을 유심히 둘러보던 중, 의문점이 생겼다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/9A8593AE-771B-45B6-9C22-C2D1A597E4F9/294F026F-E985-419B-AE35-A283688A892C_2/KyF5gCXPnTCZDnjLqBxnFOyhiQukoycqSyO2lIiSEuIz/Image.png)

키-값 저장소인 `bolt`라는 프로젝트는 13700개의 Star를 받았다. 디렉토리 구조를 보면 수많은 소스파일의 나열이다. 게다가 이 소스파일들은 하나의 bolt 패키지로 싹 묶여있고, 유일한 디렉토리인 cmd/bolt 안에는 역시나 bolt 패키지인 `main.go` 하나만이 딸랑 들어있다.

`cmd`, `internal`, `pkg` 등의 디렉토리를 사용하는 `Go Standard Layout`이 Go 공식은 아니라고 해도, 하나의 소스파일로 모든 구현이 이루어져있다는 것이 잘 이해는 가지 않았다.

심지어 이 프로젝트는 성공적으로 마무리되고 ReadOnly로 아카이브되어있다. 

아래는 `Istio`의 깃헙 레포지토리이다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/9A8593AE-771B-45B6-9C22-C2D1A597E4F9/394510CE-D940-4C72-AA3E-5A2466386106_2/ld5aPfPUqU4IJF9b0suiDnFPo4yv15JDx1zWF16y44cz/Image.png)

Bolt가 모든 소스파일을 루트 디렉토리에 나열해둔 것과는 달리 그래도 좀 정리가 되어있다는 느낌이 든다.

Istio는 MSA로 설계되어있어서 `Operator`, `Istioctl`, `Pilot` 등의 도메인 디렉토리를 확인할 수 있고, 각 도메인 디렉토리 내부에는 `cmd`, `pkg` 디렉토리가 별도로 존재한다.

루트 디렉토리의 pkg 디렉토리 내부에는 또 수많은 디렉토리가 존재한다. 마치 도메인인듯, 객체인듯, 기능인듯한 이 디렉토리들이 너무 많아 도저히 프로젝트의 전체 구성을 파악하기가 어려웠다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/9A8593AE-771B-45B6-9C22-C2D1A597E4F9/A0A9F1DC-BB1B-43EB-B16C-33DC18ED4A9F_2/nXyHxIaKIanXYNyUvmfq0xIBVDON571dp7MAsi8vK7Uz/Image.png)

pkg 디렉토리 내부엔 레이어 컴포넌트들이 위치해있을 줄 알았다. Controller, Service, Repository, Handlers, Routers와 같은 추상적인 개념보다는 맡고있는 기능이나 도메인 중심으로 디렉토들이 구조화되어있는 것을 볼 수 있었다.

쿠버네티스 역시 Go로 작성된 오픈소스인데, 쿠버네티스 역시 pkg 디렉토리에 들어가보면 `securitycontext`, `serviceaccount`, `kubelet`, `kubectl`, `controlplane`, `apiserver` 등 쿠버네티스 리소스 별로 디렉토리와 패키지가 나누어져있는 것을 확인할 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/9A8593AE-771B-45B6-9C22-C2D1A597E4F9/8EF782B7-8FB6-4983-A71C-F45F2DEAC505_2/JxJZaiR1KcE0a9POjtUj0dEOikM7RqcFIjsZxnJedXoz/Image.png)

왜 Go는 레이어별로 구조를 나누지 않고, 기능이나 도메인 별로 디렉토리와 패키지를 나누는 것일까?

## 3. Golang은 객체지향 언어가 아니다

`Golang`은 객체 지향 언어가 아니다. golang의 struct와 interface로 객체지향을 흉내낼 순 있지만, 엄밀히 말하면 Go는 객체 지향 언어가 아니다.

많이 사용되는 MVC 등의 디자인 패턴이나 Hexagonal, Clean 아키텍처 역시 OOP에 기반하고있다.

OOP에서는 클래스가 먼저 정의된 이후, 의존성을 주입해 클래스 외부에서 객체를 생성하고, 클래스의 메서드를 이용할 수 있다. `Top-Down`(하향식) 방식이라고 할 수 있다.

반대로 Go는 `Bottom-Up`(상향식) 방식에 가깝다. 우선 로직을 먼저 작성하고, 이후에 코드의 복잡성이 증가할 때 인터페이스를 통해 코드를 분리해서 `Polymorphism`을 구현할 수 있고, 재사용성을 높이고 싶을 때 다른 패키지로 코드를 분리할 수 있다.

이런 언어적인 특성과 차이가 있기 때문에, Go에서의 Design 방식은 다른 OOP 언어들과 다르다.

정답이야 없지만, Go는 일반적으로 Package Focused Design(패키지 중심 설계)를 적용한다. 대부분의 유명한 Golang 오픈소스 프로젝트들 보면 모두 패지키 중심으로 설계되어있고, 약간의 디테일 차이만 존재한다.

Go로 Clean Architecture, MVC 패턴 등을 구현하는 방법이라며 디렉토리 구조를 소개하고 코드 예시를 보여주는 많은 글들이 있지만, 실제로 대규모의 엔터프라이즈급 오픈소스 프로젝트들에서 그런 방식으로 설계되어있는 프로젝트는 아직까지 보지 못했다.

Go를 쓴다면 Go답게 설계하고, Go답게 코딩하고, Go답게 테스트/유지/보수하는 것이 좋다고 생각한다. 

한 개발자는 자신이 구현한 클린 아키텍처 디렉토리 구조를 가지고있는 프로젝트 Go 커뮤니티에 게시했다가 여러 질타를 받았다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/9A8593AE-771B-45B6-9C22-C2D1A597E4F9/81A30B09-4719-41B7-8A2E-1405A30D50A4_2/t2oZV63ZAygHxX1XNFsNv0wbIbqqIzx4CCjEnyidkcYz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/9A8593AE-771B-45B6-9C22-C2D1A597E4F9/7DC6DC03-4BEE-4DA6-931F-8DB743C03E35_2/gyfxvJ1ynAjlyvka3trgHash5DbuT0bEm1yg9mhTCD8z/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/9A8593AE-771B-45B6-9C22-C2D1A597E4F9/0D555846-EAF0-4EE6-8678-0C10B25388C6_2/6t2vVUESCtDkSj4Jc5uMJpNNSaV2TlY0xwxoJ3HZE8Az/Image.png)

## 4. Package Design

### Package Naming

패키지의 이름은 그 패키지(모듈)에 무엇이 들어있는지를 명시하는 게 아니라, 이 패키지에서 무엇을 제공할 수 있는 패키지인지를 명시해야한다.

Go의 기본 내장 패키지들도 그렇게 네이밍되어있다. 

이름을 지을 때, common/base/utils 등의 이름으로 짓는 것은 피해야한다. 보통 이 패키지 안에는 프로젝트 전체에서 공통으로 사용되는 시간 데이터 포맷 변경 함수나, JSON 직렬화 함수 등 어디서든 불러서 사용될 수 있는 함수들이 들어가있고, import cycle이 생기는 것을 막기 위해 따로 묶어두곤 하는데, 묶는 것이 잘못됐다는 게 아니라, 이름이 패키지가 제공하는 것을 내포하고있지 않고 그냥 패지키 안에 들어있는 것들의 집합을 표현하고 있기 때문에 차라리 Time이나 Format등의 여러개 패키지로 나눠서 관리하는 것이 좋다.

### Why not?

벨 연구소의 브라이언 커니핸은 C 언어의 단점으로 컴파일러 관점에서 코드에 방화벽을 생성할 수 없다는 것을 문제로 꼽았다. OOP를 통해 개발자 레벨에서는 의존성을 관리하고 레이어를 생성할 수 있지만, 컴파일러 레벨로 넘어가면 그게 소용이 없어진다는 것이다.

Package Oriented Design은 하나의 패키지를 마치 작은 마이크로서비스 또는 API로 본다. 마치 모든 것을 막고 시작하는 Zero-trust 아키텍처처럼 처음 코드를 작성할 때부터 필요한 패키지만을 가져다 쓰게됨으로써 결합을 약화시키고 Scalable한 프로그램을 개발할 수 있게 된다.

중요한 것은, 하나의 패키지는 하나의 역할만을 가져야한다. SOLID 5 원칙 중 단일 책임 원칙처럼, 하나의 역할만을 가지고있어야 결합성이 낮아지고 확장성이 높아지며 심지어 하나의 컨벤션으로 작용해서 타인이 코드를 이해하기도 쉬워진다.

Util 패키지에 여러 기능들이 함께 속해있는 것보다 각 기능이 필요한 패키지에 각각 포함되는 것이 좋다. 물론 코드 재사용성이 조금 떨어질 수 있다. 그래서 Go로 개발된 많은 오픈소스 프로젝트들에서 Util 패키지를 그대로 사용하고 있는 것 같기도 하다. 어쨌든 원칙적으로, Go스러움 입장에서 보자면 적합하지는 않다.

Go의 기본 내장 패키지인 net/http 패키지에서도 net/http/server 패키지가 있는 게 아니라 그냥 net/http 패키지 안에 server.go 소스파일로 분리되어있다.

## 5. Summary

Go의 패키지는 다른 개발자가 가져다가 쓸 수 있다. Internal 디렉토리 안에 있는 건 Go에 의해 강제적으로 막히고 명시적으로 pkg에 있는 패키지들은 사용이 가능하다.

이건 마치 Go를 사용하는 전세계 모든 프로젝트가 하나의 프로젝트로 연결되어있는 것 같은 느낌을 준다.

Go의 내장 패키지부터, 내가 생성한 패키지, 다른 개발자가 작성한 오픈소스의 패키지 등 모든 패키지를 가져다가 쓸 수 있다.

패키지는 마이크로서비스와 결이 비슷하다. 내가 만든 프로젝트의 패키지를 누구든 언제든 가져다가 쓸 수 있다. 코드를 하드코딩하는 방식이 아니라 공식적으로 가져다가 쓰는 OpenAPI와 같은 느낌이다. 그래서 나의 패키지를 언제든 타인이 가져가서 쓸 수 있다는 생각으로 테스트를 신중하게 해야하고, 패키지의 성능을 정확히 측정해야한다.

비단 이런 도의적인 차원에서 뿐만 아니라 같이 서비스를 개발하는 팀 내에 다른 팀원이 내가 작성한 패키지를 언제든 가져다가 쓸 수 있는 것이다. 곧 패키지 = 나의 개발 실력과도 같다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/9A8593AE-771B-45B6-9C22-C2D1A597E4F9/F0091E5B-D86B-4B37-9339-0B02D21343A1_2/532jYqVnevnHVdMwNER4U0sIqorLg5sKmTG7lkXR8aIz/Image.png)

## References

[https://dave.cheney.net/practical-go/presentations/qcon-china.html#_package_design](https://dave.cheney.net/practical-go/presentations/qcon-china.html#_package_design)

[https://blog.gopheracademy.com/advent-2016/go-and-package-focused-design/](https://blog.gopheracademy.com/advent-2016/go-and-package-focused-design/)

[https://dev.to/ballweera/go-package-structure-design-b0m](https://dev.to/ballweera/go-package-structure-design-b0m)

[https://www.gobeyond.dev/standard-package-layout/](https://www.gobeyond.dev/standard-package-layout/)

[https://changhoi.kim/posts/go/go-pkg-architecture-theory/](https://changhoi.kim/posts/go/go-pkg-architecture-theory/)

[https://rakyll.org/style-packages/](https://rakyll.org/style-packages/)

[https://www.reddit.com/r/golang/comments/eamopu/go_software_design/](https://www.reddit.com/r/golang/comments/eamopu/go_software_design/)

[https://hackernoon.com/go-design-patterns-an-introduction-to-solid](https://hackernoon.com/go-design-patterns-an-introduction-to-solid)

[https://medium.com/swlh/provider-model-in-go-and-why-you-should-use-it-clean-architecture-1d84cfe1b097](https://medium.com/swlh/provider-model-in-go-and-why-you-should-use-it-clean-architecture-1d84cfe1b097)
