[ID: 6]
[Tags: KUBERNETES]
[Title: 쿠버네티스 ingress에 대하여]
[WriteTime: 2023-07-15]
[ImageNames: 1234517c-fa11-4296-87cb-3c5a99c7b677.png]
		
		## Content

1. Preamble
2. Terminology
3. 5 Ingress Usecases
4. References

## 1. Preamble


`Ingress`와 `IngressController`는 리버스프록시 및 로드밸런서 역할을 담당하는 쿠버네티스의 오브젝트이다. 커플 채팅 서비스를 쿠버네티스에 배포할 예정이어서, 인그레스 컨트롤러에 대해 공부하고 적용하기로 했다.
> 쿠버네티스 공식 도큐먼트를 참조했다.

## 2. Terminology


## Ingress란?


`Ingress`는 내부 서비스를 외부에서 접근이 가능하도록 하기 위해 클러스터 가장 앞단에서 HTTP, HTTPS 경로를 노출시키는 오브젝트이다.

인그레스를 통해 정상적으로 포트를 노출시키기 위해서는 `IngressController`가 필요하다. 인그레스는 K8s에 정의된 오브젝트이지만, 인그레스 컨트롤러는 다양한 벤더에서 제공하는 서드파티 인그레스 컨트롤러들 중 선택해서 구성할 수 있다. 대표적으로 `NginX`에서 제공하는 인그레스 컨트롤러가 있다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
    paths:
    - path: /testpath
    pathType: Prefix
    backend:
      service:
      name: test
      port:
        number: 80
```


### Annotation이란?


`Annotation`은 오브젝트에 메타데이터를 첨부할 수 있게해주는 애트리뷰트/필드이다. 키와 값으로 이루어진 `map`배열인데, 키와 값으로는 `문자열`만 사용할 수 있다. 위의 Yaml 파일에 나온 어노테이션을 예시로 들자면,

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
```


이 어노테이션을 적용하면, 모든 트래픽의 path가 `/`로 rewrite되어서 서비스로 라우팅된다.

FE / BE 간 리버스프록시를 사용할 때, 백엔드 경로에 /api를 붙여서 백엔드와 프론트엔드를 구분해서 라우팅을 하는 경우가 많은데, 이 rewrite 어노테이션을 이용하면 프론트엔드에서는 `HOST+/api/~` 경로로 요청을 보내지만, 실제 백엔드에서는 rewrite된 `\~` 경로로 요청을 받을 수 있게 된다.

`/api`는 백엔드를 구분하기 위한 목적밖에 없기 때문에, `/api`를 보고 어디로 라우팅을 해야할지 판단한 후에는 /로 rewrite을 해줄 수 있게되는 것이다.

### IngressClassName이란?


`ingressClassName`은 인그레스 클래스의 이름을 지정하는 부분이다.

한 클러스터 안에 여러개의 인그레스 컨트롤러가 실행될 수 있고, 이 인그레스 클래스 네임을 통해 각 인그레스 컨트롤러마다 다른 인그레스를 처리할 수 있다.

예를 들면, `인그레스 컨트롤러1`은 클라이언트에서 오는 패킷의 라우팅을 수행하고, `인그레스 컨트롤러2`는 마이크로서비스에서 서비스간의 통신을 수행할 수 있다.

사용자가 브라우저를 통해 도메인에 접속하면 컨트롤러1이 백엔드 서비스들로 라우팅을 하고, 접속해서 여러 마이크로서비스 간의 통신에는 컨트롤러2가 라우팅을 수행하게 되는 것이다.

인그레스 클래스 이름을 지정하기 전에 인그레스 클래스 오브젝트가 배포되어있어야한다. 설정파일 예시는 아래와 같다.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx-example
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: nginx-example
```


`metadata.name`은 이후에 인그레스 리소스 설정에서 `ingressClassName`의 값으로 사용된다.

`spec.controller`는 어떤 인그레스 컨트롤러를 쓸 것인지 명시해주는 부분이다.

`parameters`는 인그레스 컨트롤러에 전달될 설정들을 정의하는 부분이다.

### spec.rules란?


인그레스에서 `rules`는 어떤 기준으로 라우팅을 할 것이냐에 관한 부분인데, `호스트`와 `Path`를 기반으로 라우팅을 어디로 할지 결정한다.

FE와 BE를 예시로 서브도메인을 적용하면, `www.example.com` HOST에서는 프론트엔드 서비스로, `api.example.com` HOST에서는 백엔드 서비스로 라우팅을 할 수 있다.

설정파일은 아래와 같이 구성된다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 8080
```


rules의 애트리뷰트 중 `pathType`은 `Prefix`, `Exact`, `Mixed` 를 지정할 수 있다.


1. `Prefix`는 접두사를 기준으로 판단한다는 것이다.

path가 `/`, pathType이 `Prefix`로 되어있다면, 모든 경로는 기본적으로 가장 앞에 `/` 가 붙어있기 때문에 모든 경로를 일치하는 경로로 보고 라우팅을 한다. 그래서 `/`가 모든 트래픽을 라우팅할 수 있는 것이다.

`/api`로 path를 설정했을 때도 접두사를 보는 것이기 때문에 `/api/usr/id` 경로로 요청이 들어온다고 가정하면, 앞에 `/api`가 있기때문에 일치하는 경로라고 판단하고 라우팅하게 된다. 만약 여러 Path에 겹치는 경로가 있으면 더 많이 겹치는 Path로 라우팅한다.

`/api` 경로에는 `/`도 포함되어있지만 `/`로 라우팅 되지않고 `/api`로 라우팅되는 이유가 바로 거기에 있다.


2. `Exact`는 정확히 그 경로만 보는 것이다.

path가 `/`로 정의되어있어도, pathType이 `Exact`면 `/` 경로로 오는 요청을 제외하고는 모든 요청이 거부된다. 참고로 경로의 가장 마지막 `/`는 무시될 수 있다.

예를 들어 `www.google.com`을 host로 지정하고, path는 `/`, pathType은 `Exact`로 설정했다고 가정하자.

Exact에 따르면 `www.google.com/` 로 들어온 경로만 라우팅해야할 것 같지만, 경로상 마지막 `/`은 무시되기 때문에, `www.google.com/` 과, `www.google.com`은 모두 라우팅될 수 있다.

## 3. 5 Ingress Usecases


인그레스의 사용 유형이 꼭 다섯가지로 제한되는 것은 아니겠지만, K8S Official Document에서 소개된 유형들을 정리해보겠다.

### 1. 단일 서비스로 지원되는 인그레스


만약 인그레스 리소스에 rule을 정의하지 않는다면, 적어도 `defaultBackend`는 정의를 해야한다.

defaultBackend는 트래픽을 라우딩해야하는데 적합한 rule이 없을 때, 트래픽을 defaultBackend로 라우팅하게 된다. 근데 rule자체가 없다면, 이 인그레스는 요청받는 모든 트래픽을 defaultBackend service(단일 서비스)로 라우팅하게된다. 예시 코드는 아래와 같다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
  service:
    name: test
    port:
      number: 80
```


### 2. 간단한 팬아웃(fanout)


위에서 보았던 `Path` 기반의 라우팅을 의미한다.

### 3. 이름 기반의 가상 호스팅


위에서 보았던 `호스트` 기반의 라우팅을 의미한다.

### 4. TLS


인그레스 컨트롤러는 TLS 종료를 수행할 수 있다. 방법은 시크릿에 인증서와 개인키를 정의하고, ingress 설정파일에서 시크릿을 불러오면 된다. 이를 통해 HTTPS 통신을 구현할 수 있다.

TLS 통신의 종료지점이 되기 때문에, 인그레스 컨트롤러 이후의 트래픽은 일반 `plain-text`로 복호화된다. 시크릿 설정 예시 코드는 아래와 같다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt:
    tls.key:
      type: kubernetes.io/tls
```


위의 시크릿을 이용해 TLS 종료 지점인 ingress의 설정파일을 구성하는 코드 예시는 아래와 같다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
            tls:
            - hosts:
              - www.example.com
              - api.example.com
```


### 5. 로드 밸런싱


인그레스 컨트롤러는 로드밸런싱도 수행한다. 서비스로 트래픽이 들어왔을 때, 서비스에 연결된 파드가 여러 개라면, 인그레스 컨트롤러는 로드밸런싱 정책 설정에 따라 라우팅한다.

도큐먼트에서 기본적인 로드밸런싱 정책으로는 `가중치`를 예시르 들었다.

가중치는 처음 서버를 구성할 때 지정한 가중치가 쭉 이어지는 방식이다. 노드A는 리소스가 크고 B는 작다고 했을 때, A에 더 큰 가중치를 부여해서 A로 더 많은 트래픽이 전달될 수 있게한다. 인그레스 컨트롤러는 이런 기본적인 로드밸런싱 정책을 이용해서 라우팅을 한다.

도큐먼트에서 소개한 고급 로드밸런싱 중 `지속적인 세션`은 한 번 연결된 클라이언트와 서비스는 쭉 연결이 이어지게 하는 것이다.

만약 일반적으로 라운드로빈 로드밸런싱을 한다면 클라이언트는 매번 서비스에 접근할 때마다 다른 pod로 라우팅될텐데, 지속적인 세션은 한 번 연결된 파드에만 접근되게 함으로써 캐싱을 이용해 빠른 작업이 이루어질 수 있다.

또 `동적 가중치`는 리소스의 상태, 응답시간에 따라 가중치를 다르게 부여하는 것이다. 응답시간이 긴 파드가 있으면 가중치를 낮게 주는 방식으로 각 노드와 pod의 동적인 상태에 따라 효율적으로 트래픽을 라우팅할 수 있게 된다.

## 4. References


[K8s Official Docs-Ingress](https://kubernetes.io/ko/docs/concepts/services-networking/ingress/)
