eks 모듈 말고 리소스로 변경해야함

→ eks 모듈에서는 node group의 instance id를 제공하지 않음

이게 있어야 manually 프로비저닝한 nlb의 대상그룹으로 eks 노드들을 지정해줄 수 있음

왜 nlb 자동 프로비저닝을 안하고 nlb를 수동으로 생성함?

→ 자동 프로비저닝을 하면 IaC가 어려워짐, 그리고 route53 도메인과 nlb를 테라폼으로 연동하려고해도 동적으로 프로비저닝된 nlb는 알수가 없어서 이게 불가능함

더 넓은 범위를 테라폼으로 관리하기 위해 불가피한 선택

그래서 TargetGroupBinding 리소스 사용, 이걸 통해 사설서브넷의 인그레스컨트롤러 파드가 aws nlb와 연결되어서 외부 트래픽을 수신할 수 있음

이 때 보안그룹 잘 열어줘야하는 거 필수

테라폼으로 코드를 잘 관리하기 위해


- eks모듈 → 리소스로 변경
- nlb 테라폼으로 수동 생성

[ns-1154.awsdns-16.org](http://ns-1154.awsdns-16.org).

[ns-100.awsdns-12.com](http://ns-100.awsdns-12.com).

[ns-1772.awsdns-29.co.uk](http://ns-1772.awsdns-29.co.uk).

[ns-565.awsdns-06.net](http://ns-565.awsdns-06.net).

region bounded asg

레이어마다 asg를 나눠서 배포

노드 하나에 파드가 쏠리지 않게 hostname topology key를 통해 제어

[AWS Official Docs](https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html#cross-zone-load-balancing) : AWS ELB Cross AZ

→ 이건 유용하나, 다른 AZ로 트래픽을 전송하는 것은 성능이 떨어지기에 약간의 불균형을 감소하더라도 topology key를 이용해 최대한 파드가 노드에 고르게 분포되게 하고, region bounded ASG로 가용영역에 노드를 고르게 분포시키기로 결정

## Trouble Shooting 젠킨스 파드 간 EFS 동기화 안되는 문제


[Stack Overflow](https://stackoverflow.com/collectives/ci-cd/beta/discussions/77554651/load-balanced-jenkins)

sync mount options로 해결
