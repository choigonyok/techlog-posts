[ID: 27]
[Tags: GOLANG DEV]
[Title: Golang CPU 사용량 100% 이슈]
[WriteTime: 2023/10/15]
[ImageNames: e056ddbd-ff9c-4283-be3b-b8510c9d99ac.png]

## Contents

1. Preamble
2. 원인분석 #1 : ArgoCD Deployment
3. 원인분석 #2 : Golang Version
4. 원인분석 #3 : Compile Daemon
5. 원인분석 #4 : Golang third-party Packages
6. 원인분석 #5 : Linux top Command
7. GOMAXPROCS
8. Script Test
9. Dockerizing Test
10. Summary
11. Retrospect

## Preamble


MSA 프로젝트 개발환경을 구성하다가 CPU usage가 100%에 도달하는 이슈가 생겼다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/98FCD007-723B-4C77-80DA-FA06CC4C1716_2/4plvpKSW3rc0SC18Vmh7rDzJBUr1BbxxMRZgXS5Toxkz/%202023-09-23%20%205.19.37.png)

CPU 사용률이 100%에 도달한다고는 하지만 CPU 온도는 평소에는 30~40도를 유지하고 있었고, 로컬 클러스터를 생성하고 초기 Pod들을 배포할 때만 피크가 70도대로 10초 정도 올라갔다 내려오는 상황이었기 때문에, 크게 심각한 상황은 아니었다.

게다가 M2 Pro의 쓰로틀링은 108도이기 때문에, 이대로 놔둔다고 해도 CPU의 성능이나 수명에 나쁜 영향을 주지는 않을 것이었다.

다만 로컬의 CPU가 10코어이며, 아직 본격적인 개발을 이루어지지 않은 상태인데도 불구하고, CPU 사용률이 100%가 된다는 것은 뭔가 내부적으로 이슈가 있는 것이 아닌가 하는 생각이 들었다.

이후에 구현이 이루어지면서 더 늘어날 리소스 사용량까지 생각해서 미리 원인을 찾고 해결하기로 했다.

## 원인분석 #1 : ArgoCD Deployment


아래는 초기 개발환경 구성을 위한 스크립트 일부이다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/E32FAA8A-DA69-4652-87EB-F7AAF866AF0E_2/ILxZiLYZx9fEZnTmN5kBJykKHG2nrxae09q94nXhEicz/%202023-09-23%20%203.44.53.png)

스크립트가 끝나갈 때 CPU가 많이 쓰이는 걸 확인했고, 가장 마지막에 배포되는 ArgoCD에 원인이 있겠다고 판단했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/84A35E04-2731-47BE-85FC-E99923C439FD_2/yPtZZ6rQ4syZ7Nk1ZfGW3LxEngajxAr4Tq7V4fDEZL4z/Image.png)

위 사진은 ArgoCD 매니페스트의 일부이다. 보는 것처럼 20530줄이나 되는 상대적으로 큰 파일이다.

20530줄의 코드에 들어있는 수많은 쿠버네티스 오브젝트를 생성하느라 CPU를 많이 사용한다고 생각하고, 스크립트에서 argoCD Pod의 배포 라인을 제외하고 다시 스크립트를 수행해보았지만 변함없이 CPU는 이전과 유사한 사용률과 온도를 기록하고있었다.

## 원인분석 #2 : Golang Version


스크립트에서 argoCD 배포 라인이 사라지니 가장 마지막에 빌드되는 Golang에서 높은 사용량을 기록하는 것이라고 판단하고, latest 버전인 Golang v1.21.1의 이슈인가 싶어서 Golang의 릴리즈를 확인해보았다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/7C9533C8-D6B2-4527-ACE8-63D716837D56_2/22Pm9hrkb9KW8MuX2fpw8HbREILfFZLVFBolqVyr05sz/Image.png)

내가 지정한 1.21.1 버전은 stable 버전이었다. Golang 버전 이슈도 아니었다.

## 원인분석 #3 : Compile Daemon


Golang의 버전 문제가 아니라면, Compile Daemon의 문제일 수도 있겠다고 생각했다.

Compile Daemon은 컴파일 언어인 Golang에서 핫/라이브 리로딩을 사용할 수 있게 해주는 오픈소스 툴이다. Compile Daemon을 통해, Golang 코드를 수정한 후 다시 빌드하지 않고도 개발환경에서 변경사항을 쉽게 확인할 수 있다.

Compile Daemon이 문제라면 다른 핫/라이브 리로딩 지원 툴인 Air를 써봐야겠다고 생각하며 도커파일을 아래와 같이 수정했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/D7F2F05D-7FA8-49BD-B747-1194B65388F9_2/14yyz8QjcdQvXSVRFvLJJd32ItyaiDdc4OynGqCqKPMz/Image.png)

Compile Daemom을 사용하지 않고, 빌드파일을 생성해 실행하는 방식으로 스크립트를 돌려보았지만 CPU는 여전히 100%의 사용률을 보이고있었다.

## 원인분석 #4 : Golang third-party Packages


compile daemon이 원인이 아니라면 import하고 있는 다른 패키지들의 이슈일 수도 있겠다고 생각했다. 개발환경 구축 테스트를 위해 import한 mysql 패키지나, gin 프레임워크 패키지, 매시지 큐를 위한 rabbitmq 패키지 등 여러 서드파티 패키지들을 import해서 사용하고 있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/87EB35F5-32F5-4B22-A0A3-A00909EA9A40_2/IFNYkTLYUZqzkxwdDhkTy81ztKXJPMgSyzEgp69XZr0z/Image.png)

아래처럼 기존의 모든 로직들을 삭제하고 기본적인 Listner만 남겨둔 채 다시 스크립트를 실행해보았다. 그러나 외부 서드파티 패키지들도 이슈가 아니었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/152D6564-1256-4452-8922-EC0821E46904_2/LPnVYCJ7L3cDdyUfDM6wyXvyJpWrPsvjJZyYPJ7irb8z/Image.png)

## 원인분석 #5 : Linux top Command


결국 정확히 어떤 프로세스에서 리소스를 점유하는 것인지 확인해보아야했다. minikube가 VM을 통해 로컬 쿠버네티스 클러스터를 구현하는 것과는 달리, KIND는 **컨테이너 가상화**를 통해 로컬 쿠버네티스 클러스터를 구현한다.  때문에 KIND 클러스터가 돌아가고있는 컨테이너에 아래 Docker 커맨드를 통해 접근했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/6EF8EEFB-537A-4390-8071-B6AF356EAE63_2/tCdb8xf0mhxV9OfV0T9Sn8CsuQEdF9ywb2ZDPc5tQjMz/Image.png)

리눅스 top 커맨드를 사용해서 리소스 사용 프로세스를 확인하면서 스크립트를 다시 실행했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/7602C3CB-9CC4-4529-93FF-49E95F7C98F4_2/5a5GAKsWXAErEMIJutcev6BqqDnVD4g1t8BcCRP1ZXgz/Image.png)

그러다 아래 사진과 같이, Go의 compile 프로세스가 혼자서 670%, 약 7개의 코어를 순간적으로 독점하는 것을 확인할 수 있었다. 심지어는 스크립트 실행 중 백그라운드로 켜두었던 음악이 리소스 부족으로 강제 중단되는 일까지 생겼다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/778F7F1E-576E-495B-9410-1CE400AE1498_2/U5RtqPJVYBZXFMButAsQaqW2R6Uwfs1kVsbyvWkTCL0z/Image.png)

프로젝트의 Java도 동일한 컴파일 언어인데, 왜 Golang만 유난히 이렇게 많은 양의 리소스를 차지하는 것일까? 결국 나와 같은 상황을 겪었던 한 사람의 질문을 통해 답을 찾게 되었다.

## GOMAXPROCS


[StackOverFlow](https://stackoverflow.com/questions/39064914/why-go-takes-so-much-cpu-to-build-a-package)

Golang에는 GOMAXPROCS라는 환경변수가 있다. 이 GOMAXPROCS을 통해서 Golang에서 사용할 CPU의 숫자를 설정할 수 있다. 기본적으로 Golang은 최대한 빠른 컴파일을 위해 CPU를 최대한으로 끌어다 쓴다. 이 과정에서 CPU 사용률 100%까지 도달하게 된 것이다. 

도커파일에 GOMAXPROCS 환경변수를 지정해주어서 Golang이 쓸 수 있는 CPU 수를 제한할 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/D74FD8FA-FDC1-465D-9D87-9A0A7D5E0D7C_2/gPZ5i7hDSFqcqiYF2XQyTiIK2zskA1Bkq5IklGw4pDEz/Image.png)

## Script Test


Golang에게 CPU 코어 수를 얼마나 할당해야할지 판단하기 위해 테스트를 진행하기로 했다.

테스트는 GOMAXPROCS가 1일 때와 전체 코어 수인 10일 때, Golang Docker를 빌드하고 Run 하는데 걸리는 시간 및 CPU의 피크 온도를 측정해보았다.

### GOMAXPROCS = 1


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/226E7661-60F4-467A-A0F1-1BE60F918D3C_2/bBKbFZRcJBjd0ZWZo81w30R0ensyqsUZX1JzalfpaKUz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/42F7E310-5525-495B-A6E5-DAA8B2E0D281_2/vTvpihkfsYy8JvMPpltA8y8m7OckgCYZXEaEycxMo7Uz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/728B92E8-5CEE-4641-8B9A-EED04C696EA6_2/tqxfFB4IKxpsqwgB82Wzy6LnWZykuOmeJ0fki7YLAqgz/%202023-09-26%20%203.38.28.png)

스크립트 실행시간은 약 3분 13초가 소요되었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/3F5D7F38-A9C9-4978-80C6-BFFC95050514_2/90cJ67FtoMWx8VKQoKNxR5opNxyroc49f5govllDA5oz/Image.png)

온도는 최대 80.1도까지 올라갔다. 

### GOMAXPROCS = 10


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/67B002B0-6C1A-47C4-9129-E9957EB69AAB_2/wUkximCQtGENgLIGEqFxKAFZYRMNXjSIMIbn14nQ6Fwz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/722E2D26-E954-4E22-9A7C-6BAADB402492_2/ColewX1vHKIcESEftIoWayheQRH85XH9QQy8BDlJFsIz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/376AD7A3-99B6-43A6-A13C-A86F2EEE7A63_2/lhjf5xH1ON5fCz8q7kXxbUMoCgxGY5xjJfIdWbveP5kz/Image.png)

스크립트 실행시간은 약 3분 53초가 소요되었고, 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/E6423ABD-E002-4F7F-9F48-69F61B4AFF4A_2/JLyYUZG1tgFn5NXyW8olUQFNaqJm5XrNxXsIIdMtSxsz/Image.png)

최대 온도는 89.4도까지 올라갔다.

코어를 많이 쓸 수록 온도는 올라가면서 속도는 느려지는 것 같아보이는 상황이 생겼다. 스크립트가 Golang 하나만 빌드/실행하는 것이 아니고, 중간중간 Pod간 의존성을 위해 sleep time을 추가했다보니 이런 결과가 나온 것 같았다.

Golang만을 대상으로 컨테이너를 따로 빌드해서 실행해보기로했다.

## Dockerizing Test


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/3EAE61C7-3480-4381-A892-0A43B47EBF01_2/yPNnSlBTC2IFoeHrbR1cDJsgaKn6kSxCr1hIayo6ToIz/Image.png)

위 사진처럼 테스트를 위한 스크립트를 짜고, 실행했다.

### GOMAXPROCS = 1


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/5551F6C6-E209-430F-93C6-F436603CCC47_2/Y28j9Q6DtPVibIh6DCasbbUWzCurfArXCFL1BrJZPdkz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/9BFE4403-3E7E-4A7A-9666-7485C71BE162_2/EBDMBLqpbdTKf0xi2jmMFLgAMUnj5s7r5hiR7ELDkeYz/Image.png)

코어를 하나 할당했을 때 실행시간은 2분 37초, 최고온도는 61도까지 올라갔다.

### GOMAXPROCS = 10


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/46FC979D-341A-47FC-8FF9-FF6AC1FCCF9B_2/2dzb2xnYmTGpvYyloM1Lcrgld9YECNLuh6KdfxyqzGkz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/6F00E7BD-5F76-41C0-9DE7-6737CFDECBC6/FDF4E22B-9180-4820-8023-C63262BD7AC4_2/1iuDZjHoibV3CLJlktZ3ZybjU8pfDXXUdMJGOuylgBIz/Image.png)

코어를 10개 할당했을 때 실행시간은 1분 29초, 최도온도는 89도까지 올라갔다.

약 1분이 넘는 실행시간 차가 있었다. 2분 30여초 중 1분 차이이기 떄문에 거의 2배에 가까운 시간차가 난 셈이다.

온도 역시 차이가 많이 있었다. 평균 온도는 27.7도 차이, 코어의 피크 온도는 38.2도 차이가 났다.

## Summary


테스트 결과를 통해서 아직 개발환경 구성단계이고, 개발하면서 로직이 쌓여가면 리소스를 더 많이 사용하게 될 것이기 때문에, 우선 각 Golang 백엔드 서비스마다 2개의 코어를 할당해두기로 했다. 이후 온도 변화 추이를 지켜보며 추가적으로 CPU를 할당해 나가기로 결정했다.

## Retrospect


트러블 슈팅을 하는 과정에서 가장 먼저 리소스를 잡고있는 프로세스가 무엇인지를 먼저 확인해보았다면, 이슈 해결 시간을 단축할 수 있었을 것 같다. 어림짐작을 통해 이슈를 해결해가는 것이 아니라, 정확히 문제 상황이 무엇인지를 인식하고, 이슈의 범위를 좁혀나가며 해결하는 것이 더 효과적인 트러블 슈팅 방법이라는 것을 배웠다.