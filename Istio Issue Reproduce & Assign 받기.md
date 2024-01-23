[ID: 58]
		[Tags: ISTIO INFRA]
		[Title: Istio Issue Reproduce & Assign 받기]
		[WriteTime: 2024-01-21]
		[ImageNames: 8d61ff28-532e-4853-b944-1fe543080c05.png 283c517c-fa11-4296-87cb-3c5a99c7b604762843-28a7-4750-b17b-dba4e05cd68e.png d9ad7189-725c-46a9-ae4a-79e4354abe5c.png 31ce9d08-900f-4521-9163-f24af55d0f20.png b0565ecd-d1e5-41af-81a6-42d2204232a7.png 1d9d33c0-1541-4e11-9c5d-ff7f55285802.png 4f3c41f0-82d7-40aa-91bf-8ed13fc1800a.png 0efc0ea7-d5b5-4ef1-b09c-2da0002075a2.png 04762843-28a7-4750-b17b-dba4e05cd68e.png eaac0525-8c8f-46e6-aefb-c8a40c765bc2.png 369da3bf-6258-40d8-aaaf-595282222084.png 1d9ad7189-725c-46a9-ae4a-79e4354abe5c.png 231ce9d08-900f-4521-9163-f24af55d0f20.png 3b0565ecd-d1e5-41af-81a6-42d2204232a7.png 41d9d33c0-1541-4e11-9c5d-ff7f55285802.png]

## Content

1. Preamble
2. Issue Reproduce용 KinD 클러스터 구성
3. RBAC 설정
4. initContainer Volume Projection
5. 엠비언트 메시 내부 인바운드 트래픽 확인
6. 엠비언트 메시 내부 아웃바운드 트래픽 확인
7. Ambient Mesh가 아니면 해결이 가능하다?
8. 코드베이스 수정을 위한 Issue Assign 요청
9. References

## 1. Preamble

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/A4B23C6B-2085-40C5-A1B7-F44E17A54ECC_2/zl6MEF146fH616SfI2TNxZGSLPjwOf9Dm6h1wbvNcGIz/Image.png)

Istio 깃허브 레포에 새로운 이슈가 하나 등록되었다.

[Istio CNI blocks traffic in application init containers #48854](https://github.com/istio/istio/issues/48854)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/6F0BA732-0983-4437-A7C1-DC4EC6A5813C_2/kp58EBgNPmnXtwAzJf1nIWxWmNudixDE2loawxdalTEz/Image.png)

요약하자면 initContainer로 kubectl 이미지를 pull해서 커맨드를 실행하니 아래와 같은 에러메시지가 출력된다는 것이었다.

```go
E0118 09:57:46.188460       8 memcache.go:265] couldn't get current server API group list: Get "https://172.20.0.1:443/api?timeout=32s": EOF
E0118 09:57:56.195553       8 memcache.go:265] couldn't get current server API group list: Get "https://172.20.0.1:443/api?timeout=32s": EOF
E0118 09:58:06.202729       8 memcache.go:265] couldn't get current server API group list: Get "https://172.20.0.1:443/api?timeout=32s": EOF
Unable to connect to the server: EOF
```


kubectl이 쿠버네티스 API서버에 접근하지 못하는 것으로 보아 kubectl의 `./kube/config` 파일에 클러스터 데이터가 포함되어있지 않아서가 아닐까 하고 답변을 달았다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/35FF3FB1-1401-416D-9C57-F4D8BC066461_2/2n90DvBrVyGOx45hV8Sw187lyLTJKHN0PKax245A2lwz/Image.png)

이후에 issue Author로부터 엠비언트를 적용하지 않으면 잘 되는데, 앰비언트를 적용하기만하면 안되는 것이라는 추가적인 정보를 얻었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/6B498457-5B28-4C86-B379-3D063BB9C280_2/hHVkbpUcaLMMpUWgmAf4v4gOOa1HTQfuRW6K0Z8oLYsz/Image.png)

Istio의 컨트리뷰터 및 콜라보레이터들은 너무나 열성적이어서, 이슈가 생기자마자 기여할만한 이슈들에는 바로 Assignee들이 할당된다. 이 이슈를 내가 해결해보기로 했다.
> 이 글은 실제 Istio 오픈소스의 코드베이스에 기여하게된 과정을 담은 글이다.


## 2. Issue Reproduce용 KinD 클러스터 구성


우선 이슈를 Reproduce(복기)하기위해 KinD로 클러스터를 생성했다.

우선 Istio를 배포하지 않을 상태로 InitConatiner를 먼저 테스트해본 뒤, Istio를 배포한 이후의 결과와 비교하기로했다.

KinD는 멀티노드 클러스터를 지원하지만, 테스트의 간편함을 위해 단일 노드 클러스터를 생성했다.

```yaml
kind create cluster --config=- <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: test
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 80
    listenAddress: "127.0.0.1"
EOF
```


테스트용 `nginx` 이미지를 사용해 initContainer를 포함한 `Deployment`, `Pod`, `Service` 오브젝트를 배포했다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template: 
    metadata:
      labels:
        app: nginx
    spec:
      initContainers:
        - name: init-kubectl
          image: bitnami/kubectl
          command:
          - sh
          - "-c"
          - |
            sleep 15
            kubectl get pods
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      nodePort: 30000
      protocol: TCP
      targetPort: 80
  type: NodePort
```


Issue Author가 작성한 내용처럼 그 오류가 똑같이 생겼다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/1C7A1D64-72B0-4A0E-8F8A-4E1F5BC17911_2/rAE2rjSRL3MOJEVBySy7G1fsFlop36FnOLDfWFgrCqYz/Image.png)

## 3. RBAC 설정


`RBAC`은 Role Based Access Control의 준말로, 쿠버네티스에서 Role을 통해 여러 접근제어를 할 수 있는 기능이다. `Role`은 일반적으로 `ServiceAccount` 리소스나 사용자인 `User`에 바인딩된다. 쿠버네티스의 컨트롤 플레인에 접근하기 위해서는 이 RBAC이 필요하다.

Istio가 아닌 일반 쿠버네티스 클러스터에서 RBAC 없이 InitContainer에 배포된 kubectl을 통해 컨트롤 플레인의 API 서버에 접근하면 아래 이미지처럼 접근이 안된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/9678DE19-FE64-4C35-834E-374A66D84A31_2/aQ0xbMXLWNbsT1Jjr1KxqbsYI2E2xP1RNhZpLySAfxcz/Image.png)

InitContainer가 컨트롤플레인에 접근할 수 있는 권한이 없기 때문이다. 아무나 다 컨트롤플레인에 접근이 가능하다면 공격자가 `Privilege Escalation`을 통해 전체 클러스터 및 서비스에 접근이 가능해지게 되어 보안상 위험하다.

파드가 컨트롤 플레인에 접근하려면 ServiceAccount가 파드에 할당되어야하고, 그 서비스 어카운트에 컨트롤플레인에 접근할 수 있는 권한인 Role이 정의되어있어야하며, 이 Role을 서비스 어카운트에 할당시키는 RoleBinding이 정의되어있어야한다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
automountServiceAccountToken: true
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: list-pods-cluster-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: list-pods-cluster-role-binding
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: default
roleRef:
  kind: ClusterRole
  name: list-pods-cluster-role
  apiGroup: rbac.authorization.k8s.io
```


ServiceAccount, Role, RoleBinding을 배포하고, 배포할 파드의 manifest에 아래처럼 `serviceAccountName` 필드를 지정해서 파드에 서비스 어카운트를 할당해주었다.

```yaml
...
spec:
  ...
  template: 
    ...
    spec:
      serviceAccountName: my-service-account
...
```

>  다시 실행해보니 InitContainer에서 입력한 커맨드인 `kubectl get pods`가 정상적으로 출력되었다.


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/71393542-F7D6-4B8D-8B3B-69C83017E762_2/a02WKxdZ2AEpfDMXCAoWejrtZFyokRBCUHk0WMUXqfkz/%202024-01-21%20%201.57.53.png)

## Istio 엠비언트 메시 배포


이제 Istio 엠비언트 메시 내부에서 같은 작업을 실행해볼 차례이다.

Istio CLI 도구인 istioctl로 `ambient` 프로파일의 Istio를 설치했다.

```bash
istioctl install --set profile=ambient -y
```


Istio 엠비언트 메시에서는 `Namespace Label`을 지정하면 그 네임스페이스에 엠비언트 메시가 적용된다. 테스트 파드가 배치될 `default` namespace에 Label을 적용시켰다.

```bash
kubectl label namespace default istio.io/dataplane-mode=ambient
```

>  파드를 재배포하고 결과를 확인해봤지만, RBAC이 적용되어있음에도 에러가 발생했다.


## 4. initContainer Volume Projection


그래서 처음 내가 Issue Author에게 제안했던 것처럼, initContainer에 서비스 어카운트 토큰이나 kubectl이 클러스터에 접근하기위한 인증서가 정상적으로 전달되지 않을 것일 수도 있다고 판단했다.

그래서 토큰과 클러스터 인증서인 `ca.crt`를 `Volume`을 활용해 전달하기로 했다. 쿠버네티스에서는 이러한 Credential들을 볼륨으로 공유할 수 있도록 `Projected Volume`을 지원한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template: 
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: my-service-account
      volumes:
      - name: init-job
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 7200
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
      initContainers:
        - name: init-kubectl
          image: bitnami/kubectl
          volumeMounts:
          - name: init-job
            readOnly: true
            mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          command:
          - bash
          - "-c"
          - |
            sleep 15
            ls /var/run/secrets/kubernetes.io/serviceaccount
            cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
            cat /var/run/secrets/kubernetes.io/serviceaccount/token

            kubectl config set-cluster cfc --server=https://kubernetes.default --certificate-authority=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
            kubectl config set-context cfc --cluster=cfc
            kubectl config set-credentials user --token=/var/run/secrets/kubernetes.io/serviceaccount/token
            kubectl config set-context cfc --user=user
            kubectl config use-context cfc

            kubectl config current-context           
            kubectl get pods
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```


`volumes.projected.sources.serviceAccountToken`과 `volumes.projected.sources.configMap.items.key=ca.crt`를 통해서 인증서와 서비스 어카운트 토큰을 볼륨으로 전달할 수 있다.

bash 스크립트도 추가되었는데,


-  `kubectl config set-cluster cfc --server=https://kubernetes.default --certificate-authority=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt` 는 쿠버네티스 클러스터 인증서 ca.crt를 가지고 kubectl의 `cluster` 리소스를 생성하는 커맨드이다.
- `kubectl config set-context cfc --cluster=cfc` 는 ca.crt로 생성한 cluster를 가지고 kubectl의 `context` 리소스를 생성하는 커맨드이다.
- `kubectl config set-credentials user --token=/var/run/secrets/kubernetes.io/serviceaccount/token`는 서비스어카운트 토큰을 가지고 kubectl의 `user` 리소스를 생성하는 커맨드이다.
- `kubectl config set-context cfc --user=user` 는 생성한 user를 context에 매핑하는 커맨드이다.
- `kubectl config use-context cfc` 는 최종적으로 cluster와 user가 매핑된 context를 지금 kubectl가 사용하겠다고 지정시키는 커맨드이다.

context를 통해 kubectl은 하나의 CLI로 여러 클러스터를 관리할 수 있게된다.

이렇게 실행시킨 후 로그를 확인했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/826C0EF7-788F-4A30-A987-55F2323106CF_2/HPkBzUK6idjldKwxxtJnPlR1vgm8DUxxNUQyXMLjvLgz/%202024-01-20%20%2010.55.34.png)

cluster, user, context는 잘 생성되었고, `kubectl config current-context`의 결과로 `cfg`가 정상적으로 출력되었지만, 에러는 그대로였다.
>  Istio 엠비언트 메시를 적용하지 않았을 때 정상적으로 작동하고, 로그에도 `ca.crt`와 `token`을 통해 kubectl의 context, cluster의 설정이 정상적으로 완료된 것을 확인할 수 있었기 때문에, kubectl의 문제가 아닌 Istio의 문제인 것으로 확신할 수 있었다.


## 5. 엠비언트 메시 내부 인바운드 트래픽 확인


ztunnel의 로그를 확인해보았다. 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/E5987B0F-1808-4CF5-AAC1-64B954EF528E_2/qgHe2brvcOZNxo4WEHcFijfoVWuflhyrdA3Z5LYbvncz/%202024-01-20%20%2011.23.56.png)

`10.244.0.14`는 테스트용 nginx 파드의 IP주소이다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/AC438E7A-B65F-4F7B-9E95-091A40B96238_2/fsekTWGGLmOtvzlnJZ6cb7gvyQHdYot3NnS0zxnC2qkz/%202024-01-20%20%2011.24.46.png)

ztunnel이 nginx 파드를 `unknown source`로 인식하고있었다. 그래서 nginx에서 외부로 나가는 outbound 트래픽을 처리하지 않았던 것이다.

엠비언트 메시 내부에서 밖으로 나가는 outbount 트래픽이 아닌, 외부에서 내부로 들어오는 inbound 트래픽도 문제가 있는지 체크하기 위해 엠비언트 메시 labeling이 되어있지 않는 `test` 네임스페이스에 nginx 파드를 추가로 배포했다.

기본적으로 엠비언트 메시 내부에서는 `mTLS`를 자동으로 적용시키기 때문에, 외부(클러스터 외부와 ambient가 적용되지 않은 네임스페이스 포함)에서 엠비언트 메시로 접근하게 되면 접근이 막힌다.

이 경우 `STRICT` 로 되어있던 `PeerAuthentication`을 `PERMISSIVE`로 설정해주면 mTLS가 아닌 외부에서도 HTTP 통신으로 엠비언트 메시에 접근이 가능해진다.

```bash
kubectl apply -n default -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: peerauth
spec:
  mtls:
    mode: PERMISSIVE
EOF
```


그리고 엠비언트 메시 외부의 `test` 네임스페이스에서 엠비언트 메시 내부 `default` 네임스페이스로 curl 커맨드를 보낸 후 ztunnel의 로그를 살펴보았다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/D6D27D81-3A42-40EE-80E1-E70C2693679E_2/BpvwBQxb5mhVHuZi09rW4OiKJPKoSmZNIc0QmIdkPdkz/Image.png)

메시 외부에서 내부로 들어가는 트래픽을 `PeerAuthentication`에서 `PERMISSIVE`로 설정해두었기 때문에, passthrough된 것을 확인할 수 있었다.

여기서는 destination으로 `10.244.0.10:80`이 잘 지정되서 요청이 전달되었다.
>  엠비언트 메시 외부에서 내부로 들어오는 트래픽은 문제가 없고, 내부에서 외부로 나가는 트래픽이 문제란 것을 확인할 수 있었다.


## 6. 엠비언트 메시 내부 아웃바운드 트래픽 확인


엠비언트 메시 내부에 문제가 있는 것은 확인되었고, 메인컨테이너를 먼저 체크해보았다.

메인 컨테이너인 nginx에서 자기 자신에게 curl 요청을 보내보았고, 정상적으로 접근이 됐다.

```bash
curl nginx
```


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/73609018-6CC5-410C-A273-F5ECE98C9D7F_2/zf9F9FU1qmmaUxYIHB3BauHNxLmCvgtuQgcaMn8JxQQz/Image.png)

메시 외부에 배포했던 `test` 네임스페이스의 `test` 파드에도 아웃바운드 트래픽을 보내보았고, 정상적으로 접근이 되었다.

```bash
curl test.test.svc.cluster.local
```


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/47C78634-447F-4061-BD87-434C43EB1258_2/C1sfwfTsyoa9mzzePZ2JNsSIyQIfS35Ayex2p6FX7BIz/Image.png)
>  엠비언트 메시 내부에서 나가는 아웃바운드 트래픽, 그 중에서도 initContainer에서 나가는 트래픽에서만 에러가 발생한다는 것이 확인되었다.


## 7. Ambient Mesh가 아니면 해결이 가능하다?


관련해서 Close된 다른 이슈들과 Istio 커뮤니티를 찾아보다가,  initContainer에서는 mTLS가 적용되지 않는것이 원인이라는 글을 발견했다.

공식문서에도 관련된 내용이 있었는데, 보안적 위험성을 감수하고 HTTP 통신을 허용하는 `PeerAuthentication` 리소스의 `PERMISSIVE` 설정을 적용하거나, 어노테이션을 통해 특정 CIDR 블록과 포트에서만 사이드카 프록시로 트래픽이 인터셉트되지 않도록 지정해주어야한다고 말했다.

### Sidecar Exclude Annotation


```yaml
...
spec:
  ...
  template: 
    metadata:
      annotations:
        traffic.sidecar.istio.io/excludeOutboundIPRanges: 10.244.0.0/24
        traffic.sidecar.istio.io/excludeOutboundPorts: "80"
  template:
    ...
    spec:
      ...
      initContainers:
        ...
        securityContext:
          runAsUser: 1337
...
```


이 두 방식 모두 일반적인 Istio 서비스 메시에서는 정상적으로 동작하며, kubectl initContainer가 쿠버네티스 컨트롤 플레인의 API 서버에 접근해서 `kubectl get pods` 커맨드를 정상적으로 수행하는 것을 확인할 수 있었다.

문제는, 이 두 방식 모두 엠비언트 메시에서는 적용되지 않는다는 것이다.

이미 `PeerAuthentication`의 `PERMISSIVE` 설정은 해둔 상태였고, 어노테이션을 추가해봤지면 에러는 그대로 있었다.

## 8. 코드베이스 수정을 위한 Issue Assign 요청


위의 과정들을 통해 이 이슈가 그저 Istio를 미숙하게 다루는데에서 생기는 실수가 아니라 소프트웨어적 버그 혹은 아직 구현되지 않은 것으로 판단하게 되었고, 평소 Isito 레포에서 가장 활발히 활동하던 Istio 메인테이너를 호출해서 이 이슈를 할당해달라고 요청했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/C7D1B486-D7E1-4B00-8A6B-20E1BA369168_2/8x0C65teE5IF9wulVniblinfnCaGtBYn06FEe8Z3xHMz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/D918083C-9D83-4FF0-B11F-9ECC4DD9064D/E5DE0CAA-6F29-4AEA-A95D-55A5EE933C6D_2/3PLKoZe522Z0LNRghLrm8ryv4uLcJrDy6gkMYnYcqn8z/Image.png)

메인테이너가 아직 구현되지 않은 것이 아니라 버그인 것 같다는 답변을 달아주었고, 관련해서 버그를 찾아서 직접 고쳐보기로 했다.

이 이슈를 해결해나가는 과정을 앞으로 글로 쭉 써볼 예정이다.

## 9. References


[Issue - Istio CNI blocks traffic in application init containers](https://github.com/istio/istio/issues/48854)

[Istio Official Docs-Compatibility with application init containers](https://istio.io/latest/docs/setup/additional-setup/cni/#compatibility-with-application-init-containers)

[Istio Community-ISTIO CNI drops initContainers outgoning traffic](https://discuss.istio.io/t/istio-cni-drops-initcontainers-outgoing-traffic/2311)

[IBM-Using service account tokens to connect with the API server](https://www.ibm.com/docs/en/cloud-private/3.2.x?topic=kubectl-using-service-account-tokens-connect-api-server)

[K8s Official Docs-Launch a Pod using service account token projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#launch-a-pod-using-service-account-token-projection)

[K8s Official Docs-Manually create an API token for a ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-an-api-token-for-a-serviceaccount)

[Istio Official Docs-L4 Networking & mTLS with Ztunnel](https://istio.io/latest/docs/ops/ambient/usage/ztunnel/)

[Istio Blog-Introducing Ambient Mesh](https://istio.io/v1.15/blog/2022/introducing-ambient-mesh/)

[Service account token Volume Projection이란?](https://sa-na.tistory.com/entry/BoundServiceAccountTokenVolume-v121-Default)
