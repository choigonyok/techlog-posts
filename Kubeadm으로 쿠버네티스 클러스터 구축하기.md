[ID: 1]
[Tags: KUBERNETES INFRA]
[Title: Kubeadm으로 쿠버네티스 클러스터 구축하기]
[WriteTime: 2023-06-24]
[ImageNames: b9d12434-7b7e-4545-a245-3956bea74a8b.png]
[Subtitle: 마스터/워커노드 구성을 위한 배시 스크립트 작성]


## Contents

1. Preamble
2. 클러스터 구축 스크립트에서 사용되는 리눅스 커맨드
3. 2>/dev/null & 리눅스 File Descriptor
4. 마스터/워커노드 공통 구성
5. 마스터노드 추가구성
6. 워커노드 추가구성
7. 마스터/워커 노드 공통 구성 스크립트 전문
8. 마스터노드 추가구성 스크립트 전문
9. References

## 1. Preamble


AWS는 쿠버네티스 클러스터를 쉽게 구축할 수 있도록  EKS라는 Managed K8s Service를 제공한다. 이 서비스는 실무에서도 많이 사용되고, 쿠버네티스를 학습하는 많은 개발자들이 애용하고있다.

내부적으로 쿠버네티스 클러스터가 어떤 과정을 거쳐 구축되지 알지 못한채 매니지드 서비스를 사용하는 것과 알고나서 사용하는 것은 큰 차이가 있다고 생각했다.

kubectl, kubeadm, kubelet, iptables 등의 쿠버네티스에서 주요하게 쓰이는 인터페이스, 컴포넌트들의 역할과 클러스터 구성과정, 리눅스 커맨드들에 대한 이해를 높히고 싶어서 kubeadm을 이용해 AtoZ 쿠버네티스 클러스터 구성을 경험해보기로 했다.

## 2. 클러스터 구축 스크립트에서 사용되는 리눅스 커맨드


* `cat` : 표준 입력을 받아서 파일로 저장하는 커맨드이다.

* `tee` : 표준 입력을 받아서 파일로 저장하는 커맨드이다.

`echo "HELLO" >> test.txt` 를 `tee` 커맨드를 사용해서 명령하면 

```bash
cat << EOF | tee test.txt
HELLO
EOF
```


로 표현할 수 있다.

* `modprobe` : 커널 모듈을 로드하는 커맨드이다.

* `sysctl` : 커널 파라미터 값을 확인하거나 변경하는 커맨드이다.

*  `swapoff` : 스왑 공간을 비활성화하는 커맨드이다. 컨테이너 간 격리를 위해 사용된다. 스왑 공간이 활성화 되어있으면 서로 다른 컨테이너끼리 데이터를 공유하는 일이 발생할 수 있다.

* `crontab` : cronjob을 관리하는 커맨드이다. cronjob은 정해진 시간마다 반복해서 수행되는 프로세스를 의미한다.

* `wget` :  웹에서 파일을 다운로드하는데 사용된다.

* `tar` : 파일을 압축시키거나 압축 해제시키는 커맨드이다.

* `systemctl` : 서비스를 관리하는 커맨드이다. 리눅스에서 서비스는 백그라운드에서 실행되는 소프트웨어를 의미한다.

* `install` : 파일을 특정 위치로 복사하고, 파일에 대한 권한을 변경하는 커맨드이다.

* `apt-mart` : 데비안 계열 리눅스에서 사용되는 패키지 관리자 apt의 마킹 작업을 수행한다.

## 3. 2>/dev/null & Linux File Descriptor


리눅스에는 `/dev` 경로에 `null` 파일이 있다. 가장 앞의 2는 STDERR를 의미한다. 이 2가 STDERR를 의미하게된 배경은 리눅스 `File Descriptor` 때문이다.

리눅스에는 `Everything is a file` 이란 말이 있다. 리눅스에서는 텍스트, 이미지, 동영상부터, 디렉토리, IPC를 위한 소켓, 표준입출력 등의 작업들도 모두 파일을 통해 처리한다. 

이 때 FD(File Descriptor)가 사용되는데 FD는 각 파일에 대한 포인터라고 볼 수 있다. 파일에 접근할 떄 이 FD를 통해 접근하게된다. 이 FD는 프로세스에 의해 추가/삭제 될 수 있지만 Default로 0번 FD는 STDIN, 1번 FD는 STDOUT, 2번 FD는 STDERR로 고정되어있다. 리눅스가 실행되면 기본적으로 이 0, 1, 2번 FD를 통해 STDIN, STDOUT, STDERR 파일이 Open된다. 익숙한 커맨드인 `echo` 커맨드를 사용해 특정 스트링을 표준 출력으로 출력한다면, 내부적으로는 1번 FD에 INPUT 스트링이 Copy되어서 STDOUT 파일에 작성되게 되는 것이다.

`2>/dev/null` 의 2는 이 FD 2번인 STDERR 파일을 의미하는 것이고, 이 파일로 들어오는 데이터를 /dev/null로 보내겠다는 의미이다. 즉, 에러가 생기면 무시하겠다는 표현인 것이다.

## 4. 마스터/워커 노드 공통 구성


마스터노드이던, 워커노드이던 인스턴스 내부에서 컨테이너를 구동시킬 수 있는 환경을 구성해야하는 것은 동일하기 때문에 구성이 대체로 유사하다. 

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```


`/etc/modules-load.d/k8s.conf` 파일을 생성해서 내용에 `overlay\\n br_netfilter` 이라는 스트링을 넣는 커맨드이다.

리눅스에서는 부팅이 될 때 `/etc/modules-load.d` 디렉토리에 존재하는 모든 파일들을 읽어서, 해당 파일들에 명시되어있는 모든 커널 모듈들을 로드하는 작업을 수행한다. 서버가 종료되었다가 다시 실행되더라도 이 파일을 통해 알아서 필요한 커널 모듈들이 로드되도록 설정해두는 것이다.

파일이 위치해야할 자리에 `<< EOF` 가 사용되면, 다음 `EOF`가 나오기 전까지를 읽어서 파일로 인식하겠다는 커맨드이다. EOF은 `End Of File`의 준말인데, 그냥 관습적으로 사용되는 것이고, 꼭 EOF가 아니더라도 상관없다.

개인적으로 Kubectl을 사용해서 파드를 배포할 때도 아래처럼 EOF를 즐겨 사용하곤 한다.

```bash
kubectl apply -f - << EOF
apiVersion: apps/v1
kind: Deployment
...
EOF
```


```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```


`/etc/modules-load.d` 경로의 파일을 통해 재부팅시에는 알아서 커널 모듈이 로드되지만, 지금 당장은 파일을 수정한다고 알아서 모듈이 로드되지는 않기 때문에 manually `modprobe` 커맨드를 사용해 커널 모듈을 로드해준다.

### overlay 모듈이란


`Overlay` 모듈은 `Overlay Network Driver`와 관련된 기능을 제공한다.

컨테이너 이미지를 

이 드라이버는 여러 컨테이너 간의 분산된 네트워크를 제공한다. 이 네트워크는 호스트 네트워크 위에 존재하고, 각 컨테이너에서 네트워크 트래픽이 발생할 때 올바른 곳으로 라우팅 될 수 있도록 돕는다.

Overlay 파일시스템은 `lowerdir`, `upperdir`, `merged` 세 가지 레이어로 구성된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/15D835C5-B904-414B-8FAD-42836FBB1DC6/AA650AE8-422D-4612-911C-FB271248BAC6_2/nBqQWCEtL9PxXzn3FZuzLYyhns969IbGPbkJf3EA0o8z/Image.png)

### br_netfilter 모듈이란


`br_netfilter` 모듈은 리눅스 네트워크 브릿지에 대한 방화벽 관련 기능을 제공하는 모듈이다.

네트워크 브릿지는 리눅스 내부에서 스위치 역할을 하는 컴포넌트이다. 일반적으로 스위치는 여러 엔드디바이스들이 연결되며 엔드디바이스들의 MAC주소를 테이블로 가지고있는 L2 디바이스이다.

이와 유사하게 리눅스 내부에서 여러 네트워크 인터페이스들과 연결되어서 하나의 게이트웨이 역할을 하는 컴포넌트라고 볼 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/15D835C5-B904-414B-8FAD-42836FBB1DC6/24C49510-9557-45D7-B918-8914A87320F4_2/U5M2cN0e8LrWW5TpyQvHwDh8LjhOEbg9hMm9dvWyajsz/Image.png)

여러 컨테이너들의 네트워크 인터페이스로 트래픽을 포워딩하거나, 컨테이너에서 출발한 트래픽을 인스턴스의 네트워크 인터페이스로 포워딩하는, 말그대로 네트워크 인터페이스 간의 브릿지 역할을 해준다.

이 네트워크 브릿지에 방화벽을 설정해서 패킷을 필터링할 수 있는데, 이런 기능을 제공하는 커널 모듈이 `br_netfilter`이다.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```


마찬가지로 `/etc/sysctl.d/k8s.conf` 파일을 생성하고, EOF 부터 EOF 까지의 스트링을 입력해준다.

`/etc/sysctl.d` 디렉토리는 리눅스가 부팅될 때 커널 매개변수를 설정하고 조정하는데 사용되는 디렉토리이다. 

`net.bridge.bridge-nf-call-iptables`는 IPv4에서 iptables 사용여부에 대한 커널 변수이다.

`net.bridge.bridge-nf-call-ip6tables`는 IPv6에서 iptables 사용여부에 대한 커널 변수이다.

`net.ipv4.ip_forward`는 패킷을 여러 다른 네트워크 인터페이스로 포워딩할 수 있게하는 커널 변수이다. 이 변수를 true로 설정하면 쿠버네티스의 파드 간의 패킷 포워딩이 가능해지게된다.

```bash
sudo sysctl --system
```


이 커맨드는 위에서 설정한 커널 변수들을 현재 시스템에도 즉시 적용시킨다.

```bash
sudo swapoff -a
```


현재 활성화되어있는 모든 스왑 공간을 비활성화시킨다.

```bash
(crontab -l 2>/dev/null; echo '@reboot /sbin/swapoff -a') | crontab - || true
```


```bash
wget https://github.com/containerd/containerd/releases/download/v1.7.3/containerd-1.7.3-linux-amd64.ar.gz
sudo tar Czxvf /usr/local containerd-1.7.3-linux-amd64.tar.gz
```


`wget` 커맨드로 해당 경로의 바이너리 파일을 로컬로 다운받아오고, `tar` 커맨드로 이 파일의 압축을 해제시켜서 `/usr/local` 경로에 저장한다.

이 파일은 `containerd` 인데, 컨테이너를 구현시키는 컨테이너 런타임 도구이다. containerd를 통해 컨테이너들이 서로 격리되고, 컨테이너 별 스토리지 관리, 컨테이너에서 사용되는 컨테이너 이미지 관리, 컨테이너 이미지 실행 등의 실질적으로 컨테이너가 구동되기위한 기능들을 제공한다.

```bash
sudo mv containerd.service /usr/lib/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```


압축해제된 containerd 서비스를 리눅스의 systemd가 서비스를 인식할 수 있는 `/usr/lib/systemd/system` 경로로 이동시켜준다.

그리고 systemd가 변경사항을 적용하도록 데몬을 리로드하고,  `enable` 서브커맨드를 통해  부팅시에도 containerd가 자동으로 실행되도록 설정하면서 지금 당장 바로도 실행시킨다.

```bash
wget https://github.com/opencontainers/runc/releases/download/v1.1.8/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
sudo mkdir -p /etc/containerd/
```


`wget`으로 이번엔 `runc` 를 다운로드 받아 755 권한을 부여하고 `/usr/local/sbin/runc` 경로로 이동시킨다.

runc는 실질적으로 Low Level에서 container 관련 기능을 수행하는 오픈소스 도구이다. containerd는 내부적으로 runc를 이용해 container를 관리한다.

그리고 `mkdir` 커맨드에 `-p` 옵션을  주어서 /etc/containerd/ 디렉토리를 생성한다.

```bash
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \\\\= false/SystemdCgroup \\\\= true/g' /etc/containerd/config.toml
```


이전에 `systemctl enable` 커맨드로 실행시킨 containerd 서비스의 CLI를 활용해서 containerd의 기본 설정 파일을 읽어와 `/etc/containerd/config.toml` 파일에 저장한다. 그리고 `sed` 커맨드를 사용해서 이 파일의 내용을 수정한다.

파일의 내용 중 `SystemdCgroup = false` 를 찾아서 `SystemdCgroup = true` 로 패치하는 커맨드이다.

Cgroup은 CPU, Mem, 스토리지 등의 리소스를 그룹화해서 각 프로세스 별로 최대치와 우선순위를 설정할 수 있는 리눅스 기능이다. 이 Cgroup을 통해 컨테이너간 리소스 격리를 수행할 수 있게된다.

```bash
sudo systemctl restart containerd
```


설정파일을 변경했으니 containerd를 재시작해서 `SystemdCgroup = true`가 적용되도록한다.

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```


apt 패키지 매니저를 사용해서 `apt-transport-https`, `ca-certification`, `curl` 세 가지 도구를 다운받는다.

그리고 다운받은 curl을 사용해서 다운로드 받은 도구들의 공개키를 다운로드 받아서 도구들의 신뢰성을 체크한다.

```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```


`/etc/apt/sources.list.d/kubernetes.list` 파일을 생성해서 `deb ... main` 스트링을 내용으로 입력한다.

이 내용은 `kubectl`, `kubeadm`, `kubelet` 패키지를 설치하기 위해 패지키가 위치한 패키지 저장소를 지정하는 것이고, apt 패키지 매니저는 `apt-get install` 로 패키지를 설치할 떄, `/etc/apt/sources.list.d` 에 위치한 파일들을 읽어서 해당 패키지 저장소 내에서 패키지들을 검색해서 설치하게된다.

```bash
sudo apt-get update && apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl containerd
```


쿠버네티스의 주요 컴포넌트이자 도구인 `kubelet`, `kubeadm`, `kubectl` 을 다운로드 받는다.

apt 패키지 매니저는 패키지의 새로운 버전이 나올 때마다 알아서 업데이트를 하는데, 이러다보면 업데이트된 버전과 버전 충돌이 생길 수 있기 때문에 다운로드한 패키지들이 자동으로 업데이트되지 않도록 `hold`를 마킹해준다.

## 5. 마스터노드 추가 구성


마스터노드는 kubeadm을 통해 쿠버네티스 마스터노드를 구성하기 위한 스크립트가 추가적으로 필요하다.

```bash
IPADDR=$(curl ifconfig.me && echo "")
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
```


우선 세 변수를 초기화한다.

`curl ifconfig.me`는 현재 호스트의 IP주소를 출력한다. 이 값을 `IPADDR`에 저장한다.

`hostname`은 현재 호스트의 이름을 출력해주는데, -s 옵션으로 부가적인 정보 없이 정확히 이름만을 출력하게된다. 이 값을 `NODENAME`에 저장한다.

`POD_CIDR`에는 노드 내부에 파드가 배포될 떄 사용될 Private IP 주소의 볼록이 저장된다. 사설 IP 대역이기 때문에 임의로 IP와 서브넷을 지정하면 된다.

```bash
touch text.txt
```


`touch` 커맨드로 `text.txt` 라는 이름의 빈 파일을 생성한다.

```bash
sudo kubeadm init --control-plane-endpoint=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  -pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap > text.txt
```


kubeadm CLI로 위에서 초기화한 변수들을 통해 쿠버네티스 클러스터를 구성한다. 마지막 `> text.txt` 를 통해 kubeadm init의 실행 로그가 text.txt에 저장된다.

```bash
sudo mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


호스트의 홈디렉토리에 `.kube` 디렉토리를 생성한다. 그리고 `/etc/kubernetes/admin.conf` 파일이 `.kube` 디렉토리에 `config`라는 이름으로 복사된다. kubectl CLI은 이 .kube 디렉토리의 파일들을 보고 kubectl을 사용하는 관리자가 여러 쿠버네티스 컨텍스트를 변경해가며 다중 쿠버네티스 클러스터를 관리할 수 있도록 지원한다.

`id -u`와 `id -g` 커맨드로 UID, GID를 지정해서 이 파일을 읽을 권한을 부여해준다.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
kubectl apply -f https://raw.githubusercontent.com/techiescamp/kubeadm-scripts/main/manifests/metrics-server.yaml
```


마지막으로는 쿠버네티스 네트워킹을 위해 `calico` 네트워크 플러그인을 클러스터에 배포해주고, `kubectl top nodes` 등의 노드 메트릭을 관리하는 메트릭 서버를 클러스터에 배포해준다.

이렇게 하면 쿠버네티스 클러스터/컨트롤 플레인 생성 및 마스터 노드의 구성이 완료된다.

## 6. 워커노드 추가 구성


클러스터는 마스터노드에서 구성했다. 워커노드는 이 마스터노드로 `Join`되는 추가 과정이 필요하다.

앞선 마스터노드 구성에서 `kubeadm init`의 로그를 `text.txt` 파일에 저장했었다. 이 로그에는 워커노드를 Join 시키기 위한 커맨드가 포함되어있기 떄문이다.

로그 중 `kubeadm join ~ --token ~` 형식의 커맨드를 복사해서 공통 구성이 완료된 워커노드에서 실행하면 정상적으로 워커노드가 클러스터에 조인된다.

마스터노드에서 `kubectl get nodes`를 실행해보면 노드가 추가된 것을 확인할 수 있다.

##  7. 마스터/워커 노드 공통 구성 스크립트 전문


```other
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
sudo swapoff -a
(crontab -l 2>/dev/null; echo '@reboot /sbin/swapoff -a') | crontab - || true
wget https://github.com/containerd/containerd/releases/download/v1.7.3/containerd-1.7.3-linux-amd64.ar.gz
sudo tar Czxvf /usr/local containerd-1.7.3-linux-amd64.tar.gz
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mv containerd.service /usr/lib/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
wget https://github.com/opencontainers/runc/releases/download/v1.1.8/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
sudo mkdir -p /etc/containerd/
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \\\\= false/SystemdCgroup \\\\= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl containerd
```


## 8. 마스터노드 추가 구성 스크립트 전문


```bash
IPADDR=$(curl ifconfig.me && echo "")
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
touch text.txt
sudo kubeadm init --control-plane-endpoint=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  -pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap > text.txt
sudo mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
kubectl apply -f https://raw.githubusercontent.com/techiescamp/kubeadm-scripts/main/manifests/metrics-server.yaml
```


## 9. References


[How to Install Containerd on Ubuntu 22.04 / Ubuntu 20.04](https://www.itzgeek.com/how-tos/linux/ubuntu-how-tos/install-containerd-on-ubuntu-22-04.html)

[How To Setup Kubernetes Cluster Using Kubeadm](https://devopscube.com/setup-kubernetes-cluster-kubeadm/)

[github.com/saiyam1814/kubeadm + containerd 1.23](https://gist.github.com/saiyam1814/ba56e5fb09d712501d5a4b51e8ad85a5#file-kubeadm-containerd-1-23-L33)

[Docker Official Docs-Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/#install-from-a-package)

[K8s Official Docs-kubeadm 설치하기](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

[카카오 기술블로그-[컨테이너 인터널 #2] 컨테이너 파일시스템](https://tech.kakaoenterprise.com/171)
