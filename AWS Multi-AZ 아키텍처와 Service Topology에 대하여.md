[ID: 32]
		[Tags: CLOUD KUBERNETES PROJECTS]
		[Title: AWS Multi-AZ 아키텍처와 Service Topology에 대하여]
		[WriteTime: ]
		[ImageNames: ]
		
		## Contents

1. Preamble
2. ISSUE - Pending state
3. High Availability를 보장할 순 없을까?
4. Multi-AZ 아키텍처에 대해
5. ELB에서 Service로의 트래픽 전달 과정
6. Imbalanced Loadbalancing을 해결하려면
7. Service Topology
8. 레딧 쿠버네티스 커뮤니티에 질문
9. ISSUE - EBS Volume Permission Denied
10. Summary

## 1. Preamble


Jenkins에서 AWS EBS를 볼륨으로 사용하고 싶었다. 호스트의 루트 볼륨을 그냥 사용하다보니 초기 EKS 설정을 하며 클러스터를 삭제 후 재생성할 때마다 설정해둔 Jenkins 플러그인이나 크리덴셜 설정들도 초기화되어서, 별도의 볼륨으로 관리하여 생산성을 높이고자 했다.
>  이 글은 테크블로그 어플리케이션 리팩토링 과정에서 CSI Driver와 IAM Role의 연결이 완료된 이후 EBS를 PV로 사용하면서 고가용성을 보장할 수 있는 방법에 대해 고민한 과정을 담았다.


2. ## ISSUE - Pending state


CSI Driver를 정상적으로 클러스터에 배포한 이후, Driver가 Pod로 실행되고있으니 이제 설정이 다 끝났다고 생각했다. StorageClass를 통해 볼륨을 프로비저닝하고 PVC를 통해 볼륨을 Claim 하도록 Jenkins Pod의 manifest를 설정한 후, Pod를 배포했다.

근데 노드의 리소스가 충분했음에도 불구하고 Jenkins Pod는 계속 Pending status를 유지하고있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/889B36F5-DF83-4A1C-8BBF-0C094B0388A0/16BAEA12-8F3D-47AF-B8CD-66299597B3DA_2/1HNN3lUkmB7GlrKbwWNzlsm6zb50HfQDiKyqxWcspwYz/Image.png)

로그를 확인해보니 위 사진처럼 volume node affinity conflict가 일어났다는 메시지를 확인할 수 있었다. StorageClass의 설정을 확인해보니 의도한대로 아래 사진처럼 ap-northeast-2의 모든 가용영역에 대해 AllowedTopologies가 설정되어있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/421F3018-E7C5-40D3-BA0B-BDE4F6E02D43_2/IlhHdwDzgYGAI7rBKaBt2RTcERyhBCVIVBdqBKCaUtIz/Image.png)

이 ebs-sc StorageClass를 통해 생성된 PV를 확인해보니

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/8B4F3C3E-7143-46DB-84E3-14711313815B_2/xcUeDsOr4XvDyqIblQFAimsTbtBstn90ZGot107EKQAz/Image.png)

아래 사진처럼 ap-northeast-2b AZ(Availability Zone: 가용영역)에 PV가 생성되어있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/68D4B280-1BA6-4C5E-8CE3-323A992EC32A_2/rE3MqtS95vw1lxIpD5dOGRslSvqWzAcO2pBjjm4tfksz/Image.png)

affinity 충돌이라길래 AWS 콘솔에 접속해 확인해보니 Jenkins Pod가 배포되어있는 유일한 워커노드 한 대는 ap-northeast-2a에 생성되어있었다. Jenkins Pod 역시 하나뿐인 워커노드의 AZ인 ap-northeast-2a에 배포되었고, Pod의 AZ와 프로비저닝된 EBS 볼륨의 AZ가 일치하지 않아 충돌이 생겼다는 걸 알 수 있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/B73AC58B-EB70-446F-9CE6-0CC3F3FACD61_2/6v01WcbudSxg2JwxBpm7UGmrtnFXTRbhZAYuCOQgKiAz/Image.png)

그래서 기존 StorageClass manifest에서 현재 워커노드가 배치되어있는 AZ인 ap-northeast-2a만 남기고 나머지를 전부 주석처리했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/7650BD2A-0F01-49AA-ACFA-F7677E0A6164_2/jSna9jzIGJOU2ZTNM1qwRUqyTB2eIi61ngUSikDSoh8z/Image.png)

Jenkins Deployment 오브젝트를 삭제 후 재배포하니, 아래 사진처럼 정상적으로 EBS가 생성되고 jenkins Pod와 연결되는 것을 확인할 수 있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/5125EE52-65D2-4175-B029-B08F690E130A_2/VQB3x1UqUaVy6WWD8aCNTfuMxBdFlEnDHCVyV5yJ9l4z/Image.png)

지금이야 테스트를 위해 생성한 하나뿐인 워커노드가 ap-northeast-2a에 배치되어있고, Jenkins Pod는 이 AZ에 배포될 수 밖에 없는 상황이기에 StorageClass가 생성할 볼륨의 AZ를 특정해서 문제를 해결할 수 있었지만, 만약 전체 가용영역에 걸쳐 여러 워커노드들을 운영중이었다면 Jenkins Pod가 여러 AZ 중 어디에 배포될지 모르니 StorageClass의 AZ를 특정할 수 없었을 것이다.

그렇다고 manifest에서 주석을 다시 해제해서 전체 AZ로 설정해두어도, StorageClass로 PV가 생성될 때는 Jenkins Pod가 어느 AZ에 생성될 지 알 수 없기때문에  PV의 AZ와 Jenkins Pod의 AZ는 얼마든지 일치하지 않을 수 있다.

따라서 StorageClass를 하나의 AZ로 특정시키고, EBS 볼륨을 사용할 Pod(여기서는 Jenkins)가 해당 AZ에 생성된 노드에만 배포될 수 있게 nodeSelector를 통해 Affinity 설정을 해주어야한다.

## 3. High Availability를 보장할 순 없을까?


이 방법을 통해 문제를 해결할 수는 있으나 무언가 깔끔하지 않다는 느낌을 받았다. 볼륨을 억지로 하나의 AZ에만 배포되도록 한다는 것은 곧 고가용성을 포기한다는 의미와도 같다.

ap-northeast-2a 가용 영역 전체가 상태불능이 되었다고 가정했을 때, 고가용성이 보장된다면 문제없이 어플리케이션이 다른 가용영역으로 옮겨져서 사용가능해야하는데, 꼼짝없이 어플리케이션이 중단되어버리는 사태가 생긴다.

물론 그렇게되면 급한 상황에서는 Jenkins없이 manually 배포할 수도 있고, Jenkins를 다른 가용영역에 새롭게 배포해서 해결할 수도 있지만, 어쨌든 고가용성이 보장되지 못하는 건 사실이고 사용자 경험에 안좋은 영향을 줄 수 있다.

그렇다고해서 단순히 volumeBindingMode를 WaitForFirstConsumer로 변경해서는 안된다. default인 Immediate는 PVC 리소스가 생성될 때 즉시 Dynamic하게 볼륨을 생성하고, 생성된 PV와 자기 자신 PVC를 바인딩하는데, WaitForFirstConsumer 옵션을 적용하면 Pod가 PVC를 요청할 때 이 작업을 수행하게된다.

WaitForFirstConsumer 옵션은 불필요하게 스토리지를 미리 생성하지 않아도 돼서 서버 비용을 절감할 수 있고, Pod가 생성되면서 PVC를 Claim할 때 스토리지가 생성되기 때문에, 파드가 위치한 가용영역을 알 수 있어서 PV가 생성된다는 장점이 있긴하다.

그러나 WaitForFirstConsumer 옵션은 이름처럼 FirstConsumer에 대한 생성만 관여한다. 만약 가용 영역 전체에WaitForFirstConsumer 옵션으로 Binding된 볼륨은 처음 생성된 가용영역에 그대로 남아있게 된다. 결국 앞서 언급한 **volume node affinity conflict**가 다시 일어나게된다.

## 4. Multi-AZ 아키텍처에 대해


이와 관련해 AWS에서 **"Creating Kubernetes Auto Scaling Groups for Multiple Availability Zones"**이라는 제목으로 작성된 공식 블로그 글이 있었다. 이 글은 고가용성이 보장되는 쿠버네티스 구축을 위해 설정할 수 있는 AZ 아키텍처 솔루션들을 소개하고있다.

[AWS Blog Post](https://aws.amazon.com/ko/blogs/containers/amazon-eks-cluster-multi-zone-auto-scaling-groups/)

블로그 글의 결론부터 말하자면 방법이 깔끔하지 않지만 "**어쩔 수 없다**"고 말한다. EBS는 하나의 AZ를 위한 볼륨이고, cross-AZ를 위한 볼륨으로는 대안으로 EFS를 사용해야한다. 그럼에도 이 글을 소개하는 이유는 EBS 이외에도 AZ와 고가용성 보장에 대한 좋은 내용들이 많았기 때문이다.

AWS에서는, 특히 EKS에서는 고가용성을 위해 ASG(AutoScalingGroup)를 통한 오토스케일링을 활용하라고 한다.

- ###  Availability Zone bounded Auto Scaling groups


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/A388735B-0A31-4437-A8A0-55105900F8C2_2/ZGXvh6fEscJvCvEe2xRyIK9a7TxVeKg1txYs1s1eIVwz/Image.png)

EBS 볼륨을 사용하는 Pod의 nodeSelector로 배치되는 노드의 경우엔 위 사진처럼 각 AZ에 대해 ASG를 설정해서 Pod에 리소스 부족 이슈가 생겼을 때 오토스케일링을 가능하게 해야한다. HPA로도 리소스 부족을 충당할 수 있지만 혹시 전체 인스턴스 리소스를 다 사용하고도 부족한 경우가 있을 수 있으니 ASG를 통해 스케일아웃을 가능하게 해야한다.

AWS에서도 이 경우 nodeSelector를 통해 항상 Pod가 PV의 AZ와 같은 AZ의 노드에 배포되도록 설정해야한다고 말하면서, 그렇지 않으면 내가 앞서 겪었던 다른 AZ를 참조하는데서 발생하는 conflict가 생길 수 있다고 언급한다.

"cross-AZ는 안돼서 AZ 전체에 문제가 생기면 어쩔 수 없지만 그래도 이렇게하면 리소스 부족에 대한 장애라도 해결할 수 있다!" 이런 느낌을 글에서 받았다.

ASG를 가용영역 내에 설정함으로써 리소스 부족에 대한 문제는 같은 AZ 안에 오토스케일링을 진행함으로써 해결할 수 있지만, AZ 전체 장애가 생기는 경우에는 따로 대비할 수 있는 것이 없다.

- ###  Region bounded Auto Scaling groups


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/F375D317-F27D-4BAC-9A2F-94EC356C68CA_2/CfxPdXH9J29S5VacqpHxdWtcbATfs8PYuqZfjbRCr3Az/Image.png)

EBS CSI처럼 특정 가용 영역을 지정해야하는 서비스를 사용하는 게 아니라면 일반적으로 위 사진처럼 region-specific, 즉 cross-AZ한 ASG를 사용하는 것이 좋다.

예를 들어 Pod1이 배포된 Node1, Pod2가 배포된 Node2, Pod3이 배포된 Node3이 각가 2개씩 구성되어야한다면 AZ1에는 Node1, 2 / AZ2에는 Node2, 3 / AZ3에는 Node1, 3 이런 식으로 같은 AZ에 중복되는 노드가 최대한 없도록 구성되게 하는 것이다.

이 예시에서 AZ1 전체가 불능이 되어도, ASG의 자동 장애 감지를 통해 AZ1에 배치되어있던 Node1, 2는 AZ2, 3으로 옮겨지게된다.

이러면 리소스가 부족할 때는 다른 다른 AZ에 노드가 오토스케일링 되고, 특정 AZ에 장애가 생기면 해당 AZ의 노드들을 다른 AZ들로 옮길 수 있다. AZ specific한 기능들을 사용해야하는 것이 아니라면 두 가지 장점을 모두 충족시킬 수 있는 이 전략을 사용하는 것이 가장 좋다.

- ### Individual instances without Auto Scaling groups


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/D79ED0E4-7350-479D-8A39-91E88A71824C_2/evZVp06qxcx4kRPRvJmLwd4wUzj8TWc3KxIqhhYhx04z/Image.png)

이 방식은 ASG를 사용하지 않고 각 인스턴스를 독립적으로 배포하는 방식이다. 이 방식은 좋지않다. 당연한 말이지만, EKS에서 ASG에 속하지 않는 노드들은 오토스케일링이 되지 않는다. 쿠버네티스의 큰 장점 중 하나를 버려버리는 것과도 같고, 추가로 AZ failure에 대한 tolerance도 보장되지 않는다.

## 5. ELB에서 Service로의 트래픽 전달 과정


### externalTrafficPolicy: Cluster 방식


AWS ELB는 받은 트래픽을 로드밸런싱 알고리즘에 따라 한 곳에 전달한다. 그 노드가 실제 트래픽이 도착해야할 Service 오브젝트가 존재하는 곳이든 아니든 그냥 일단 보낸다. 로드밸런서 입장에서는 어느 노드가 목적지인지를 알 수가 없다.

그럼 어느 노드가 됐든, 그 노드의 kube-proxy는 iptables을 참조해서 자기 노드에게 필요한 트래픽이 맞는지 아닌지를 판단한다. 아니라면 올바른 목적지로 트래픽을 전달하게된다.

결국 로드밸런서가 막무가내(?)로 아무 노드에나 전달한 트래픽이 실제 트래픽이 가야할 목적지일 확률은 높지 않다. 불필요하게 한 번의 hop이 추가되는 경우가 대부분인데, 이건 로드밸런서가 클러스터 내부의 Pod 배포 상황을 알 수 없기에 어쩔 수 없이 생기는 hop이다. 이를 **Double-Hop Dilemma**라고 한다. 만약 서비스가 커지면 이 불필요한 hop의 추가로 인해 성능 저하 이슈가 생길 수 있다.

이런 과정을 거치는 방식이 LoadBalancer 타입 서비스의 Default 방식인 "Cluster"이다. 이 방식을 externalTrafficPolicy 옵션 설정을 통해 "Local" 방식으로 변경할 수 있다.

### externalTrafficPolicy: Local 방식


반대로 Local 방식은 정확히 목적지인 Pod가 실행되고 있는 노드에만 트래픽이 전달된다. 추가적인 Hop이 생길 일이 없다. Cluster 방식은 Default라 따로 설정해주지 않으면 알아서 적용되고, Local 방식은 아래와 같이 적용해줄 수 있다.

```dockerfile
apiVersion: v1
kind: Service
metadata:
  name: example-service
spec:
  selector:
    app: example
  ports:
    - port: 8765
      targetPort: 9376
  externalTrafficPolicy: Local
  type: LoadBalancer
```


근데 왜 Default가 Cluster인 걸까? Local 방식에 단점이 있기 때문이다. 모든 노드가 같은 Pod 수를 가지고있는게  아니라면, 로드밸런싱이 공평하게 안될 수도 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/56E289D1-94EF-4EE3-95C1-0C613A65A386_2/c0D8AtZaOq0lCiLGMaHUXiLHa4veOddzIFYuBTphPvkz/Image.png)

위 사진은 노드 3개 모두 목적지 Pod를 가지고있는 상황이다. 로드밸런서는 3곳에 대해 라운드로빈 로드밸런싱을 수행한다. 그럼 Pod가 쏠려있는 쪽은 상대적으로 트래픽을 덜 받게되고, AZ3에 있는 Pod가 상대적으로 과부하 상태에 놓이게된다.

이럴 때 두 가지 선택지가 있다.


1.  Cluster 방식으로 쓰기

Cluster 방식은 추가적인 Hop이 생기긴 하지만, 차라리 Hop이 생기는 게 imbalance한 로드밸런싱이 되는 것보다 더 낫다.


2.  Imbalance를 해결하기

Pod를 전체 노드에 걸쳐 고르게 배포할 수 있다면 추가적인 Hop도 안생기고 각 노드도 고르게 트래픽을 받을 수 있게 된다.

## 6. Imbalanced Loadbalancing을 해결하려면


앞서 말했듯이 로드밸런서는 어느 노드에 파드가 더 많아서 트래픽을 잘 처리할 수 있는지를 알지 못한다. 그냥 정의된 로드밸런싱 알고리즘(라운드 로빈 등)에 따라 트래픽을 던질 뿐이다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/C1D5893C-F329-4F0A-B8D8-3EA2F5631149_2/xxlT5PiLZeQxyrSw5c5AD993BeUlb0HkZq7ecEurhKwz/Image.png)

효과적으로 파드를 balance있게 분배하기 위해서 hostname을 활용한 anti-affinity를 활용하면 간단히 해결된다.

```dockerfile
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
           - key: k8s-app
             operator: In
             values:
             - my-app
```


hostname 기반 anti-affinity는 동일한 이름을 가진 노드에 같은 pod가 위치하지 않도록 해준다. 설정이 preferredDuringSchedulingIgnoredDuringExecution인데, prefer로 시작하기 때문에 required와는 다르게 100% 무조건 지켜지는 것은 아니어서 유연성이 있다.

## 7. Service Topology


Service 오브젝트는 Selector와 일치하는 Pod들에게 라운드 로빈으로 트래픽을 전달한다. **그리고 Service 오브젝트는 Pod와 달리 특정 노드에 종속되어있는 오브젝트가 아니다.**

위 사진에서의 예시를 보자. 전체 AZ의 수가 하나이든 몇 백 개이든, 결국 로드밸런서는 한 요청 당 딱 한 번의 라우팅을 하게된다. 근데 만약 로드밸런서의 트래픽을 받은 Serivce 오브젝트가 Pod로 트래픽을 전달하려는데, 다음 Pod가 클라이언트와 다른 AZ에 배치되어있다면 어떨까?

서로 다른 AZ를 넘어다니는 트래픽은 한 AZ 안에서 전달되는 트래픽에 비해 상대적으로 느릴 수 밖에 없다. 가벼운 예시를 들자면 컵라면을 사기위해 집 바로 앞의 편의점을 놔두고 부산의 편의점에 다녀오는 것과 같다. 이왕이면 클라이언트가 위치한 가용영역에 생성되어있는 Pod와 연결되면 빠를텐데, 라운드로빈 떄문에 어쩔 수 없이 부산까지 다녀와야하는 일이 생기게 된다.

쿠버네티스는 이를 위해 서비스 오브젝트에 **Topology Key**를 도입했다. 토폴로지 키는 접근하고자 하는 리소스를 지정할 수 있게 해주는 옵션이다.

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
           - key: k8s-app
             operator: In
             values:
             - my-app
        topologyKey: kubernetes.io/hostname
```


## 8. Reddit K8s community에 질문


초반에 ASG와 Topology key에 대한 개념이 잘 이해되지 않았는데, 정보의 바다라는 구글 안에서도 답을 찾기가 힘들었다. 결국 쿠버네티스 레딧에 처음 글을 써보게 되었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/0B6C1504-0D21-4660-8200-F48058B07827_2/VXX8WePoMyyAFppBt6PQ9FEraDkkMYnR6DRUuwAmlOcz/Image.png)

아래와 같은 친절한 답변이 왔다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/6F61BE0C-7EF6-4097-819E-E0707A9616C7_2/ZyD5HoLaxJx0EdpDECelqsvvHDGQbIi4iaDPvso84BIz/Image.png)

추가적인 질문도 했는데, 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/53B3EC90-DC6E-48DB-917A-5CA37118404D_2/JydsIZuvTSq6YTOedE6ydykiA7OGyUhaOQc56BTkuVYz/Image.png)

아래와 같은 답을 받았다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/41653034-5790-4966-BC7C-E77B18FC6C56_2/xKuVFzXTdryYlzZRalrsTsxDnA7etfhEelmYidxvuwsz/Image.png)

## 9. ISSUE - EBS Volume Permission Denied


다시 쿠버네티스 클러스터로 돌아와보니 잘만 running하고있는 줄 알았던 Jenkins Pod가 15번째 restart를 막 다시 시작하려고 하고있는 것을 발견했다. 로그를 확인해보니,

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/16EDE26C-267C-4620-B736-8140A7EBBF32_2/7UeAcLzEPMNMRmt1y4JIpJJAMajKsVzxEqtRdcNUWykz/Image.png)

Jenkins 이미지 빌드를 위해 작성한 도커파일에서 new jenkins user를 생성하고, jenkins의 작업 디렉토리인 /var/jenkins_home에 대해 jenkins user의 권한 설정을 해두었다. 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/38151DEE-868A-4436-B401-7C52CCA6A8AF_2/6DAkuDlFwDR74O4MZH9COlNIr5MibQVW6xR1UJ3eICMz/Image.png)

Jenkins Pod가 생성된 이후 Pod 내부에서 jenkins user로 컨테이너가 실행중인 상황에서, EBS 볼륨이 default인 root 권한으로 컨테이너에 마운트되면서 jenkins user가 permission이 없어서 볼륨에 접근하지 못한 것으로 판단했다.

이 문제는 InitContainer와 fsGroup, 두 방법을 통해 해결할 수 있다. 

### InitContainers


```dockerfile
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: jenkins
  template: 
    metadata:
      labels:
        app.kubernetes.io/name: jenkins
      name: jenkins
      namespace: devops-system
    spec:
      serviceAccountName: jenkins
      initContainers:
        - name: container-for-chown
          image: alpine:3
          command:
          - chown
          - -R
          - 1000:1000
          - /var/jenkins_home
          volumeMounts:
            - name: jenkins
              mountPath: /var/jenkins_home # Volume mount in initContainer
      volumes:
        - name: jenkins
          persistentVolumeClaim:
            claimName: jenkins
      containers:
      - name: jenkins-master
        image: achoistic98/techlog-jenkins
        ports:
        - containerPort: 8080
        - containerPort: 50000
        resources:
          requests:
            memory: "256Mi"
          limits:
            memory: "512Mi"
            cpu: "1000m"
```


InitContainers를 통해 Pod 생성 시 가장 먼저 실행될 컨테이너를 정의할 수 있다. Jenkins container는 위에 첨부한 이미지처럼 jenkins user로 컨테이너가 실행된다. 따라서 Jenkins container manifest에서 command나 args를 작성해도 애초에 이미지 자체가 superuser 권한으로 실행되지 않기 때문에 volume permission을 수정하기 위한 chmod, chown을 사용할 수가 없다.

그래서 먼저 initContainer를 생성해서 가벼운 이미지인 alpine 이미지를 실행시켜준다. 원래 이미지에서 User를 따로 지정하지 않으면 default로 root 권한으로 컨테이너가 실행된다. 볼륨을 마운트하고, root 권한을 이용해 볼륨의 owner을 수정해주면, 이후에 생성되는 Jenkins 컨테이너가 볼륨에 정상적으로 접근할 수 있게된다.

이 솔루션도 괜찮지만 fsGroup을 통해 추가적으로 컨테이너를 배포하는 과정 없이 더 간단하게 해결할 수도 있다.

### fsGroup


fsGroup은 쿠버네티스 보안 중 SecurityContext와 리눅스 커널에 대해 학습하며 익힌 개념인데, 간단히 말하자면 Pod 내의 모든 컨테이너들에게 같이 공유하는 GID를 하나 추가하는 것이다. 이 fsGroup ID를 통해 모든 컨테이너들은 하나의 볼륨을 공유할 수 있게된다.

만약 볼륨이 추가되었는데 볼륨의 권한이 1000:2000으로 설정되어있다고 가정하자. Pod안에 두 개의 컨테이너가 각각 1000:2000, 1001:2001 의 UID:GID를 가지고있을 떄, 이 볼륨에는 1000:2000 컨테이너만 접근할 수 있다. 볼륨을 공유하고 싶다면 Pod level에서 fsGroup을 지정해주면 같은 볼륨을 공유할 수 있게된다.

왜냐하면 fsGroup 설정은 모든 Pod내의 컨테이너에 같은 fsGID를 부여해주는 것 뿐만 아니라, Pod에 마운트된 볼륨에도 fsGID에 대한 접근권한을 추가해주기 때문이다.

따라서 위의 예시에서 fsGroup을 1500으로 설정하면 두 컨테이너 모두 1500의 fsGID, 볼륨의 owner도 1500으로 설정되게된다. 

Jenkins 예시로 돌아와서, Pod에 컨테이너는 하나이지만 EBS 볼륨의 소유권한은 default인 0:0 (root:root)로 설정되어있기 때문에, 볼륨의 권한을 변경시키기 위해 fsGroup를 사용할 수 있다. 아래 쿠버네티스 공식문서 링크는 fsGroup이 Pod에 마운트된 볼륨의 소유 권한도 변경시켜준다는 내용이 담겨있다.

[Kubernetes Official Docs](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod)

그래서 InitConatiner를 생성해서 볼륨이 마운트된 이후에 다시 한 번 볼륨이 마운트된 디렉토리에 대한 권한을 Jenkins에게 부여해주었다.

```dockerfile
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: jenkins
  template: 
    metadata:
      labels:
        app.kubernetes.io/name: jenkins
      name: jenkins
      namespace: devops-system
    spec:
      serviceAccountName: jenkins
      securityContext: # .spec level securityContext
        fsGroup: 1500
      containers:
      - name: jenkins-master
        image: achoistic98/techlog-jenkins
        ports:
        - containerPort: 8080
        - containerPort: 50000
        securityContext: # .spec.containers level securityContext
          runAsUser: 1000
          runAsGroup: 1000
        resources:
          requests:
            memory: "256Mi"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        volumeMounts:
          - name: jenkins
            mountPath: /var/jenkins_home
      volumes:
        - name: jenkins
          persistentVolumeClaim:
            claimName: jenkins
```


## 10. Summary


블루 그린 배포전략, 멀티 클러스터, 멀티 AZs, 오토스케일링 등 사실 고가용성을 보장한다는 것은 사용자 경험을 좋게해주고, 서비스의 reliability를 높인다. 그만큼 더 많은 비용이 들어가기도 한다. 서비스에 있어 신뢰성, 안정성은 정말 중요하다.

신뢰성은 실제로 깨지기 전까지는 얼마나 문제가 있는지 파악하기 어렵다는 특성을 가진다. (개인적인 생각이다) 그래서 언제 깨져버릴 지 모르는 서비스의 신뢰성을, 깨지지 않도록 더 성장시키고 발전시키기 위해 끊임없이 새 기술을 적용시키고 더 좋은 발전방향을 찾아나가려는 엔지니어의 열정이 필요한 것 같다.

## References


[AWS Blog Post](https://aws.amazon.com/ko/blogs/containers/amazon-eks-cluster-multi-zone-auto-scaling-groups/) - Multi-AZ 아키텍처에 대해

[Kubernetes Official Docs](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod) - fsGroup은 마운트된 볼륨의 권한도 fsGroup으로 재설정해준다

[Kubernetes Official Docs](https://kubernetes.io/ko/docs/tutorials/services/source-ip/#type-nodeport-%EC%9D%B8-%EC%84%9C%EB%B9%84%EC%8A%A4%EC%97%90%EC%84%9C-%EC%86%8C%EC%8A%A4-ip) - ExternalTrafficPolicy가 Cluster일 때 SourceIP를 확인할 수 없는 이유

[externalTrafficPolicy : Cluster vs Local](https://medium.com/expedia-group-tech/request-load-distribution-in-kubernetes-and-aws-e139a3fec4ba)

[Online Base64 Encoding/Decoding Site](https://www.base64decode.org/)

Local 설정의 장점은 요청을 보낸 사용자의 실제 IP주소를 확인할 수 있다는 것이다. 이와 관련한 공식문서 내용을 찾을 수 있었다. [Kubernetes Official Docs](https://kubernetes.io/ko/docs/tutorials/services/source-ip/#type-nodeport-%EC%9D%B8-%EC%84%9C%EB%B9%84%EC%8A%A4%EC%97%90%EC%84%9C-%EC%86%8C%EC%8A%A4-ip)

### NodePort 타입 서비스로 패킷이 전달될 떄


외부에서 서비스 오브젝트로 보내진 트래픽은 NAT가 적용된다. NAT는 Network Address Tranlation의 준말로, IP를 변환시켜준다. 내부에서 외부로 트래픽이 나갈 때 내부의 Private IP를 NAT를 통해 Public IP로 변환시켜준다. 이 방향을 SNAT(Source Network Address Translation)이라고 하고, 반대로 외부에서 접속을 내부에 전달할 때 외부의 Public IP를 내부의 Private IP로 변환시키는 것을 DNAT(Destination Network Address Translation)이라고 한다.

마치 외부 입장에서는 받는 응답이 어디서 온 줄을 모르니까 NAT가 리버스 프록시의 역할을 하고, 내부 입장에서는 받는 요청이 어디서 온 줄을 모르니까 NAT가 포워드 프록시 역할을 하게 된다.

###  SNAT


SNAT에 대해 좀 더 자세히 살펴보자. 


1. 사설에서 응답되는 통신을 위해 필요

만약 사설 IP로 되어있으면 응답을 받은 A측에서 같은 엔드포인트에 또 요청을 보내야할 때, IP가 사설로 되어있어서 올바른 요청을 보낼 수가 없다.

둘 다 사설로 되어있을 때에도, 사설 IP로 보내면 


- 클라이언트는 `node2:nodePort`로 패킷을 보낸다.
- `node2`는 소스 IP 주소(SNAT)를 패킷 상에서 자신의 IP 주소로 교체한다.
- `noee2`는 대상 IP를 패킷 상에서 파드의 IP로 교체한다.
- 패킷은 node 1로 라우팅 된 다음 엔드포인트로 라우팅 된다.
- 파드의 응답은 node2로 다시 라우팅된다.
- 파드의 응답은 클라이언트로 다시 전송된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/81F1BB15-8EF7-405A-9C56-8A26163D3B42_2/U9Lt2Aehc1nEaMjp2DsJFJxjj29SXcDmCkLnsJwMvAUz/Image.png)

이와 반대로 Local로 설정하게되면 앞서 말한 것처럼 다른 노드로는 트래픽을 전달하지 않고, 오직 실제 서비스 엔드포인트가 있는 노드로만 트래픽을 전달하게된다.


- 클라이언트는 패킷을 엔드포인트가 없는 `node2:nodePort` 보낸다.
- 패킷은 버려진다.
- 클라이언트는 패킷을 엔드포인트를 가진 `node1:nodePort` 보낸다.
- node1은 패킷을 올바른 소스 IP 주소로 엔드포인트로 라우팅 한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/5C3B5A52-C05C-4FEB-8C28-4A56CA391217_2/O07gcaQEFGpWaZuFEyUtndJy5de4VqxmuVnXCfbIMlMz/Image.png)

## Loadbalancer 타입 서비스로 패킷이 전달될 떄


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/BA754B6E-308D-4B37-AFFE-8D59D8F3F9D6/0010639E-589D-4E79-975C-74EF6C192FAB_2/q3Bs3QZpNx8w1dSkYE1vyHkLKVGWlU79t8WtckoqoMAz/Image.png)
