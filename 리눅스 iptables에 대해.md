[ID: 59]
[Tags: Istio network]
[Title: 리눅스 iptables에 대해]
[WriteTime: 2024-01-28]
[ImageNames: 5cff130e-a27f-4081-b0fc-1156fbff73d6.png]

## Contents

1. Preamble
2. iptables란
3. iptables Architecture
4. iptables Packet Flow 
5. iptables 커맨드 예시
6. iptables rule 우선순위
7. References

## 1. Preamble


Istio Issue를 해결하기위해 `istio-cni` 코드베이스를 분석하며 왜 istio-cni에서는 initContainer의 아웃바운드 트래픽이 `ztunnel`의 트래픽 인터셉트에서 exclude되지 않는 것인가를 파악하고있었다. 

그러다가 ztunnel과 istio-proxy 모두 iptables 구성을 통해 트래픽을 인터셉트한다는 것을 알게되었고, iptables에 대해 알아야 이 이슈를 해결할 수 있겠다는 생각에 학습을 시작하게되었다.

## 2. iptables란


리눅스에서는 방화벽을 관리하기 위해 `netfilter`라는 커널 모듈을 사용한다. iptables는 이 netfilter를 사용하기위한 인터페이스이다. `방화벽`은 어떤 패킷이 서버로 들어오거나/나가는 것을 허용/거부하는 필터를 말한다.

중요한 역할을 하는만큼 iptables를 구성하기 위해서는 루트 권한이 필요하고, `NAT`, `로깅`, `포워딩`, `리다이렉팅` 등 여러 방면으로 활용될 수 있다.

## 3. iptables Architecture


iptables에는 `chain`, `table`, `target`, `rule`등 여러 구성요소들이 있다.

### Chain


체인은 `rule`을 추가함으로써 패킷에 대해 특정 작업을 수행하는 컴포넌트이다. iptables에는 다섯개의 체인이 있다.


- PREROUTING

PREROUTING 체인은 네트워크 인터페이스에 패킷이 도착하자마자 수행되는 체인이다. 패킷을 조작하거나 패킷을 없애버리는 등의 작업이 가능하다.


- INPUT

INPUT 체인은 인바운드 트래픽과 관련된 체인이다. `rule`을 포함하고있고. 이 룰을 통해 서버로 들어오는 악성 패킷을 차단하는 등의 필터링을 수행할 수 있다.


- FORWARD

FORWARD 체인은 패킷을 포워딩하는 체인이다. 이 체인에도 여러 룰들이 적용될 수 있고, 이 룰을 통해 어디로 라우팅/포워딩 할지를 결정하게된다.


- OUTPUT

OUTPUT 체인은 아웃바운드 트래픽과 관련된 체인이다. 룰을 통해 패킷이 외부로 나가게 할 수도 있고, 막을 수도 있다.


- POSTROUTING

POSTROUTING 체인은 패킷이 서버를 떠나기 직전 특정 task를 수행하는 체인이다. 

### Table


테이블은 패킷과 관련해 특정 역할을 담당한다. 하나의 테이블 안에는 여러 개의 체인이 default로 포함되어있고, 사용자 정의 체인을 추가하면 가장 마지막 체인으로 하나씩 추가된다. 체인의 순서를 변경할 수도 있다.


- FILTER

FILTER 테이블은 iptables의 default 테이블이다. 패킷의 필터링을 담당하는 테이블이다.


- NAT

NAT 테이블은 Network Address Translation의 준말로, 새로운 커넥션을 생성하는 테이블이다.


- MANGLE

MANGLE 테이블은 패킷을 조작하는 등 특수한 경우에 사용되는 테이블이다.


- RAW

RAW 테이블은 raw 패킷과 관련있는 테이블이다. 커넥션의 state를 추적하는데 주로 사용된다.


- SECURITY

SECURITY 테이블은 FILTER 테이블을 패킷이 거친 이후에, 보안을 담당하는 테이블이다. 리눅스에서 보안을 담당하는 `SELinux`와 관련이 많다.

### Target


타겟은 패킷에 대한 작업을 의미한다. `Terminating` 타겟과 `Non-Terminating` 타겟으로 나뉜다.


- Terminating 타겟

특정 패킷이 매칭이 되자마자 즉시 실행되는 작업을 의미한다. 종류로는 `ACCEPT`, `DROP`, `REJECT` 타겟이 있고, 각각 패킷을 허용하거나 패킷을 버리거나 패킷을 돌려보낸다.


- Non-Terminating 타겟

패킷 매칭이 되어도 또 매칭될 때까지 내버려두는 작업을 의미한다. 대표적으로 `LOG` 타겟이 있다. 로깅을 위해 패킷을 매칭하더라도 원래 목적지로 패킷이 흘러가야하기 때문에 이러한 용도로 Non-Terminating 타겟이 사용될 수 있다.

### Rule


룰은 위에서 설명한 Chain, Table, Target을 활용해서 어떤 패킷이 어떤 테이블에서 어떤 체인들을 거치게할 것인지에 대한 규칙을 의미한다.

iptables에서는 iptables 커맨드를 통해 룰들을 정의하고 관리할 수 있다.

## 4. iptables Packet Flow


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B9CA67CA-CDD1-4ADF-8929-61EE6D8E6B9F/69A97DE6-62AF-4C71-9920-A47EB353F260_2/vuGROe1lG7oPE47Iu0W3HYIycmQoMWiOR7padkhJTmcz/Image.png)

## 5. iptables 커맨드 예시


```yaml
iptables -L (-t TABLE)
```


-L 플래그는 체인 리스트를 확인하는 플래그이다. -t는 테이블을 지정하는 플래그인데, 테이블이 지정되어있지 않으면 default로 FILTER 테이블의 체인 리스트가 출력된다.

```yaml
iptables -A CHAIN -j ACCEPT
```


-A 플래그는 Append 플래그로, 해당 체인에 룰을 추가하겠다는 의미이다.

-j 플래그는 패킷에 대한 타겟을 지정하는 플래그이다. `ACCEPT` 말고도 `DROP`, `REJECT`, `LOG`, `RETURN` 등이 가능하다.

```yaml
iptables -A INPUT -p TCP --dport 15001 -j REJECT
```


-p 플래그는 프로토콜을 지정하는 것을 의미하고, --dport는 destination port, 목적지 포트를 의미한다.

이 룰은 INPUT 체인에서, 즉 인바운드 트래픽이 15001 포트로 들어오고 TCP 프로토콜일 경우에 해당 트래픽을 REJECT한다는 규칙을 설정하는 것이다.

```yaml
iptables -A OUTPUT -p TCP -s 10.24.1.1 -j DROP
```


--dport가 목적지 포트였던 것처럼 --sport는 출발지 포트를 의미하고, -s 플래그는 출발지(source) IP주소을 의미하고 -d는 목적지 IP주소를 의미한다.

이 룰의 경우에는 10.24.1.1에서 나가는 패킷이 OUTPUT에 매치되면 패킷을 DROP하라는 규칙이다.

## 6. iptables rule 우선순위


iptables에서 룰은 먼저 적용된 것이 더 높은 우선순위를 갖는다.

```yaml
iptables -A INPUT --jump ACCEPT --protocol all   --source 127.0.0.1
iptables -A INPUT --jump ACCEPT --protocol tcp   --dport 22
iptables -A INPUT --jump REJECT --protocol all
```


이러한 순서로 iptables가 구성되었다고 가정하자.

우선 127.0.0.1, 호스트에서 출발해서 호스트로 들어오는 모든 트래픽은 허용된다.

외부에서 들어오는 패킷중에서는 22번 포트로 들어오는 TCP 패킷만 허용된다.

그리고 마지막으로 모든 패킷을 REJECT한다. 그러나 먼저 적용된 두 규칙이 우선순위가 높기 때문에, 결론적으로 localhost 트래픽과 22번 포트로 들어오는 TCP 트래픽만 허용하고 나머지 모든 트래픽을 REJECT하는 iptables이 구성되게 된다.

## 7. References


[Medium-What Is iptables and How to Use It?What Is iptables and How to Use It?](https://medium.com/skilluped/what-is-iptables-and-how-to-use-it-781818422e52)

[Medium-A Guide on IPtables](https://medium.com/@mzainkh/a-guide-on-iptables-c4babdc2ea9c)

[LinkedIn-Long story short most used iptables rules! (once forever)Long story short most used iptables rules! (once forever)](https://www.linkedin.com/pulse/iptables-commonly-used-rules-other-stories-short-once-zamani-rad/)
