[ID: 55]
[Tags: istio golang kubernetes]
[Title: Istio-cni란? & 코드베이스 분석]
[WriteTime: 2024-01-17]
[ImageNames: dbd1770c-20f5-4c85-b7d9-72d69e7b23a4.png]

## Content

1. Preamble
2. InitContainer
3. Istio CNI Plugin
4. CNI Plugin의 장점
5. CNI Plugin이 iptables를 구성하는 과정
6. References

## 1. Preamble

> The Istio CNI plugin is **a replacement for the istio-init container that performs the same networking functionality but without requiring Istio users to enable elevated Kubernetes RBAC permissions**.


요약하면, Istio 쿠버네티스 RBAC의 권한 상승을 요구하지 않고도 Istio-init container와 동일하게 동작하도록 도와주는 CNI 플러그인이라고 한다.

RBAC의 권한 상승은 보안적으로 위험성이 존재한다. Istio는 정말 완성도 높은 오픈소스 프로젝트이지만, 언제나 버그는 발생하고 취약점을 통해 공격당했을 때, escalation된 권한을 통해 다른 쿠버네티스 리소스(시크릿 등)에도 접근할 수 있는 가능성이 있다.

Istio CNI 플러그인은 이런 부분을 보완한 Istio의 부가 기능이라고 말할 수 있겠다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/77897DA3-409E-4890-A6A3-AB233477898A/7120E481-5C1C-4EF3-A527-37F902B542B4_2/ckngWMQpIUF8PEorrv4UYExkExqyHWmskyaKaqDOx8wz/Image.png)

## 2. InitContainer


InitContainer는 어플리케이션 컨테이너가 실행되기 이전에 먼저 실행되는 컨테이너이다. 이 컨테이너를 통해 어플리케이션에 대한 사전 설정 등을 미리 해둘 수 있다.

### 내가 initContainer를 활용했던 방식


블로그 서비스에서, 현재는 MySQL 데이터베이스의 Master/Slave 설정을 위해 initContainer를 활용했었다.

```go
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-ha
spec:
  serviceName: mysql-ha
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      initContainers:
        - name: init-mysql
          image: achoistic98/blog_mysqls:latest
          command:
          - bash
          - "-c"
          - |
            set -ex
            [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
            ordinal=${BASH_REMATCH[1]}
            if [[ $ordinal -eq 0 ]]; then
              cp /cfg/mysql-master.cnf /mnt/
              cp /cfg/init-master.sql /mnt/
            else
              cp /cfg/mysql-slave.cnf /mnt/
              cp /cfg/init-slave.sql /mnt/
            fi
          volumeMounts:
          - name: local-vol
            mountPath: /mnt/
      containers:
        - name: database
          image: mysql:latest
          ports:
            - containerPort: 3306
          volumeMounts:
          - name: local-vol
            mountPath: /etc/mysql/conf.d/
          - name: local-vol
            mountPath: /docker-entrypoint-initdb.d/
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
            - name: MYSQL_DATABASE
              value: "blogdb"
            - name: MYSQL_USER
              value: "slave"
            - name: MYSQL_PASSWORD
              value: "password"
      volumes:
        - name: database-volume
          hostPath:
            path: /volumes
  volumeClaimTemplates:
  - metadata:
      name: local-vol
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-storage"
      resources:
        requests:
          storage: 1Gi
```


`StatefulSet` 리소스는 파드가 클러스터에 배치될 때, 랜덤스트링이 생성되지 않고, 파드 명 뒤에 `-0`, `-1` 이런 식으로 숫자가 붙어서 파드가 생성된다.

위 코드의 경우에는 이름이 `mysql-ha` 로 설정되어있기 때문에, 파드의 이름은 `mysql-ha-0`, `mysql-ha-1`, `mysql-ha-2`로 생성된다.

Master/Slave 데이터베이스 구성을 위해서 DB 파드 하나는 `master`로, 나머지 둘은 `slave`로 생성해야했고, `StatefulSet`으로 매번 같은 이름으로 배치되는 파드의 이름을 파싱해서 `-0` 으로 파드 명이 끝나는 경우에만 `master`에 필요한 컨피그 파일을 컨테이너에 넣어주고, 나머지는 `slave`에 필요한 DB 컨피그 파일을 넣어주는 방식으로 `initContainer`를 활용했다.

### Istio-init


Istio는 기본적으로 메시에 배포되는 모든 파드 안에 `istio-init`이라는 default initContainer를 주입한다. 이 `istio-init`  는 iptables 규칙을 생성한다. 이 iptables를 통해 파드의 네트워크 트래픽은 Istio 사이드가 프록시로 리다이렉팅된다. 트래픽을 중간에 가로채는 역할을 하는 것이다.

`istio-init` 컨테이너가 iptables 규칙을 생성하기 위해서는 리눅스 커널의 `CAP_NET_ADMIN`과 `CAP_NET_RAW` `capability`가 필요하다. 이 두 capabilities는 네트워크 관련한 권한의 제어를 담당하는 capability이다.


- CAP_NET_ADMIN

이 Capability가 설정되면 IP 주소를 변경하거나, 방화벽 규칙을 설정하거나, 라우팅 테이블을 조작할 수 있는 권한을 갖게된다.


- CAP_NEW_RAW

이 Capability가 설정되면 `raw socket` 을 생성할 수 있고, 이 소켓은 패킷을 생성하거나, 패킷에 직접 접근해서 헤더를 조작할 수 있는 권한을 갖게된다.

### Privilege Escalation 


쿠버네티스 및 컨테이너 환경에서 컨테이너에 여러 권한을 부여하는 방식은 보안적으로 위험하다. Capability를 통해 특정 컨테이너에 권한을 부여하면, 이 권한은 컨테이너 내부에서만 유효한 것이 아니라 컨테이너가 배포된 호스트에도 유효한 권한이 된다. 

이 때 `Privilege Escalation`로인해 어플리케이션 전체가 큰 보안적인 위협에 노출될 수 있다. Privilege Escalation은 작은 부분 부터 시작해서 한 계단씩 오르며 점차 권한이 많아지는 것을 의미한다.

예를 들어, 여러 권한을 가진 컨테이너 딱 하나만 공격자에게 공격당했다고 가정하자. 그럼 이 컨테이너의 Capability로 부여된 권한을 통해, 공격자는 컨테이너가 배치된 호스트와 호스트에 배치된 다른 컨테이너들에게도 접근할 수 있게된다.

또 호스트에 접근하면, 호스트는 호스트 자체의 권한이 있기 때문에, 이 권한을 통해서 이 호스트와 통신하는 다른 호스트에게까지 접근이 가능해진다. 그러다 쿠버네티스 API 서버에까지 접근하게 되면 클러스터 전체, 서비스 전체에 접근할 수 있는 권한이 생길 수 있는 것이다.

## 3. Istio CNI Plugin


Istio CNI 플러그인을 사용하면 capability를 주지 않고도 이 istio-init 컨테이너를 완벽히 대체할 수 있다.

CNI 플러그인은 `Istio CNI DaemonSet` 을 통해 설치된다. 데몬셋으로 모든 노드에 하나씩 배포된 Istio CNI 데몬셋은 


1. 어플리케이션 파드가 이미 배치되어있으면 해당 파드를 kill 한다.
2. 각 노드마다 `CNI Network Plugin` 을 설치한다.

`CNI Network Plugin`은 사이드카 프록시로 리다이렉팅이 되도록 iptables를 구성한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/77897DA3-409E-4890-A6A3-AB233477898A/FC754293-8EFE-4157-9A82-DAC2563D52EA_2/VJxZzXajDeZFGyBsLdmRxojEXLAL7zbXPz4gRG2a9MMz/Image.png)

## 4. Istio CNI의 장점


### 보안


앞서 말했듯이 Capability를 사용하지 않기 때문에 보안적으로 `istio-init` initContainer 방식보다 안전하다. 

### LCM (LifeCycle Management)


커널의 Capability를 사용하지 않기 때문에, Kubernetes를 업그레이드할 때도 더 자유롭고, InitContainer 방식이 아니라 한 노드 안에 배치된 전체 파드가 하나의 CNI 플러그인을 통해 관리되기 때문에 각 컨테이너의 라이프 사이클 관리가 상대적으로 간편하다.

### 범용성


CNI라는 표준을 통해 만들어졌기 때문에, 쿠버네티스가 아닌 여러 환경에서 사용될 수 있다.

## 5. CNI Plugin이 iptables를 구성하는 과정


Istio CNI에 대해 알기 전에는 사이드카 프록시로 리다이렉팅하는 것이 매우 복잡한 줄 알았다.
> CNI 네트워크 플러그인이 iptables를 수정해서, 어플리케이션 파드인 `192.0.0.1:8080`로 트래픽이 오면 해당 파드의 사이드카 프록시인 `172.0.0.1:15001`로 리다이렉팅되도록, 또 사이드카 프록시의 IP인 `172.0.0.1:15001`에서 트래픽이 나간다면, 해당 트래픽을 어플리케이션 파드인 `192.0.0.1:8080`로 리다이렉팅하도록 하는 것이다. 마치 쿠버네티스의 `Service` 오브젝트의 역할과 유사하다.


원리는 쉬운데, 그럼 Istio CNI 플러그인은 어떻게 iptables를 수정할 수 있는 것인가가 가장 궁금했다. `istio-init` InitContainer에서는 iptables을 구성하기 위해 Capabilities가 필요했는데, Istio CNI 플러그인은 어떻게 Capability없이 iptables을 수정할 수 있는 것일까?

이 해답은 찾을 수가 없었다. 따라서 코드베이스를 보고 Istio-cni가 iptables를 구성하는 과정을 직접 분석해보기로 했다.
> 내용이 길고 복잡하기 때문에, 요약해서 정리된 결론만을 원한다면 아래 `Summary` 섹션으로 넘어가면 된다.


## Directory


cni 관련 코드들은 istio/istio 레포의 루트 디렉토리인 /cni에 위치한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/77897DA3-409E-4890-A6A3-AB233477898A/4A413541-FF38-4AFA-B425-9C22F8CB16E3_2/AZzsOQvUENd1KYRRH0G49V1NjVxhq0TKJXrJxtg2AWQz/Image.png)

`cni/cmd/istio-cni/main.go` 에서  `"[github.com/containernetworking/cni/pkg/skel](http://github.com/containernetworking/cni/pkg/skel)"` 패키지를 import 하고있다. 이 패키지는 오픈소스 프로젝트 CNI의 패키지로, CNI 플러그인을 구현하도록 해주는 패키지이다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/77897DA3-409E-4890-A6A3-AB233477898A/8698E622-0C48-404E-A654-726584A6969F_2/uO2SrRETQWBKtlslIPfXohE4VwM2i6xbWRr2Tw14wNgz/Image.png)

CNI의 skel 패키지 함수인 `PlugMain`으로 `add`, `check`, `delete` 커맨드를 추가한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/77897DA3-409E-4890-A6A3-AB233477898A/FD15A5C1-4202-4E96-A6E5-4260A745906C_2/LoNEKbIJFyLXPwkgDUZ73ijT714TjyRo5hn0hnZRGL4z/Image.png)

### CmdAdd()


`CmdAdd()` 함수는 `cni/pkg/plugin/plugin.go` 에 정의되어있다. 이 함수는 `parseConfig()`로 인풋에 들어온 config를 파싱하고, 해당 config를 바탕으로 `doRun()`를 통해 실행시킨다.

```go
func CmdAdd(args *skel.CmdArgs) (err error) {
  ...
  conf, err := parseConfig(args.StdinData)
  if err != nil {
      log.Errorf("istio-cni cmdAdd failed to parse config %v %v", string(args.StdinData), err)
      return err
  }
  if err := doRun(args, conf); err != nil {
      return err
  }
  ...
}
```


### doRun()


`doRun()` 함수는 파드명과 파드 네임스페이스, 파드 IP 등을 필드로 가지는 `K8sArgs` 구조체를 통해 현재 파드의 메타데이터를 파싱하고,

```go
func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  k8sArgs := K8sArgs{}
  if err := types.LoadArgs(args.Args, &k8sArgs); err != nil {
      return err
  }
  ...
}
```


사용자가 쿠버네티스의 인터셉트 룰을 따로 설정했는지를 파싱해서 가져온다. Default로는 `interceptRuleMgrType`로 `iptables`가 초기화된다.

쿠버네티스에서 인터셉트 룰은 별도의 CRD를 통해 설정이 가능한 것으로 보인다.

```go
apiVersion: getambassador.io/v1alpha1
kind: InterceptSpecification
metadata:
  name: echo-server-spec
  namespace: my-crd-namespace
spec:
  prerequisites:
  - create: build-binary
    delete: rm-binary
  connection:
    context: "shared-cluster"
    mappedNamespaces:
      - "my_app"
  workloads: 
    echo-server:
      intercepts:
        - headers: 
            x-intercept-id: foo
          service_port: 8080
          handler: echo-server
  handlers:
    - name: echo-server
      docker:
        image: thhal/echo-server:latest
```


istio CNI에서 `excludeNamespaces`를 설정할 수 있다. 아래 구성으로 Istio를 배포하면 해당 네임스페이스에서는 사이드카 프록시가 주입되지 않고, 어노테이션을 통해 사이드카 프록시를 주입해도 해당 네임스페이스에서 istio-cni를 사용하지 않기로 설정했기 때문에 트래픽이 사이드카 프록시로 리다이렉팅 되지 않는다.

```go
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: empty # Do not include other components
  components:
    cni:
      enabled: true
  values:
    cni:
      excludeNamespaces:
        - istio-system
        - kube-system
```


Istio 코드베이스에서는 파드가 배치되어있는 네임스페이스가  excludeNamespaces로 지정된 네임스페이스인지를 체크한다.

```go
func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  for _, excludeNs := range conf.Kubernetes.ExcludeNamespaces {
      if podNamespace == excludeNs {
          log.Infof("pod namespace excluded")
          return nil
      }
  }
  ...
}
```


그리고 인풋으로 받은 config를 파라미터로 하는 `newKubeClient()`로 `*kubernetes.Clientset` 타입의 객체 `client`를 리턴받는다.

```go
func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  client, err := newKubeClient(*conf)
  if err != nil {
    return err
  }
  ...
}
```


newKubeClient 함수는 `cni/pkg/plugin/kubernetes.go`에 정의되어있고, 실제 아래 쿠버네티스 패키지들을 import해서 쿠버네티스 클라이언트를 생성하는 코드는 `/pkg/kube` 경로에 별도로 정의되어있다.

```go
import {
  corev1 "k8s.io/api/core/v1"
  "k8s.io/apimachinery/pkg/api/errors"
  metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  "k8s.io/apimachinery/pkg/runtime"
  "k8s.io/apimachinery/pkg/runtime/serializer"
  "k8s.io/client-go/kubernetes"
  _ "k8s.io/client-go/plugin/pkg/client/auth"
  "k8s.io/client-go/rest"
  "k8s.io/client-go/tools/clientcmd"
  "k8s.io/client-go/tools/clientcmd/api"
}
```


그 이후에는 인풋 config에서 Istio의 ambient mesh를 사용하도록 구성되어있는 경우의 처리를 하는데, 앰비언트 메쉬 관련은 생략한다.

```go
func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  if conf.AmbientEnabled {
    ...
  }
  ...
}
```


그리고 생성한 쿠버네티스 클라이언트 객체와 파드명, 네임스페이스명을 통해 `getKubePodInfo()` 함수를 호출해 파드 데이터를 가져온다. 실패할 경우 1초로 설정된 `podRetrievalInterval`만큼 쉬고 30초로 설정된 `podRetrievalMaxRetries` 수 만큼 시도하도록 for문이 작성되어있다.

```go
var (
	podRetrievalMaxRetries = 30
	podRetrievalInterval   = 1 * time.Second
)

func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  for attempt := 1; attempt <= podRetrievalMaxRetries; attempt++ {
    pi, k8sErr = getKubePodInfo(client, podName, podNamespace)
    if k8sErr == nil {
        break
    }
    log.Debugf("Failed to get %s/%s pod info: %v", podNamespace, podName, k8sErr)
    time.Sleep(podRetrievalInterval)
  }
  ...
}  
```


만약 이미 파드가 istio-init InitContainer로 생성이 되어있다면, 앞서 말했듯이 기존에 생성된 파드를 kill하고 재생성해야하기 때문에 이를 체크하기 위해 `istio-init`이라는 구성이 컨테이너에 되어있는지를 체크한다.

```go
const (
	ISTIOINIT  = "istio-init"
}

func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  if pi.Containers.Contains(ISTIOINIT) {
      log.Infof("excluded due to being already injected with istio-init container")
      return nil
  }
  ...
}
```


그리고 컨테이너 구성에 `istio-proxy`라는 구성이 되어있는지 체크한다. istio에서 모든 사이드카 프록시의 이름은 `istio-proxy`로 설정되어있기 때문에, 이 체크가 false라면 사이드카 프록시가 아직 배포되지 않은 것으로 이해할 수 있다.

```go
const (
	ISTIOPROXY = "istio-proxy"
)

func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  if !pi.Containers.Contains(ISTIOPROXY) {
      log.Infof("excluded because it does not have istio-proxy container (have %v)", sets.SortedList(pi.Containers))
      return nil
  }
  ...
}
```


파드의 프록시 타입 구성이 비어있거나 "sidecar" 중 해당하는 게 없다면 에러가 발생하도록 예외처리도 했다.

```go
func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  if pi.ProxyType != "" && pi.ProxyType != "sidecar" {
      log.Infof("excluded %s/%s pod because it has proxy type %s", podNamespace, podName, pi.ProxyType)
      return nil
  }
  ...
}
```


그 다음으로는, 해당 파드에 사이드카 프록시 주입이 설정되어있는지를 체크한다.

```go
var (
	injectAnnotationKey = annotation.SidecarInject.Name
}
  
func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  val := pi.Annotations[injectAnnotationKey]
  if lbl, labelPresent := pi.Labels[label.SidecarInject.Name]; labelPresent {
      val = lbl
  }
  if val != "" {
      log.Debugf("contains inject annotation: %s", val)
      if injectEnabled, err := strconv.ParseBool(val); err == nil {
          if !injectEnabled {
              log.Infof("excluded due to inject-disabled annotation")
              return nil
          }
      }
  }
  ...
}
```


Istio에서는 두 가지 방식으로 사이드카 프록시 인젝션 구성을 할 수 있다.


- 파드의 label 설정을 통해

```go
metadata:
  labels:
    istio-injection: enabled
```

- 파드의 어노테이션 설정을 통해

```go
metadata:
  annotations:
    sidecar.istio.io/inject: "true"
```


`injectAnnotationKey`는 `istio.io/api/annotation` 패키지에서 "sidecar.istio.io/inject"로 초기화 되어있다.

이 키를 사용해서 map 타입의 `Annotations`에 키에 대한 값을 가져오고, Label에 주입 Label인 `istio-injection: enabled` 가 설정되어있다면 이 값으로 덮어씌운다.

만약 비어있으면 에러를 발생시킨다.

그 다음으로는 `"sidecar.istio.io/status"`로 초기화된 `sidecarStatusKey`가 map 타입 `Annotations`에 정의되어있는지 체크한다. 
> 이 어노테이션은 Istio 사이드카 프록시가 파드에 주입되면 자동으로 생기는 어노테이션으로, 주입 상태를 나타내는 어노테이션이다. 이 어노테이션이 없으면 사이드카 프록시가 정상적으로 주입되지 않은 것으로 판단하게된다.

```go
var {
  sidecarStatusKey = annotation.SidecarStatus.Name
}

func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  if _, ok := pi.Annotations[sidecarStatusKey]; !ok {
  		log.Infof("excluded due to not containing sidecar annotation")
  		return nil
  }
  ...
}
```


```go
func doRun(args *skel.CmdArgs, conf *Config) error {
  ...
  redirect, err := NewRedirect(pi)
  ...
}
```


`NewRedirect()` 함수는 `cni/pkg/plugin/redirect.go`에 정의되어있다. 이 함수를 통해 파드에 대한 리다이렉트 구성을 하게된다. 내가 알고싶었던, `어떻게 iptables를 capability 없이 수정할 수 있는가`에 대한 답을 줄 부분처럼 보인다.

이 함수는 아래서 다시 살펴보기로 하고, `doRun()`에서는 `NewRedirect()` 함수로 파드에 대한 리다이렉션 설정을 마친 후 `*Redirect` 객체를 리턴받는다.

그리고 새 iptables을 생성한다.

```go
	interceptMgrCtor := GetInterceptRuleMgrCtor(interceptRuleMgrType)
	if interceptMgrCtor == nil {
		log.Errorf("Pod redirect failed due to unavailable InterceptRuleMgr of type %s", interceptRuleMgrType)
		return fmt.Errorf("redirect failed to find InterceptRuleMgr")
	}
```


`cni/pkg/plugin/intercept_rule_mgr.go`에는 `Program()`이라는 메서드를 가진 구조체들을 `InterceptRuleMgr` 인터페이스로 선언하고 있다. 

```go
package plugin

const (
	defInterceptRuleMgrType = "iptables"
)

type InterceptRuleMgr interface {
	Program(podName, netns string, redirect *Redirect) error
}

type InterceptRuleMgrCtor func() InterceptRuleMgr

var InterceptRuleMgrTypes = map[string]InterceptRuleMgrCtor{
	"iptables": IptablesInterceptRuleMgrCtor,
}

func GetInterceptRuleMgrCtor(interceptType string) InterceptRuleMgrCtor {
	return InterceptRuleMgrTypes[interceptType]
}

func IptablesInterceptRuleMgrCtor() InterceptRuleMgr {
	return newIPTables()
}
```


리턴받은 `interceptMgrCtor`를 `rulesMgr`에 복사해서 `Program()` 메서드를 호출한다.

```go
	rulesMgr := interceptMgrCtor()
	if err := rulesMgr.Program(podName, args.Netns, redirect); err != nil {
		return err
	}
```


이 메서드는 `cni/pkg/iptables/unspecified.go`에 정의되어있는데, 

```go
package plugin

import "errors"

var ErrNotImplemented = errors.New("not implemented")

func (ipt *iptables) Program(podName, netns string, rdrct *Redirect) error {
	return ErrNotImplemented
}
```


"not implemented"를 반환하는 에러인 `ErrNotImplemeted`를 리턴하고있다.

주석에 따르면 Program 메서드는 파라미터에 맞게 iptables을 구성하는 메서드인 것 같은데, 구현이 아직 되지 않았다고 한다.

## 6. Summary


pkg 디렉토리가 루트에도 있고, 여기 저기 많아서 헷갈렸는데, 여러 패키지에서 끌어다 사용할만한 기본적인 코드들은 /pkg에서 관리하고, 이 패키지를 import하는 별도의 도메인(istio-cni 등)마다 pkg 디렉토리를 추가로 만들어서 관리하는 방식을 사용하고 있는 것 같다.

## 7. References


[Istio - Install Istio with the Istio CNI plugin](https://istio.io/v1.10/docs/setup/additional-setup/cni/)

[Medium - 3 approaches to init containers in Istio CNI compared](https://www.redhat.com/architect/istio-CNI-plugin)

[CISCO - Enhancing Istio service mesh security with a CNI plugin](https://techblog.cisco.com/blog/istio-cni-plugin)
