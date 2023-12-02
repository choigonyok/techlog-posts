[Tags: KUBERNETES INFRA PROJECTS]
[Title: HA ETCD 외부 클러스터 구축하기]
[WriteTime: 2023/10/24]
[ImageNames: ]

## Contents

1. Preamble
2. what is etcd?
3. Stacked etcd VS External etcd
4. HAProxy
5. Implement
6. Summary
7. References

## Preamble


미국의 CISA와 NSA에서 쿠버네티스를 사용하는 미국의 국가기관들의 클러스터 보안을 위해 작성한 Kubernetes Hardening Guide는 etcd를 control plane에서 분리해 별도의 외부 네트워크에 ercd를 구성하기를 권장하고있다.

MSA 프로젝트의 로컬 개발환경으로 etcd를 외부에 구축해서 보안을 강화하고, 추가적으로 무중단 서비스 운영을 위해 3개의 etcd를 구성하여 고가용성을 보장하는 etcd 클러스터를 구축하기로 했다.

## what is etcd?


etcd(엣시디)는 오픈소스 분산 키-값 저장소이다. redis와 유사하다. 

etcd는 원래부터 쿠버네티스에 속해있던 것이 아니라 쿠버네티스 *이전부터* 오픈소스로 개발되어있던 데이터베이스이다. 쿠버네티스에서 공식 백엔드 데이터베이스로 etcd를 채택하면서 etcd가 유명세를 타게 되었다. 때문에 쿠버네티스가 아니더라도 etcd를 분산 키-값 저장소로써 다양한 환경에 적용해서 별도로 사용할 수 있다.

쿠버네티스에서는 etcd를 쿠버네티스 클러스터에 관한 정보들을 저장하는 백엔드 데이터베이스로 사용한다. 쿠버네티스의 etcd에는 current stats, desired state, secret 등 클러스터가 정상적으로 작동하는데 필요한 여러 중요한 데이터들이 저장된다.

etcd가 작동하지 않으면 현재 클러스터의 상태에 대해 확인하거나, Pod를 배포하는 것부터 시작해서 그냥 전반적으로 클러스터가 정상적인 프로세스를 수행할 수 없기 때문에 RAFT 알고리즘을 이용해 3개 이상의 홀수 etcd를 운영해서 고가용성을 보장하는 것이 좋다.

또한 etcd가 공격당하면  공격자가 credential이 담겨있는 secret 리소스의 데이터를 모두 탈취할 수 있게 되며, etcd에 write 권한이 있으면 전체 쿠버네티스 클러스터에 접근할 수 있는 권한이 생기는 것과 마찬가지이기 때문에, 기존 쿠버네티스 클러스터에서 분리해서 네트워크를 별도로 두고 방화벽을 통해 보안을 강화하는 것이 좋다.

## Stacked etcd VS External etcd


etcd의 구성을 default에서 변경한다고 하면, 일반적으로 stacked etcd 또는 external etcd로 구성하게 된다. default로 etcd는 아래 사진처럼 쿠버네티스 클러스터 내부의 control plane에 속해있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/D7652709-91BE-42AC-99A0-46B6CBAA962A_2/WcbCHfGIn9yu1vMqecA7rJf2lMxv8k3kUouR8ZfeiRQz/Image.png)

### Stacked etcd


Stacked etcd는 아래 사진처럼 etcd를 외부로 분리하지 않고, control plane 자체를 중첩적으로(stacked) 쌓아서 고가용성을 높이는 방법이다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/25A36174-6551-4443-A278-5AB8B5361A47_2/KxJlaNyz9PuR6SJoWV1TTZRlmJ7Mq14xQjYdPZ2dHG0z/Image.png)

이렇게 되면 외부에 etcd를 구축할 필요 없이 여러 etcd를 운영하면서 고가용성을 높일 수 있고, etcd 뿐만 아니라 api server 등의 다른 control plane도 함께 고가용성 운영이 가능해지기 때문에, 더 안정적인 클러스터 운영이 가능하다.

단점은 control plane 전체가 여러 개 생성되는 형태이기 때문에, 원하지 않는 컴포넌트도 etcd와 함께 추가적으로 생성되게 된다. 즉 **커스터마이징이 안된다**고 요약할 수 있다.

또 etcd를 외부로 분리하지 않는 형태이기 때문에 이후 설명할 External etcd에 비해 상대적으로 **보안에 취약하다**고 볼 수 있다.

### External etcd


External etcd는 아래 사진처럼 etcd를 control plane 외부로(External) etcd를 분리하는 방법이다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/5546DA37-40F1-4681-B174-54326392C366_2/GSRZHhBLUxFdStiyrB2RELtyKNJxDTwMlGOCrte5n2Ez/Image.png)

이렇게 되면 etcd가 분리되어있기 때문에, etcd와 control plane의 네트워크를 분리할 수 있고, etcd와 control plane 사이에 방화벽을 구성해서 보안적으로 더 안전하게 etcd를 보호할 수 있다. 또 control plane과 etcd의 확장이 별도로 가능해져서 더 서비스에 적합한 방식으로 클러스터를 구성할 수 있다.

단점은 상대적으로 **구성이 더 복잡**하고, control plane과 etcd의 상태를 **별도로 관리**해야한다는 불편함이 있다.

HA external etcd cluster를 구축하면서 과정없이 인터넷의 모든 관련 글을 다 찾아보았는데, 대부분 Stacked etcd 방식으로 많이들 구현하는 것을 확인할 수 있었다.

나는 대세와는 반대로 external etcd를 구축하기로 결정했는데,


1. 보안적으로 더 안전해서
2. etcd를 분리할 줄 아는데 안하는 것과, 몰라서 안하는 것은 차이가 크다고 생각해서

external 방식을 채택하기로 했다.

## HAProxy


내가 선택한 External etcd 방식과 Stacked etcd 방식 모두, 다수의 API 서버 또는 다수의 etcd로 트래픽을 분산하기 위한 로드밸런서가 필요하다.

나는 **etcd 클러스터 로드밸런서**로 **HAProxy**를 선택했다. HAProxy는 오픈소스 프록시 / 로드밸런서이다. Kong, Nginx 등 많은 로드밸런서 도구들이 있는데 내가 HAProxy를 선택한 이유는 아래와 같다.


1. 헬스체크
2. 고급 로드밸런싱
3. 고가용성
4. 웹 대시보드

### 헬스체크


HAProxy는 헬스체크를 지원한다. 헬스체크를 통해 alive하지 않은 백엔드에는 트래픽을 전달하지 않는다.

### 고급 로드밸런싱


HAProxy는 라운드 로빈 외에도 다양한 고급 로드밸런싱 알고리즘을 지원한다.

### 고가용성


etcd가 중단되지 않도록 고가용성을 높이기 위해 RAFT 알고리즘을 통해 3 이상의 홀수 개로 etcd들을 운영하는데, 만약 이 etcd들로 로드밸런싱해주는 리버스 프록시 또는 로드밸런서에 문제가 생기면 etcd가 정상이어도 API 서버가 etcd에 접근하지 못하게 되어 결국 서비스 중단이 생기게된다.

결국은 etcd 뿐만 아니라 리버스 프록시 / 로드밸런서도 고가용성을 보장해줘야 etcd 클러스터의 고가용성을 온전히 보장할 수 있는데, HAProxy는 이를 지원한다. 이름부터도 HA(High Availability)Proxy인 것을 보면 알 수 있다.

### 웹 대시보드


HAProxy는 웹 대시보드를 통해 HAProxy와 연결된 백엔드(etcd)들의 상태를 쉽게 확인할 수 있다. 이를 통해 etcd 클러스터가 control plane과 별도로 관리된다는 점에서 오는 불편함을 일부 해소할 수 있다.

## Implement


구현은 총 4단계로 나뉜다.


1. etcd 도커파일 config 파일 작성
2. haproxy 도커파일 및 config 파일 작성
3. docker-compose 파일 작성
4. Test
5. KIND configuration 변경
6. Test

큰 그림은, docker-compose를 통해 하나의 네트워크로 묶인 haproxy + 3 etcd 컨테이너가 포트포워딩된 HAProxy의 2379 포트를 통해 로컬과 연결될 것이다.

### 1. etcd 도커파일 config 파일 작성


```dockerfile
FROM ubuntu:22.04

RUN apt update && apt install wget -y && apt install systemctl -y

WORKDIR /usr/local

RUN wget https://github.com/etcd-io/etcd/releases/download/v3.4.26/etcd-v3.4.26-linux-amd64.tar.gz

RUN tar xvf etcd-v3.4.26-linux-amd64.tar.gz

RUN mv /usr/local/etcd-v3.4.26-linux-amd64/etcd /usr/local/bin
RUN mv /usr/local/etcd-v3.4.26-linux-amd64/etcdctl /usr/local/bin

RUN mkdir /var/lib/etcd

RUN groupadd --system etcd

RUN useradd -s /sbin/nologin --system -g etcd etcd

RUN chown -R etcd:etcd /var/lib/etcd/

COPY ./etcd.service /etc/systemd/system/etcd.service

RUN systemctl stop etcd.service

USER etcd

ENTRYPOINT [ \"systemctl\", \"start\", \"etcd.service\" ]
```


etcd 3개를 구축한다고 했을 때, etcd용 도커파일 3개와 config 파일 3개가 필요하다. 도커파일은 etcd가 모두 같은 도커파일을 사용하지만, config file은 각 etcd마다 내용을 약간 수정해야한다.

도커파일에 큰 내용은 없고 우분투 베이스 이미지에 etcd를 설치하고 실행시키는 이미지이다.

아래는 config 파일인 etcd.service의 내이다.

```dockerfile
[Unit]

Description=etcd service

Documentation=https://github.com/etcd-io/etcd

After=network.target

After=network-online.target

Wants=network-online.target

[Service]

User=etcd

Type=notify

Environment=ETCD_DATA_DIR=/var/lib/etcd

ExecStart=/usr/local/bin/etcd --name node3 --initial-advertise-peer-urls http://111.0.0.4:2380 
  --listen-peer-urls http://111.0.0.4:2380 
  --listen-client-urls http://111.0.0.4:2379 
  --advertise-client-urls http://111.0.0.4:2379 
  --initial-cluster-token etcd-cluster-1 
  --initial-cluster node1=http://111.0.0.2:2380,node2=http://111.0.0.3:2380,node3=http://111.0.0.4:2380 
  --initial-cluster-state new

Restart=always

RestartSec=10s

LimitNOFILE=40000

[Install]

WantedBy=multi-user.target
```


뒤에서 설명하겠지만, 나는 각 etcd의 PrivateIP를 111.0.0.2, 111.0.0.3, 111.0.0.4로 할당했다. 이 글을 따라서 구성할 때는 이후 docker-compose.yml에서 설정하는 사설 IP에 맞게 내용을 수정해야한다.

2379 포트는 etcd가 외부 트래픽을 받을 때 사용하는 포트이고, 2380 포트는 etcd끼리 통신하며 데이터 정합성을 유지하고 RAFT 알고리즘을 수행하기 위한 포트이다.

```dockerfile
ExecStart=/usr/local/bin/etcd --name node3 --initial-advertise-peer-urls http://111.0.0.4:2380 
```


line 21, 에서 --name 옵션으로 설정되어있는 node3은 각 노드에 맞게 node1, node2로 변경해주어야한다.

또,

```dockerfile
--initial-cluster node1=http://111.0.0.2:2380,node2=http://111.0.0.3:2380,node3=http://111.0.0.4:2380
```


line 26, 의 이 부분을 제외한 모든 111.0.0.4 HOST는 해당 노드에 맞는 IP로 변경해주어야한다.

### 2. haproxy 도커파일 및 config 파일 작성


```dockerfile
FROM ubuntu:22.04

RUN apt update && apt install haproxy -y

RUN apt install systemctl -y

COPY ./haproxy.cfg /etc/haproxy/haproxy.cfg

RUN systemctl stop haproxy.service

ENTRYPOINT [ \"systemctl\", \"start\", \"haproxy.service\" ]
```


도커파일에는 특별한 부분은 없고, etcd 도커파일과 마찬가지로 우분투 베이스 이미지에 HAProxy를 설치하는 내용이다.

```dockerfile
global
        log     127.0.0.1 local2

defaults
        log global
        mode tcp
        retries 2
        timeout client 30m
        timeout connect 4s
        timeout server 30m
        timeout check 5s

listen stats
        bind  *:9000
        mode  http
        stats enable
        stats uri /haproxy_stats
        stats auth admin:admin

listen  etcd
        bind  *:2379
        mode  tcp
        balance roundrobin
        option  tcp-check
        server  node1  111.0.0.2:2379  check
        server  node2  111.0.0.3:2379  check
        server  node3  111.0.0.4:2379  check

```


stats은 로컬의 9000번 포트 + /haproxy_stats 경로에서 HAProxy 웹 대시보드를 확인할 수 있게 해준다.

etcd의 이름과 IP를 명시해주고, 헬스체크 옵션 및 라운드 로빈으로 로드밸런싱하도록 설정한다.

### 3. docker-compose 파일 작성 


```dockerfile
version: \'3\'
networks:
  etcd:
    ipam:
      config:
        - subnet: 111.0.0.0/24
services:
  node1:
    container_name: node1
    build:
      dockerfile: Dockerfile
      context: ./etcd1
    networks:
      etcd:
        ipv4_address: 111.0.0.2

  node2:
      container_name: node2
      build:
        dockerfile: Dockerfile
        context: ./etcd2
      networks:
        etcd:
          ipv4_address: 111.0.0.3
        
    node3:
      container_name: node3  
      build:
        dockerfile: Dockerfile
        context: ./etcd3
      networks:
        etcd:
          ipv4_address: 111.0.0.4

  haproxy:
      container_name: haproxy
      build:
        dockerfile: Dockerfile
        context: ./haproxy
      depends_on:
        - node1
        - node2
        - node3
      ports:
        - 2379:2379 # endpoint for K8S apiServer
        - 9000:9000 # for Web UI Dashboard
      networks:
        - etcd
  ```


**networks** 필드를 통해 etcd 라는 도커 네트워크를 생성해주고, 서브넷을 지정한다.

각 서비스에는 **build** 필드를 통한 도커파일의 위치와 이름을 지정해주고, etcd 네트워크에 속하도록 명시하면서 **ipv4_address** 필드로 네트워크 내부 사설 IP 주소를 임의로 할당해준다.

HAProxy 서비스는 3개의 etcd 컨테이너가 모두 생성된 후에 HAProxy가 생성될 수 있도록 **depends_on** 필드를 통해 priority를 지정해준다.

HAProxy는 다른 네트워크(외부)인 쿠버네티스 클러스터의 접근을 가능하게 해야하기 떄문에, local로의 2379, 9000 포트 포워딩을 구성한다. 2379는 etcd로의 라우팅을 위해서, 9000은 웹 대시보드 사용을 위해 구성했다.

### 4. Test


이제 docker-compose.yml 파일 레벨에서

```dockerfile
docker-compose up --build
```


커맨드를 입력하면,

```dockerfile
docker ps
```


docker ps 커맨드로 실행 중인 컨테이너들을 확인했을 떄, 아래와 같이 정상적으로 출력되게된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/1A5C6685-0693-4384-AF9F-FFF9009B2B5A_2/ZAW820QG1VZUI06QjnEb9KYMeZ837UOFB3exD04A7EEz/Image.png)
> etcd 컨테이너 내부로 접속해서 curl을 이용해 etcd 데이터에 값을 저장하고 조회하는 테스트를 진행했다.


```dockerfile
docker exec -it CONTAINER_ID /bin/bash
```


위 커맨드로 etcd 컨테이너 내부로 접속하고,

```dockerfile
curl -X POST -H \"Content-Type: application/json\" -d \'{\"key\": \"dGVzdGtleQ==\", \"value\": \"dGVzdHZhbHVl\"}\' [http://localhost:2379/v3/kv/put](http://localhost:2379/v3/kv/put)
```


위 명령어로 etcd에 키-값을 POST요청으로 HAProxy에 보냈다. **키-값은 base64로 인코딩한 값을 넣어야한다.** 테스트를 위해 key로는 \"testkey\", value로는 \"testvalue\"를 base64로 인코딩한 값인,  \"dGVzdGtleQ==\" 와 \"dGVzdHZhbHVl\"를 지정했다.

이후 

```dockerfile
curl [http://localhost:2379/v3/kv/range](http://localhost:2379/v3/kv/range) -d \'{\"key\": \"dGVzdGtleQ==\"}\'
```


위 명령어로 key인 \"dGVzdGtleQ==\" 에 대한 값을 GET했더니, 조금 구조가 복잡하긴 하지만 아래 사진과 같이 정상적으로 아까 POST요청으로 보낸 value인 \"dGVzdHZhbHVl\"가 출력되는 것을 확인할 수 있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/F2D2F531-EDF5-42E5-9F86-236A2E6F16D1_2/fEWsG96ULx9ipWkNPZvDg4qOMHKUkELiWdlutSboBTAz/Image.png)

docker-compose.yml 파일에서 포트 포워딩은 HAProxy에 했기 때문에, localhost:2379로의 요청은 HAProxy로 전달되어서 라운드로빈을 통해 3개의 etcd 중 하나에 저장되었다.

etcd들은 자체적으로 2380 포트를 통해 데이터를 일치시켰고, GET 요청 역시 HAProxy를 통해 etcd 중 하나로 전달되고, 저장되어있던 key에 대한 value가 응답되었다.

### 5. KIND configuration 변경


이제 KIND cluster의 API server와 external etcd cluster의 haproxy를 연결해주어야하고,

```dockerfile
kind create cluster
```


를 통해 자동으로 생성되던 etcd의 생성을 막아야한다. 이를 위해 config 파일을 하나 생성했다.

```dockerfile
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    etcd:
      external:
        endpoints: 
        - \"http://host.docker.internal:2379\"
        caFile: \"\"
        certFile: \"\"
        keyFile: \"\"
```


KIND는 kubeadm의 설정을 변경할 수 있는 기능을 제공한다. kubeadmConfigPatches 필드를 활용해서 etcd의 엔드포인트를 127.0.0.1:2379 가 아닌 http://host.docker.internal:2379로 변경해주면 된다.

원래 컨테이너는 호스트에 접근할 수 없다. 그게 컨테이너의 중요한 기능 중 하나이다. 컨테이너에서 localhost로 접근하면 호스트인 로컬의 localhost로 접근되는 것이 아니라, 해당 컨테이너 자기 자신에게 접근된다.

컨테이너에서 호스트에 접근하려면 호스트의 PublicIP로 접근해야하고, 실험해본 결과 그렇게 해도 정상적으로 동작은 한다. 다만 나의 경우에 아래와 같은 세 가지 문제점이 있었다.


1. 팀 프로젝트

팀 프로젝트로 진행하고 있었기 때문에, 내 로컬의 IP를 하드코딩해서 공유저장소에 올리면 다른 팀원이 etcd 클러스터를 실행할 때마다 자신의 IP로 코드를 변경해야한다.


2.  Not StaticIP

PublicIP는 PC가 재부팅될 때마다 변경되게 된다. 나 역시도 매번 컴퓨터를 재시동할 때마다 새롭게 할당된 IP로 코드를 수정해야한다.


3.  공유저장소 보안

이 코드는 팀원과의 협업을 위해 공유저장소에 올라가야하는데, 나의 로컬 IP가 공유저장소에 올라가는 것은 보안적으로 좋지 않다.

이러한 이유로 다른 방법을 찾아야만 했다. 그러다가 host.docker.internal을 찾게되었다.

**host.docker.internal**를 통해서 컨테이너는 자신의 호스트에 접근할 수 있게 된다. 로컬 개발환경에서 앞으로도 아주 유용하게 쓰이게 될 것 같다.

### 6. Test


마지막으로 전체 테스트를 진행했다. 클러스터를 처음부터 쭉 생성하고 문제가 없는지 확인했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/84602B66-7332-46B8-BE38-EE6FCCA8368A_2/JneWa6ZD5KO9tufX4na5ShT81CmWWixOuBomJuyEANwz/Image.png)

위 사진처럼 etcd pod가 생성되지 않은 것을 확인할 수 있었고, 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/1B7FB9FD-B484-4DCB-AD59-E404041CB638_2/sHBKvAvQfxVN2rHp5FaaU6KypMp27fAsmUa7rKwJHyQz/Image.png)

위 사진처럼 전체 클러스터가 정상 작동하는 것 역시 확인할 수 있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/2A82FD00-EF9F-4CB0-96D7-FF44ADD21B1D_2/zsaPI2Fen6FOZk506zT6ybQr8IiJdhYjC03heXBJGlQz/Image.png)

위 사진처럼 kiali를 통해 확인했을 때 모든 서비스들이 정상적으로 연결되고 작동중인 것을 확인할 수 있었고,

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6D4E6B9E-F3DE-4C4A-89E9-2DF30E7540E3/5AE99F9F-1F68-45B0-B826-FA9135804547_2/3aU899yOwsTntY7kP26kMOHU4Qm1x4Z9oLyuWzaGQykz/Image.png)

브라우저에서 http://localhost:9000/haproxy_stats 로 접속하면 3개의 etcd 노드들이 그린 라이트로 정상 작동 중이며, 각각 30, 29, 29개씩의 요청을 받아 라운드 로빈도 정상 동작하는 것을 확인할 수 있었다.

## Summary


정말 과장없이 구글 상의 모든 external etcd cluster와 관련된 글을 다 찾아봤지만, 나와 같이


1. KIND 로컬 쿠버네티스 클러스터에서
2. control plane과 etcd를 분리해서
3. HAProxy로 etcd들을 로드밸런싱하면서
4. docker compose로 네트워크를 구현하는

사례는 단 한 가지도 찾을 수 없었다. 그 때문에 정말 많은 고생을 했지만, 결국 완벽하게 완성시키고 나니까 마치 어플리케이션 하나를 개발한 것과 같이 무에서 유를 창조한 것 같은 성취감을 느낄 수 있었다.

이 구현을 해나가면서 네트워크와 HAProxy, etcd에 대해 조금 더 알아갈 수 있어서 좋았다.

## References


레퍼런스가 100개 가까이 될 것 같지만 다 기입할 수 없어서 일부만 가져왔다.

[KIND Official Docs](https://kind.sigs.k8s.io/docs/user/configuration/#networking)

[[Docker]HAProxy를 이용한 로드 밸런싱](https://blog.neonkid.xyz/231)

[고가용성 쿠버네티스를 구축해 봅시다!](https://tech.cloudmt.co.kr/2022/06/27/k8s-highly-available-clusters/)

[Creating an etcd cluster and replacing the default etcd in a Kubernetes cluster](https://blog.devops.dev/creating-an-etcd-cluster-and-replacing-the-default-etcd-in-a-kubernetes-cluster-4004522c92f4)

[How to Setup etcd Cluster](https://facsiaginsa.com/etcd/how-to-setup-etcd-cluster)

[Docker, HAProxy 이용해 서비스 무중단 배포하기](https://velog.io/@suhongkim98/Docker-HAProxy-%EC%9D%B4%EC%9A%A9%ED%95%B4-%EB%A1%9C%EB%93%9C%EB%B0%B8%EB%9F%B0%EC%8B%B1-%ED%95%98%EA%B8%B0)

[Clustering Guide](https://etcd.io/docs/v3.4/op-guide/clustering/)