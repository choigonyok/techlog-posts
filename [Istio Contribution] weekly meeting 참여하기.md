[ID: 48]
		[Tags: ISTIO]
		[Title: [Istio Contribution] weekly meeting 참여하기]
		[WriteTime: 2024-01-03]
		[ImageNames: 3fe11234-585c-4c22-ab48-12348d1cf5f1.png a11a9732-76ba-49dc-a637-277265dc29b8.png]
		
		
##  Content


## 1. Preamble

> 이 글은 Istio Github 레포의 CONTRIBUTING.md를 기반으로 작성되었다.


지난 반 년간 개발자, 데브옵스 엔지니어로서 학습해오면서 DDD, 디프만, AUSG, DND 2회 등 여러 IT 동아리에 지원해왔다. 이 정도면 나의 열정과 적극성을 충분히 어필했다고 생각했지만 매번 탈락했다.

3학년 2학기가 마무리된 지금, 나의 수준을 한 단계 더 높이기 위해 무엇이라도 해야겠다고 생각하던 찰나 오픈소스 컨트리뷰션에 도전해보기로 결심했다.

결과를 목표로 도전하면 금방 지친다. 결과까지 가는 과정을 결과보다 더 즐길 줄 알아야 지치지 않고 해낼 수 있다.

최대한 빨리 컨트리뷰터가 되어야겠다는 결과적인 목적보다, 컨트리뷰터가 되는 과정에서 얻은 배움과 성장을 목표로 몇 달이 되든, 몇 년이 되든, Istio가 나를 질려할 때까지 이거 하나는 계속 붙들고 도전하기로 했다.

## 2. Weekly meeting


Istio에는 여러 working group이 있다. 각 working group은 특정 파트를 담당한다. 예를 들어 Docs 그룹은 사용자 도큐먼트 등을 관리하고, Networking 그룹은 트래픽 관리, TCP 지원, 프록시 인젝션 등을 담당한다. 각 working group에는 리드들이 존재한다.

모든 working group들은 매주 수요일마다 오전 9시에 정기적으로 구글 밋을 이용해서 회의를 진행한다. 수요일 오전 9시는 PST, 태평양 시간대를 기준으로 하기 때문에, 한국 시간으로 변환하면 매주 목요일 오전 2시이다. 이 미팅은 커미터나 컨트리뷰터 뿐만 아니라 Istio에 관심이 있는 누구나 참여할 수 있다고 한다.

그래서 직접 참여해보기로 했다. 어떻게 보면 웬만한 실무보다도 더 수준높은 오픈소스 프로젝트의 실무 회의를 경험할 수 있다는 것이 너무 기대되고 궁금했다.

구글 밋 링크는 아래와 같다. [https://meet.google.com/qza-pfbq-wne](https://meet.google.com/qza-pfbq-wne)

## 3. 새로운 기능에 대한 기여


새로운 기능에 대해 컨트리뷰션하는 방식은


-  해당하는 working group의 슬랙 채널에서 참여자들과 충분히 토의를 거쳐서 다수의 동의를 얻는다.
- 깃허브 issue를 생성해서 requirements 및 usecases와 함께 토의를 이어간다. 제안하는 디자인(아키텍처)와 자세한 기술적 구현에 대한 토의도 포함시킨다.
- 깃허브 issue에서 충분히 토의되고 도입할 가치가 있다고 판단되면, Working Group 리드가 디자인 도큐먼트를 요청할 것이고, 그럼 디자인 도큐먼트를 만들고 링크를 깃허브 issue로 생성한다.
- 슬랙 채널에 알려서 리뷰를 요청한다.
- Working Group 리드는 다른 Working Group과의 협업이 필요하다고 판단되면 주간 정기 회의 때 관련해서 토의를 하게 될 것이다.
- 기술적인 해결방안이 찾아지고, 기능에 대한 동의를 받으면 Working Group의 슬랙 채널에 아키텍처 디자인과 기능 구현 계획에 대한 노트를 공유한다.
- 코딩을 하고 PR을 생성한다.
- 사용사례를 포함한 새로운 기능 관련 도큐먼트에 대한 PR을 Istio.io에 생성한다.
- 리뷰 이후 Merge 한다

PR을 생성할 때는 Contributor license agreements를 신청해야한다.

Github Istio 레포의 리드미에서 자세한 내용을 더 확인할 수 있다. [https://github.com/istio/community/blob/master/CONTRIBUTING.md](https://github.com/istio/community/blob/master/CONTRIBUTING.md)

## 4. 개발환경 세팅


기여하기 위해 개발환경 세팅이 필요하다.

## 5. Bug Issue에 대한 기여


만약 새로운 기능이 아니라 발견된 버그에 대해 생성되어있는 issue에 기여하고싶다면, 해당 이슈에 대해 관심이 있고, 해결해보겠다는 의사를 밝혀야한다. 

##  6. Issue 생성


버그에 대한 이슈를 생성할 때는 사용하는 OS와 사용하고있는 Istio 버전/커밋에 대해 명시해야하고, 가독성을 높이기 위해 5줄 이하의 짧고 간결한 내용으로 이슈를 생성해야한다.

## 7. Summary


이 정도로 Istio 컨트리뷰션에 대한 기본 사항을 이해해보았다. 이제 Istio의 코드를 전체적으로 뜯어보면서 구조 및 아키텍처를 파악해보려고 한다.
