[ID: 54]
[Tags: PROJECTS CLOUD INFRA SEC]
[Title: 테라폼 & ACM & Route53로 TLS 구현 자동화]
[WriteTime: 2024-01-09]
[ImageNames: 1d9dff4c-c52d-4305-9558-7a8b753f54dd.png]

## Content

1. Preamble
2. HTTPS/TLS/SSL이란
3. TLS 동작방식
4. Route53 테라폼 구현
5. ACM 테라폼 구현
6. References

## 1. Preamble


테크 블로그에 HTTPS를 구현했다. HTTPS는 클라이언트와 서버 간의 통신을 암호화하기 때문에 HTTP보다 상대적으로 보안에 강력하다. 뿐만 아니라, 대부분의 브라우저에서는 HTTP를 통해 서버에 접근할 때 아래 사진처럼 사용자에게 무언가 보안적으로 위험하다는 피드백을 주기 때문에 사용자 경험 측면에서도 HTTPS를 사용하는 것이 좋다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D1F76A8D-D660-4441-BE9C-7CDC0B079456/269766F9-10B6-43D5-ACE8-B22FB5C77D3E_2/R1BPQYcMMgXVnYWtra4sFvI5vY8yekHla6Aar7oGs2Uz/Image.png)

스크립트와 테라폼으로 테크 블로그 어플리케이션을 프로비저닝할 때, HTTPS 역시 자동으로 적용되도록 하기 원했고, 이 기능을 테라폼과 AWS의 ACM을 통해 구현하기로 했다.

## 2. HTTPS/TLS/SSL이란


HTTP는 HypterText Transfer Protocol의 준말로, 다양한 엔드 디바이스들이 인터넷이라는 하나의 환경에서 정보를 주고받기 위한 일종의 규칙, 프로토콜이다.

HTTPS는 이 HTTP의 가장 마지막에 Security을 추가해서 HTTP에 보안을 추가했다는 의미를 가지고있다.

SSL과 TLS는 HTTPS와 함께 꼭 붙어 따라다니는 용어이다. SSL은 Secure Sockets Layer, TLS는 Transfer Layer Security의 준말로, HTTPS의 보안을 실질적으로 구현하는 또 다른 프로토콜이라고 할 수 있다.

두 용어를 혼용해서 사용하기도 하지만, SSL은 과거 방식이고, 현재는 TLS만 사용된다.

## 3. TLS 동작방식


데이터를 암호화하기 위해서 키가 사용된다. 키를 사용해 암호화/복호화하는 방식은 **대칭 키**를 사용하는 방식과 **비대칭 키**를 사용하는 방식으로 구분된다.

대칭 키는 데이터를 암호화하는 키와 복호화하는 키가 같은 것을 의미한다. 이럴 경우 서버에서 클라이언트에게 키를 전달해줘야 상대방이 복호화를 할 수 있는데, 중간에 키가 탈취당하면 누구나 데이터를 복호화해서 내용을 확인할 수 있게된다.

그래서 TLS는 **비대칭 키**를 사용한다. 암호화하는 키와 복호화하는 키가 다른 것이다. 암호화 키는 **공개** **키**(Public Key)라고 부르고, 복호화 키는 **사설 키**(Private Key)라고 부른다. 이 때 역시 서버에서 클라이언트에 키를 전달해주어야하는데, 이 때 공개 키를 전달해준다.

그럼 중간에 누군가 키를 탈취하더라도, 앞으로 클라이언트가 서버에게 보낼 데이터를 확인이 불가능하다. 해당 키는 복호화 키인 사설 키가 아니라 암호화 키인 공개 키이기 때문이다.

사실, TLS에서 실질적으로 서버가 클라이언트에게 암호화 키를 전달하진 않는다. 이 과정은 중간에 CA(Certificate Authority)라고 하는 외부 기관/서비스의 도움을 받는다.

왜 외부 기관을 굳이 거치게 되는 걸까?

클라이언트(브라우저)는 접속하려고 하는 웹 페이지가 신뢰할 수 있는 페이지인지 아닌지를 알 수 없다. 앞서 말한 것처럼 비대칭 키를 사용해서 통신을 암호화하게되면 제 3자가 중간에 통신을 훔쳐보지 못하게 할 수는 있지만, 그렇다고해서 클라이언트가 통신할 대상인 서버가 신뢰할만한 대상인지까지는 알 수 없다.

남들이 엿듣지 못하는 방에서 단 둘이 이야기(비대칭 키 사용)를 하는 것이, 내 앞에 있는 사람이 나에게 해를 가하지 않을 착한 사람이라는 것을 보장해주진 않는다. 전혀 별개의 이야기이다.

따라서 남들이 엿듣지 못하는 방에 들어가기 전에, 나와 같이 그 방으로 들어갈 상대방(서버)가 착한 사람인가를 먼저 확인해야한다. 이 과정을 **CA**를 통해 할 수 있다.

Let\'s Encrypt 등의 여러 CA들이 존재하고, 모든 CA들은 보안적으로 안전한지 국제적으로 꾸준히 확인받게된다.

CA를 통해 서버의 신뢰성을 확인하는 절차를 간단히 설명하면 아래와 같다.


1. 서버가 자신의 신뢰성을 검증받기 위해 특정 CA에 검증 요청을 한다.
2. CA는 해당 서버를 검증하고, 서버에게 인증서를 발급한다.
3. 서버는 자체적으로 공개 키와 사설 키를 생성한다.
4. 서버는 클라이언트가 HTTPS로 접근하면, 해당 인증서와 자신이 생성한 공개 키를 클라이언트에 제공한다.
5. 클라이언트는 인증서를 확인하고 서버를 신뢰한다.
6. 클라이언트는 서버가 제공한 공개 키를 사용해 통신을 암호화해서 전송한다.
7. 서버는 암호화된 요청을 받으면, 가지고있는 사설 키로 요청을 복호화해서 내용을 확인한다.

이 때, CA가 발급하는 인증서를 TLS Certificate라고 부르고, 서버가 암호화된 요청을 사설 키를 통해 복호화하는 과정을 TLS Termination(종료)라고 부른다.

## 4. Route53 테라폼 구현


Route53은 AWS에서 제공하는 DNS(Domain Name System)이다. Route53을 통해 간편하게 도메인을 생성할 수도 있고, 해당 도메인에 대한 여러 레코드들을 프로비저닝 및 관리할 수 있다.

나의 경우에 우선 manually 도메인을 등록해두었다. (choigonyok.com)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D1F76A8D-D660-4441-BE9C-7CDC0B079456/43B0C4A6-1F91-4B50-B3F3-2A4549BB4FD1_2/4GAnft2dPyESxyqLIjHxhVyh2qiWhTDyx1FAlBWooTsz/Image.png)

기존에는 도메인을 등록하는 과정까지도 테라폼을 통해 구현해서 자동화시키려고 했다. 그러나 매번 서버를 종료시킬 때마다 함께 삭제되었던 도메인을 다시 등록하게 될 때, 인증 과정에서 최소 몇 십분에서 최대 하루 이상의 시간이 소요되는 것을 알아차리게 되었다.

그래서 등록은 계속 해두는 상태에서, 해당 도메인에 대한 호스팅 영역만 삭제/생성하는 방식으로 방향을 틀게 되었다.

도메인과 관련해서 **네임서버 레코드**, **호스팅 영역**에 대해 알아야한다.

### 네임서버 레코드


네임서버 레코드는 도메인을 등록하면 자동으로 생성되는 4개의 도메인을 의미한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D1F76A8D-D660-4441-BE9C-7CDC0B079456/A7AC9EBC-6C52-416A-9248-80B8CA291456_2/9IN58XxR4eirEjfnfGWnp69V1w7KhHVOUqZsF9s0lqMz/Image.png)

웹 브라우저에서 문자열 주소인 도메인을 입력하면, 해당 도메인에 등록되어있는 이 네임서버 레코드 주소에 위치하는 DNS에 도메인을 질의하게되고, DNS는 알맞은 IP주소를 리턴해주게된다. 이후에 클라이언트는 DNS로부터 리턴받은 IP주소로 접근할 수 있다.

### 호스팅 영역


호스팅 영역에는 DNS에 해당 도메인이 질의되었을 때, 어떤 IP주소를 리턴할 것인지가 정의된다.

```go
resource "aws_route53_zone" "main" {
  name = "choigonyok.com"
}

resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.choigonyok.com"
  type    = "A"
  alias {
    name = aws_lb.blog.dns_name
    zone_id = aws_lb.blog.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "ci" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "ci.choigonyok.com"
  type    = "A"
  alias {
    name = aws_lb.blog.dns_name
    zone_id = aws_lb.blog.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "cd" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "cd.choigonyok.com"
  type    = "A"
  alias {
    name = aws_lb.blog.dns_name
    zone_id = aws_lb.blog.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53domains_registered_domain" "choigonyok_com" {
  domain_name = "choigonyok.com"

  name_server {
    name = "${aws_route53_zone.main.name_servers.0}"
  }

  name_server {
    name = "${aws_route53_zone.main.name_servers.1}"
  }

  name_server {
    name = "${aws_route53_zone.main.name_servers.2}"
  }

  name_server {
    name = "${aws_route53_zone.main.name_servers.3}"
  }
}
```


Route53/도메인 관련한 테라폼 코드를 살펴보면, aws_route53_zone 리소스를 통해 호스트 영역을 생성한다.

그리고 이 호스트 영역에서 메인 도메인인 **choigonyok.com** 이외에도 **www**, **ci**, **cd** 세 가지의 서브도메인을 생성했다. ci, cd 서브도메인은 각각 jenkins와 argocd UI 대시보드에 접근하기 위해 생성했다.

alias 필드를 통해서 이 서브도메인들을 통해 접근된 요청들은 EKS 클러스터의 **nginx ingress controller**로 포워딩 시켜주는 AWS NLB에 접근하게된다. 그럼 **nginx ingress controller**에서 **ingress** 리소스를 통해 host 별로 알맞는 서비스에 요청을 라우팅한다.

```go
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  rules:
  - host: www.choigonyok.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 80
      - path: /haproxy/mysql
        pathType: Prefix
        backend:
          service:
            name: haproxy-mysql-external-name
            port:
              number: 80
      - path: /haproxy/jenkins
        pathType: Prefix
        backend:
          service:
            name: haproxy-jenkins-external-name
            port:
              number: 80
  - host: ci.choigonyok.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins-external-name
            port:
              number: 8080
  - host: cd.choigonyok.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-external-name
            port:
              number: 80
---
apiVersion: v1
kind: Service
metadata:
  annotations:
  name: jenkins-external-name
spec:
  type: ExternalName
  externalName: jenkins-ha-haproxy.devops-system.svc.cluster.local
---
apiVersion: v1
kind: Service
metadata:
  annotations:
  name: argocd-external-name
spec:
  type: ExternalName
  externalName: argocd-server.argocd.svc.cluster.local
---
apiVersion: v1
kind: Service
metadata:
  annotations:
  name: haproxy-mysql-external-name
spec:
  type: ExternalName
  externalName: mysql-ha-haproxy.default.svc.cluster.local  
---
apiVersion: v1
kind: Service
metadata:
  annotations:
  name: haproxy-jenkins-external-name
spec:
  type: ExternalName
  externalName: jenkins-ha-haproxy.devops-system.svc.cluster.local  
```


**aws_route53domains_registered_domain** 리소스는 호스팅 영역을 생성하며 자동으로 함께 생성된 4개의 네임서버 레코드들을, Route53에 등록되어있는 choigonyok.com 도메인에 입력시켜주는 리소스이다.

이 리소스를 통해서, 앞서 말한 것처럼 ***.choigonyok.com**으로 접근하는 요청들이 호스팅 영역이 위치한 DNS에 입력된 도메인에 대한 질의를 할 수 있게된다.

## 5. ACM 테라폼 구현


ACM은 AWS Certificate Manager의 준말로, AWS에서 제공하는 TLS 인증서 프로비저닝 및 관리 서비스이다.

테라폼으로 구현한 ACM 코드는 아래와 같다.

```go
resource "aws_acm_certificate" "techlog" {
  domain_name       = "choigonyok.com"
  subject_alternative_names = ["www.choigonyok.com"]
  validation_method = "DNS"
}

resource "aws_route53_record" "dvo" {
  for_each = {
    for dvo in aws_acm_certificate.techlog.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = aws_route53_zone.main.zone_id
}

resource "aws_acm_certificate_validation" "techlog" {
  certificate_arn         = aws_acm_certificate.techlog.arn
  validation_record_fqdns = [for record in aws_route53_record.dvo : record.fqdn]
}
```


우선 테라폼의 **aws** 프로바이더에서 제공하는 리소스인 **aws_acm_certificate**를 생성했다.

테크 블로그 어플리케이션으로 접근하는 도메인인 **choigonyok.com**와 **www,choigonyok.com**에 대한 인증서를 생성했다.

그리고 테라폼으로 생성했던 Route53 호스팅 영역에 CNAME 레코드를 추가한다. 클라이언트가 HTTPS로 접근했을 때, 이 CNAME 레코드를 통해 인증서를 확인할 수 있다.

----

이렇게 구현해서 **terraform apply -auto-approve** 커맨드만 입력하면 각 도메인들을 통해 테크 블로그, Jenkins, ArgoCD 대시보드에 접근할 수 있게되고, 테크 블로그 어플리케이션엔 HTTPS로 접근이 가능해지게 되었다.

## 6. References


[TLS(HTTPS)의 동작원리와 과정](https://cuziam.tistory.com/entry/TLSHTTPS%EC%9D%98-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC%EC%99%80-%EA%B3%BC%EC%A0%95)

[AWS Certificate Manager](https://aws.amazon.com/ko/certificate-manager/)