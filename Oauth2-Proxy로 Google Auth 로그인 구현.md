[ID: 56]
[Tags: projects dev sec]
[Title: Oauth2-Proxy로 Google Auth 로그인 구현]
[WriteTime: 2024-01-28]
[ImageNames: e75bb2c4-f83c-4a30-9bca-d67ab3dc050d.png]

## Content

1. Preamble
2. Oauth2.0 Flow
3. 굳이 Authorization Code 발급이 필요한 이유
4. 구현
5. References

## 1. Preamble


직접 개발해서 운영하고 있는 기술블로그 서비스에서, 유일한 페이지 관리자인 나를 인증하기 위해 환경변수를 사용했었다.


-  `.env`

```yaml
BLOG_ID=choigonyok
BLOG_PW=devops
```


클라이언트에서 입력한 아이디와 패스워드를 백엔드에서 검증해야했기 때문에, 환경변수 파일을 깃허브 공유저장소에 푸시해서 백엔드가 읽을 수 있게 구성해야했다. 따라서 누군가 공유저장소의 블로그 소스코드에서 관련 크리덴셜을 확인하게되면 아무나 관리자로 로그인해서 블로그 게시글들을 조작할 수 있다는 보안적 위험성이 존재했다.

Oauth2 Proxy 오픈소스를 활용해 인가과정을 백엔드에서 직접 검증하지 않고, IdP를 통해 인가하는 Oauth2.0 프로토콜을 적용하기로 했다.

Oauth2.0 프로토콜 Flow에서 Authorization Code를 발급받는 과정이 굳이 왜 필요한 것인지에 대해 이해하고, 이 내용을 Oauth2 Proxy 오픈소스를 통해 블로그 서비스에 실제 구현한 과정을 기록한 글이다.

## 2. Oauth2.0 Flow


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/86E04419-0C69-4AE4-8998-B905AFAF11CD/8B99D113-E215-4102-904B-8504CA77FC14_2/qQV7lsMaLCKXaoPFQhZyaX9fdpesJxpNREax4PkKd3Qz/Image.png)

Oauth2.0 프로토콜의 플로우는 `implict` Flow와 `authorization code` Flow, 크게 두 가지로 나뉜다. 

Implict 플로우는 인가코드를 요청하고 받는 과정을 생략하고, 인가가 확인되면 바로 엑세스 토큰을 발급받는 방식이다.

Authorization code flow는 Implict flow 중간에 authorization code를 요청하고 받는 과정이 포함되어있다. 클라이언트가 인가코드를 요청하면 IdP는 로그인 페이지를 응답해서 사용자가 로그인하도록 만들고, 로그인에 성공하면 엑세스 코드가 아닌 인가코드를 응답해준다. 이 Authorization code 플로우가 더 보안상 이점이 있기 때문에 이 플로우를 적용하기로 했다.

사용자가 Admin 페이지에 접근하려고한다고 가정해보자.

* 먼저 어드민 페이지에 접속하고자하는 사용자가 클라이언트에 서비스 사용 요청을 한다. Admin 페이지로 접속하는 것이다.

* 클라이언트는 서버에 인가를 요청한다.

* 서버는 서버가 인가코드를 받기 위해 서버로 리다이렉트 되어야하는 URL, 그리고 IdP 로부터 제공받기 원하는 사용자 데이터 목록을 scope에 정의해서 URL에 포함시킨다. 그리고 클라이언트가 IdP로 리다이렉트 되도록 응답한다.

* 클라이언트는 서버에 의해 IdP로 리다이렉트된다.

* IdP는 로그인 페이지를 클라이언트에 응답한다.

* 사용자는 이 로그인 페이지에 아이디와 패스워드를 입력해 IdP에 로그인 요청을 보낸다.

* 로그인을 통해 인가가 되면, IdP는 인가코드를 URL을 쿼리에 포함시켜서 리다이렉트 URL로 클라이언트가 리다이렉트되도록 응답한다.

* 클라이언트는 IdP의 응답을 서버에게 리다이렉트한다.

* 서버는 URL 쿼리의 인가코드와, 가지고있는 client_id, client_secret을 가지고 IdP에 엑세스 토큰을 요청한다.

* IdP는 인가코드, client_id, client_secret의 validation을 체크한 이후에 엑세스 토큰을 발급하고 서버에 전달한다.

* 서버는 발급된 엑세스 토큰을 가지고 IdP에 사용자 데이터를 요청한다.

* IdP는 엑세스 토큰 유효성 검증이 완료되면, 토큰의 scope에 명시된 사용자 데이터를 서버에 응답한다.

* 서버는 이 사용자 데이터를 클라이언트에 전달하거나 필요한 작업을 수행한다.

## 3. 굳이 Authorization Code 발급이 필요한 이유


플로우애서 의문이 생겼다. 왜 IdP는 로그인을 통해 인가가 확인된 클라이언트에게 바로 엑세스 토큰을 전달하지 않고 인가코드를 전달해서 인가코드를 통해 엑세스 토큰을 발급하는 복잡한 과정을 거치는 것일까? 이 과정을 생략한 플로우가 바로 implict flow이다.

만약 인가코드 발급 과정이 없다면 클라이언트는 토큰을 받기위해 요청을 보낼 때 client_id와 client_secret을 함께 포함해서 요청을 보내야한다.

이 기록은 브라우저에 로그로 남아있을 수 있어 보안상 위험하다.

그래서 client_id와 client_secret 등의 크리덴셜을 클라이언트가 아닌 서버에서 관리하기 위해 클라이언트에는 인가코드만 전달하고, 클라이언트가 직접 엑세스토큰을 요청하는 게 아닌 인가코드를 서버에 전달해서 서버가 자신이 가지고있는 client_id와 client_secret을 포함해 엑세스 토큰을 요청하고 발급받을 수 있도록 하는 것이다.

또 토큰이 클라이언트에게 응답될 때, 이 토큰 역시 브라우저에 노출이 되고, 권한이 없는 누구나 엑세스 토큰을 활용해서 해당하는 리소스에 접근할 수 있게된다.
> 결론

> -  client_id와 client_secret을 브라우저(클라이언트)에 노출시키지 않게 하기 위해서. 노출되면 누구나 이 크리덴셜을 통해 엑세스토큰을 발급받을 수 있기 때문에
> - 토큰을 브라우저(클라이언트)에 노출시키지 않게 하기 위해서. 노출되면 누구나 이 엑세스토큰을 통해 리소스에 접근할 수 있게되기 때문에

인가코드 발급 과정을 추가하게되면 중간자에 의해 인가코드가 탈취당해도 서버에서 관리하고있는, 엑세스 토큰을 발급받기 위한 client_id, client_secret 등의 크리덴셜을 알 수 없기 때문에 리소스에 함부로 접근할 수 없어서 보안적으로 더 안전하다.

## 4. 구현


인가가 필요한 리액트 컴포넌트가 렌더링될 때마다, 인가 여부를 확인하도록 `/oauth2/auth` 경로에 GET 요청을 보내도록 코드를 작성헀다. Oauth2 Proxy는 /oauth2/auth 경로에 GET 요청을 받으면, 인가가 되어있을 경우엔 아래처럼 `202` 상태코드를 응답하고,

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/86E04419-0C69-4AE4-8998-B905AFAF11CD/A3E7C38F-554D-49CD-8CE5-1EBA4A6F22D7_2/vhbSc4MOWqmyPg2y0Y6xoPg83tqptN5vZxb7BF6FZ4Mz/Image.png)

인가가 되어있지 않으면 아래처럼 `401` 상태코드를 응답한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/86E04419-0C69-4AE4-8998-B905AFAF11CD/18AE622C-900D-4BD3-AF70-4B3AA3BB505E_2/HN7gmIBJ2OxIhVYnuPED6QbpMP65jsVSRSj6ed2cQx8z/Image.png)

```javascript
useEffect(()=>{
  axios
  .get(process.env.REACT_APP_HOST + "/oauth2/auth")
  .then((response) => {
    if (response.status !== 202) {
      navigate("/");
    }
  })
  .catch((error) => {
    if (error.response.status === 401) {
      window.location.href = "https://www.choigonyok.com/oauth2/sign_in";
    } else {
      console.error(error);
    }
  });
},[])
```


만약 401 상태코드를 응답받으면 아직 인가가 되지 않았다는 의미이기 때문에, Oauth2 Proxy가 인가 사이클을 시작하는 경로인 `/oauth2/sign_in` 경로로 접속하도록 구현했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/86E04419-0C69-4AE4-8998-B905AFAF11CD/1ACDA466-BD7B-448E-9A65-D9BA56319481_2/N4WyWPm15TEZEeapWHSFoGYE3sWIxysNN1ntI4Jj7tkz/Image.png)

버튼을 클릭하면 구글 Login 페이지가 렌더링된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/86E04419-0C69-4AE4-8998-B905AFAF11CD/AE0959BA-051F-4B9D-95F7-78C7697CB94B_2/CnYRphS7Wvjx97FI2oTMyxkL2UAxKUcwLC0VeaA9PPsz/Image.png)

그리고 Oauth2 Proxy의 Deployment 오브젝트 스펙에서 initContainer를 추가했다. 이 initContainer는 나의 이메일 주소인 `achoistic98@gmail.com`을 email.txt 파일로 생성해서 /etc/auth 경로에 저장하는 작업을 수행한다.

이 email.txt 파일은 Oauth2 Proxy의 `authenticated-emails-file` 옵션을 통해 읽어지는데, 이 옵션을 설정하면 Oauth2 Proxy는 해당 파일을 한 라인씩 읽고, Oauth2를 통해 IdP로부터 받은 email주소와 비교하게된다.

비교한 결과, 로그인이 정상적으로 되었더라도 이 email.txt 파일에 존재하지 않는 이메일일 경우, 접근에 대한 요청을 드랍시킨다. 이 옵션을 통해 Oauth2 Proxy에서 인가 말고도 인증까지 수행할 수 있게되는 것이다.

위 사진에서 `achoistic98@gmail.com` 이 아닌 `levor0805123@gmail.com` 으로 로그인을 수행하면, 아래 사진처럼 Forbidden 403 에러가 출력된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/86E04419-0C69-4AE4-8998-B905AFAF11CD/2F123ECF-0536-4978-973D-B8B9FB4A114F_2/fqnZEfTGlZD1QKXlEMd8MGstF3jHXycxyxSD5t44AkYz/Image.png)

initContainer와 Oauth2 Proxy는 같은 노드, 같은 파드 안에 생성되는 컨테이너지만, 컨테이너라는 격리된 환경에서 생성되었기 때문에, email.txt 파일을 메인 컨테이너인 Oauth2 Proxy가 접근할 수 있도록 하기 위해 쿠버네티스 볼륨 오브젝트를 생성하고 마운트시켰다.

```javascript
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oauth2-proxy
  template: 
    metadata:
      labels:
        app: oauth2-proxy
    spec:
      initContainers:
      - name: insert-email-file
        image: bash:latest
        command:
        - bash
        - "-c"
        - |
          cd /etc/auth
          echo "achoistic98@gmail.com" > email.txt
        volumeMounts:
        - name: authenticated-files
          mountPath: /etc/auth 
      containers:
      - name: oauth2-proxy
        image: quay.io/oauth2-proxy/oauth2-proxy:v7.5.1
        ports:
        - containerPort: 4180
        args:
        - --provider=google
        - --email-domain=""
        - --upstream="http://frontend.default.svc.cluster.local"
        - --http-address=0.0.0.0:4180
        - --cookie-secure=false
        - --authenticated-emails-file="/etc/auth/email.txt"
        - --redirect-url="https://www.choigonyok.com/oauth2/callback"
        - --whitelist-domain="choigonyok.com"
        - --cookie-domain="choigonyok.com"
        env:
        - name:  OAUTH2_PROXY_COOKIE_SECRET
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-secrets
              key: cookie-secret
              optional: false
        - name:  OAUTH2_PROXY_CLIENT_ID
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-secrets
              key: client-id
              optional: false
        - name:  OAUTH2_PROXY_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-secrets
              key: client-secret
              optional: false
        - name:  OAUTH2_PROXY_SET_XAUTHREQUEST
          value: "true"
        volumeMounts:
        - name: authenticated-files
          mountPath: /etc/auth
      volumes:
        - name: authenticated-files
          hostPath:
            path: /etc/auth
```


Oauth2 Proxy의 manifest도 공유저장소에 푸시될텐데, 스펙에서 정의한 환경변수들이 함께 푸시된다면 보안적으로 위험하다. 따라서 환경변수 값을 하드코딩하지 않고, 쿠버네티스 시크릿 리소스를 참조하는 방식으로 구현했다.

이 크리덴셜이 공유저장소에 푸시되면 인가코드 과정을 통해 클라이언트에 크리덴셜이 노출되지 않도록 한 의미가 없어진다. 누구나 공유저장소를 통해 크리덴셜을 확인할 수 있게되기 때문이다.

따라서 이 시크릿 리소스에는 Client_id, Cliend_secret, Cookie_secret 등의 데이터가 들어가있는데, 이 시크릿은 서버가 스핀업될 때 kubectl CLI를 통해 주입시키도록 스크립트를 작성했다.

`inject-secrets.sh`

```bash
if [[ -f ../.env ]]; then
  echo ".env file found"
  source ../.env

else
  echo ".env file not found"
  exit 1
fi

kubectl create secret generic oauth2-proxy-secrets \
--from-literal=cookie-secret=$COOKIE_SECRET \
--from-literal=client-id=$CLIENT_ID \
--from-literal=client-secret=$CLIENT_SECRET``
```


`.env` 파일은 로컬에만 존재하고, `.gitignore`에 .env 파일을 명시해서 깃허브 공유저장소에 푸시되지 않도록 설정해두었다.

이렇게 구성함으로써 공유저장소에 크리덴셜들이 노출되지 않으면서도 로컬에서 스크립트를 통해 블로그 서비스를 스핀업할 때 필요한 환경변수들을 적절히 주입할 수 있게되었다.

쿠버네티스 Ingress 리소스도 수정해주었다. Oauth2 Proxy의 default Proxy-Prefix는 `/oauth2`이다. `--proxy-prefix` 옵션을 통해 접두사를 수정할 수도 있지만, 따로 수정하지 않고 그대로 인그레스에 적용했다.

```javascript
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
      ...
      - path: /oauth2
        pathType: Prefix
        backend:
          service:
            name: oauth2-proxy
            port:
              number: 4180
      ...
```


이 ingress를 통해 클라이언트에서 `/oauth2/auth` 경로나 `/oauth2/sign_in`, 엑세스 코드를 얻기 위해 redeem을 하는 경로인 `/oauth2/callback` 경로로 요청을 보낼 때 요청이 정상적으로 Oauth2 Proxy로 전달이 될 수 있게되었다.

위 과정을 통해 admin 페이지에 접속할 때 구글 소셜 로그인 인가/인증을 거치도록 구현되었고, 한 번 인가를 마치게되면 Oauth2 Proxy에서 발급한 쿠키의 Expiration 기간 전까지는 쿠키를 통해 매번 구글 로그인을 하지 않고도 인가를 유지시킬 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/86E04419-0C69-4AE4-8998-B905AFAF11CD/C3D0B433-66D7-4E16-992A-3A25BD0A3691_2/2T6wJyghvE2tNrdoRxgl1GRS8zUYa26hzyx12yu6k6cz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/86E04419-0C69-4AE4-8998-B905AFAF11CD/17CE27E7-571E-4B9B-8E6C-9FE85AACD615_2/fOfMxT2krEvynF9r2gpUHepdZtQG1UpeY2FmZsxA09Az/Image.png)

## 5. References 


[Why does OAuth server return a authorization code instead of access token in the first step?](https://www.quora.com/Why-does-OAuth-server-return-a-authorization-code-instead-of-access-token-in-the-first-step)
