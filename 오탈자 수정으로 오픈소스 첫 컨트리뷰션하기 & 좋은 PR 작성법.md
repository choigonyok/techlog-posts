[ID: 50]
		[Tags: ISTIO DEV]
		[Title: 오탈자 수정으로 오픈소스 첫 컨트리뷰션하기 & 좋은 PR 작성법]
		[WriteTime: 2024-01-04]
		[ImageNames: dda4221e-4f62-4dda-a3ec-1077553530d8.png]
		
		## Content

1. Preamble
2. Typo 발견
3. 레포지토리 fork & branch 생성
4. 좋은 PR을 작성하는 법
5. Typo PR 생성
6. References

## 1. Preamble

> 이 글은 Istio Github 레포의 Wiki: Writing Good Pull Requests를 참고하여 작성된 글이다.


Istiod의 아키텍처 관련 리드미를 읽다가 오탈자를 발견했다. 아키텍처와 코드를 잘 파악한 후에 버그를 찾거나 혹은 Issue가 생성된 버그들을 직접 고쳐서 컨트리뷰션을 하려고 했는데, 우연치않게 오탈자를 찾게 되어서 Istio의 컨트리뷰터가 될 수 있었다.

## 2. Typo 발견


모든 오픈소스들의 도큐먼트는 기본적으로 영어로 작성되어있다. 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/7141486F-C4B1-4646-8E29-AF39034D6CA3_2/1ZzvMNCJEFtCs2IM5sPv2QyzY6Ezdm0VdDwrx9xuy3Az/Image.png)

Istiod의 아키텍처에 대해 작성된 리드미에서 게이트웨이 리소스를 통해 받은 Configuration 인풋값을 어떻게 실제 Proxy 클라이언트에 전달하는지에 대한 섹션을 읽고있었다.

그러다 coverts 라는 단어를 보게되었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/1FDA10BA-31D7-4294-B6C1-58B72162C7BE_2/VTt7DDNpQJ976J26MfFhpVepy3qxatRUt9Eg0BbSuawz/Image.png)

흐름상 converts의 typo인 것 같아 수정을 해보기로 했다.

## 3. 레포지토리 fork & branch 생성


해당 리드미가 위치한 레포지토리의 마스터 브랜치를 포크했다. 그리고 수정을 위한 브랜치를 생성했다. 

## 4. 좋은 PR을 작성하는 법


PR을 작성하기 전 변경사항에 대한 의도를 리뷰어에게 잘 설명해야한다. 그럼 리뷰어는 변경된 코드를 읽기 전에 어떤 방식으로 코드가 흘러가는 것이 좋을지를 먼저 생각해본 후에 코드를 리뷰할 수 있게되고, 더 좋은 리뷰를 받을 수 있을 가능성이 생긴다.

### Issue 생성하기


같은 문제상황에 대한 PR이 중복되게 생기지 않도록 특히 버그 fixing같은 경우 이슈를 생성하는 것이 좋다. 

### WIP PR


WIP는 Work In Progress의 준말이다. WIP PR은 PR의 prefix로 **WIP:** 를 추가함으로써 리뷰어에게 현재 코드가 리뷰를 아직 받을만한 상태가 아님을 알리는 PR이다.

굳이 리뷰를 받지도 않을 코드를 왜 굳이 PR로 생성하는 것인가 의문이 들었었는데, 개발 진행 상황을 공유하거나 현재 코드의 상태를 보고 이후 개발 방향을 논의해야할 필요성이 있을 때 등의 상황에서 WIP PR을 사용할 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/59A2B5F0-D395-47F7-AEDB-FFA1DA23BC64_2/rBWyIdqCueZwbGw6YRZ2xj8jSAAYIo3ePPaLmkNnor8z/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/8AC53F2D-42AE-453C-8926-8D4E1BBFFFD3_2/zDZb4qeSm4PfJn03a4evOxrdPV8nyiYU5flAdeddvqMz/Image.png)

### 좋은 설명을 덧붙이기


애매모호한 제목으로 PR을 생성하면 리뷰어가 해야할 일이 많아진다. PR을 생성한 개발자가 무엇을 하려고 이렇게 코드를 짠 것인지 리뷰어가 그 목적을 코드를 통해 파악해야하기 때문에 오버헤드가 많이 든다.

PR을 생성한 커밋들이 뭘 하고있는 건지를 짧게, 핵심적으로 설명하면 리뷰어의 부담이 크게 줄어든다.

하나의 PR에서 버그도 고치고, 리팩토링도 하고, 기능도 추가하고, 오탈자도 수정하는 등 작업이 많으면 PR description에 bullet-point 마크를 사용해서 작업 내용을 설명해주는 것이 좋다.

```yaml
Subject: Fix bug that causes Foo component to crash when flag is not set.
Description:
+ This is caused by an off-by-one failure during iteration of nukes to launch.
+ Also fixed a race condition by adding a lock on the trigger mechanism that caused concurrent launches that caused a crash in the silo.
+ Adding a TODO for refactoring the code as well, as the cold war is over and we don't need this particular
defense mechanism anymore.
```


이런 설명은 리뷰어 뿐만 아니라 나중에 PR들을 확인해볼 때도 유용하다.

아래는 잘못된 PR 예시이다.

```yaml
Subject: Fix minor bug.
Description:
```


### 짧게


PR이 500줄 이상 넘어갈 정도라면 여러개의 PR로 나누는 것이 좋다. PR이 길수록 리뷰어가 개발자의 의도를 이해하는데 어려움을 겪을 수 있다.

### 커밋으로 정리


여러 작업을 하나의 PR로 묶고싶다면 여러 커밋을 사용해서 리뷰어가 커밋별로 변경사항을 쉽게 파악할 수 있게 해야한다. 

### 테스트 추가


fixing하고있는 이슈가 뭐든지간에 테스트를 추가해야한다. 리뷰어는 테스트를 통해서 개발자의 의도대로 작성된 코드가 정말 잘 작동하는지를 확인할 수 있다. 그리고 같은 이슈가 또 생기지 않도록하는 수단이 될 수 있다.

해결하는 이슈에 알맞게 유닛테스트/통합테스트/E2E테스트를 적절히 구현해야한다.

### 추가 작업 트랙킹


PR과 관련해서 추가적인 작업을 리뷰어가 요청하면, PR 작성자는 해당 작업을 issue로 추가 생성하여서 이후 추가 작업을 트랙킹할 수 있도록 해야한다.

새로운 작업에서 작성되는 이슈 넘버(#로 표현되는 번호)도 루트 PR로 설정해서 해당 작업이 어영부영 넘어가지 않을 것이라는 걸 리뷰어가 확인할 수 있도록 하는 것이 좋다.

## 5. Typo PR 생성


이슈를 수정한 후에 fork한 로컬 마스터 브랜치에 푸시를 하고, istio 레포지토리에 fork 레포지토리의 merge를 요청하는 PR을 작성했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/47DEC89B-2CFE-4162-80D6-A5A40A9C392B_2/V70tBMg4WUsT2A1v8IMDBuyVDnV9c5LEdcVxRJkUMd8z/Image.png)

Istio 레포지토리에서의 첫 PR이다보니 인가/인증 작업이 필요했다. `CONTRIBUTING.md`에 설명되어있는대로 진행했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/8B4200E9-05E6-4D67-888F-EBD7C873951F_2/BbM9VhvwN12sxuyIjmDwxZzKZhuIE3WxlnAtZEjIxgIz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/DBD4EC91-B91C-42CC-9B49-1D62DBFD8A9C_2/XCy7NAVT2rRZFDglFxJJtmyhZjek86j0cRTPuViKRPAz/Image.png)

PR이 성공적으로 등록된 것을 확인했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/183F1568-CD94-4E06-A165-B16EF467BDEE_2/79MIE2tmcnJX24BxvAJ9svferRVimK62a2XdBif4HiUz/Image.png)

며칠 후에 Istio에서 메일이 왔다. 성공적으로 수정한 내용이 마스터 브랜치에 병합되었다는 내용이었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/57422FB1-01A4-4806-9CDE-3228EA52D017/974F97A7-21BA-47E7-952E-7723A8BD0F41_2/Nr24dZfEPPbbYyLExEn4UbcQ9YizObElATn6SYxnmzYz/Image.png)

직접 레포지토리에 들어가 확인해보니 성공적으로 병합된 것을 확인할 수 있었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/8FEB4AD9-C95A-4286-A0E7-495530D236B6_2/BKFwL6U9p0Rt638NOUbM5qdKlAfXyTHa4TPYYcz96W0z/Image.png)

이제 Istio 이슈나 디스커션, PR에서 당당하게 컨트리뷰터로써 내 의견을 피력할 수 있게되었다. 첫 PR은 가벼운 Typo 수정이었지만, Istio 아키텍처와 코드베이스에 대해 더 깊은 이해를 가지고 직접 기능 제안과 구현, 버그 픽싱등을 해낼 줄 아는 컨트리뷰터가 되고싶다는 생각을 하게 되었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/60FB2FA1-7E90-45CC-A991-80E5FC6A6843_2/T5xo0OoHgjQxaXdg1vLtAyaWJRzclsMEyFW090H10q0z/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/B830C062-AD20-4D51-AE9D-F6367DD08CC3/C55E1350-DFAE-4864-A9F6-5AA499DFDE5F_2/Mb3C2Ea8ktr78rM1oy2ucVDJ4P7P3Rc5IxO1HlPz760z/Image.png)

## 6. References


[Writing Good Pull Request](https://github.com/istio/istio/wiki/Writing-Good-Pull-Requests)

[The Contributor License Agreement](https://github.com/istio/community/blob/master/CLA.md)
