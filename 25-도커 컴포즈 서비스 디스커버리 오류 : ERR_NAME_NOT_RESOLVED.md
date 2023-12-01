Docker Compose (이하 도커 컴포즈)는 여러개의 컨테이너를 묶어서 작업할 수 있게 해주는 도구이다. 도커 컴포즈는 서비스 디스커버리, 전체 도커 컨테이너 이미지 빌드 및 컨테이너 셍성, 컨테이너간 하나의 통일된 네트워크 연결을 가능하게해준다.

새로운 프로젝트를 설계 및 기획하면서, 운영환경에 쿠버네티스를 통해 어플리케이션을 배포하기로 결정했다. 클라우드에 개발을 위한 전용 서버를 따로 구축하기에는 비용적인 한계가 있어 실현하지 못했다.

차선으로 도커 컴포즈를 사용하여 쿠버네티스의 동일 네트워크, 서비스 디스커버리를 통해 개발 편의성을 높이고, 도커파일 베이스 이미지로 운영환경에서 사용될 우분투 22.04를 사용해 이후 운영환경으로 이전하는 과정에서 생길 작업들을 최대한 줄여보고자 했다. 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/89687131-2C78-47B5-890D-1312152A3E24_2/z4iLC5gumLY6mkLefOKZogiyy7u9OKeRxzbKh06DyKAz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/F12F9318-1739-42B9-8A4B-E78CD47E5F79_2/VGuirdVSfoVRkpAY9aQaoex2v47iLLmHnxj0Z9bATGEz/Image.png)

도커 컴포즈 실행 후 프로그램이 원하는대로 동작하지 않는 것을 확인했다. 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/28B4E0B6-1657-4577-9B21-C48C51974760_2/1PzqOb5jGjpBqESlgiNEJ6F0x5eGD9Gp8z7yLlmmM5Uz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/458D8C31-1B2C-4671-B02B-E1C8E8177095_2/MMMqXIVGIf4zirL5yGro0P8WbHdRh51y1nStxEKQeGEz/%202023-09-19%20%2012.44.17.png)

리액트에서  도커 컴포즈 서비스 이름이 정의된 golang에 테스트용 GET 요청을 보냈는데, golang이라는 이름을 찾지 못하고있었다.

도커 컴포즈에서 서비스 디스커버리를 내부적으로 지원하는 것으로 알고있었는데, 잘못 알고있던 것인지 버전이 바뀌면서 지원을 중단한 건지 혼란스러웠다.

포트를 명시하지 않아도 보고, prefix인 http 프로토콜을 삭제도 해보았지만 여전히 golang이라는 이름으로는 백엔드에 접근할 수 없었다.

이 테스트 프로그램에는 도커 컴포즈의 서비스 디스커버리를 사용하는 코드가 더 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/9C2C9991-9FE5-4BAA-93F5-C0476BE27FE0_2/dzKtsodHVxBmZNLi1hT7AIFtIMvmzHxk6EtC4ZsOjh8z/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/CEE4E6E8-3841-4EBA-9CAF-E0CCE8FE7B25_2/H9Ky6Nd9Ydib21asUx4dmfLMtxzlg6yhxuaLxSVBTqcz/Image.png)

Go에서 MySQL의 도커 컴포즈 서비스 명인 mysql-member로 DB 연결을 시도했을 땐 정상적으로 접근이 되고, MySQL의 테스트 레코드도 정상적으로 조회가 되는 것을 확인할 수 있었다.

트러블슈팅을 위해 직접 컨테이너에 들어가서 DNS 서버 목록을 확인해보기로 헀다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/9DB865E0-2985-4661-AAC1-3284DD1FC838_2/ySXD1qz7XPIeWYymX5I9csOtIokGmlSJCgTd3DUx5zQz/Image.png)

실행중인 frontend 컨테이너에 접근해서 nslookup command를 사용하기 위한 dnsutils 패키지를 설치했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/32441C88-659C-4434-836F-8BDD13F14ED2_2/Cj9ksQRLjxmvZKI9UvNxCN6QQOqepCh7IZEslKakvRcz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/7A378C1C-BD94-4348-A700-5B5161CB632A_2/q3dtFZrByKwcuayQMhvNAuJx4NeQpxwhGN0k57UXgCAz/Image.png)

golang은 172.30.0.4에 잘 할당되어있는 것을 확인할 수 있었다. 도커 컴포즈는 기본적으로 모든 서비스를 default 네트워크로 묶어서 실행하는데, 혹시 이 네트워크 기능도 사라졌나? 그래서 같은 네트워크가 아니라 golang에 대한 IP를 인식하지 못하는 걸까?

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/31C2F6CD-C119-4954-B4A9-09DD678E3848_2/OJWbP7lOh2Aq9zbeOVHrCXLJiBLzjHVWfHJwRZieSJsz/Image.png)

teukgondae_default 네트워크는 정상적으로 잘 생성되어있었다. 가능성은 너무 적지만 혹시라도 react가 golang과 같은 teukgongdae_default 네트워크에 묶여있지 않아서 DNS서버에도 접근하지 못하는 것은 아닐까 해서 frontend 컨테이너의 상세설정을 확인해봤다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/DCB4463B-D2E2-495B-8FB5-24A5845136A4_2/DQM44cDBxkqZ1dyO3lEJ5v9ybS4drNPjrKNcUxCtDqAz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/4DA2780B-C6D2-4DCC-A690-F79BFBA624EF_2/WgouzywcQQKwjIWJLyLoANJOpTOUiTuS8yiHU98Npkwz/%202023-09-19%20%201.39.38.png)

golang과 같은 127.30.0.0/16 대역에 잘 속해있는 것을 확인할 수 있었다.


1. 같은 네트워크에 있다.
2. DNS서버에 golang도 잘 등록되어있다.
3. Golang-MySQL 간의 서비스 디스커버리는 정상 동작한다.

모든 것이 완벽한 상황이었다. frontend 컨테이너에 다시 접속해서 curl로 GET 요청을 다시 날려보기로 했다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/1D782A58-533C-436E-9E85-859B7480DE3F_2/C6qhgPnRJGMO5CjFkOC1YhMKxB76OfEUEFwgs6jmrEcz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/C0FE48B6-459F-4212-A6B1-48C1170BD8EA_2/MYz6UyOOdUrDApNYwYONhDxwKSdxbDi3eTKyxsRVzkYz/Image.png)

curl 요청으로는 서비스 디스커버리가 작동한다! 그러나 여전히 react가 보내는 GET요청에는 오류가 발생하고있었다. 같은 컨테이너 안에서 curl은 맞고, react는 아닌 상황이었다.

결국 해답을 찾았다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/6F86439F-8A40-4383-93A7-7EFBD12A4E96_2/8zaNAnDVUyKziwjkod0EEMq1xSHdJrPVBq0025fi97gz/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/C381D49D-6B6D-44DA-83A1-3369E5C5CEAA_2/NuOuYLzK4z6Cnpj865E3yMSstGXO98cNHnj6fvo9GOsz/Image.png)

리액트가 작성한 코드/빌드파일/번들링은 도커 컴포즈와 golang이 실행되는 서버에서 실행되지 않는다. npm run start로 실행한 코드의 경우에는 번들링되어서, npm run build로 실행한 코드의 경우에는 정적인 빌드파일을 생성해서 웹서버 등을 통해 브라우저에 서빙하게된다.

브라우저는 그 파일을 받아서 자체 자바스크립트 엔진을 통해 코드를 실행해서 화면을 렌더링하게 된다. 이것을 **클라이언트 사이드 렌더링**이라고 한다. 결국 개발자(나)가 작성한 react 코드는 사용자의 로컬에서 브라우저에 의해 실행되게 된다. 이 말인 즉슨 네트워크가 달라진다는 이야기이다.

frontend 컨테이너 내부에서 curl로는 golang 서비스 명으로 요청/응답이 가능했던 이유도 여기에 있다. 컨테이너 자체는 golang과 같은 네트워크 안에 있기 때문에 DNS 서버를 통해 react(127.30.0.3)에서 golang에 대한 IP를 DNS서버(127.30.0.11)에 질의해 golang(127.30.0.4)에 정상적으로 요청을 보낼 수 있었던 것이다.

브라우저에서 접근하는 과정을 살펴보면, npm run start로 실행된 react는 컨테이너 내부의 Wepack-DEV 서버를 통해 코드가 번들링 되어서 브라우저에 전달된다. 내가 localhost:3000으로 접근한 브라우저는, 가상화로 분리된 frontend 및 도커 컴포즈 네트워크가 다른 네트워크이기 때문에 golang이 누군지 나의 로컬 DNS에 아무리 질의해봐도 응답이 없었고, 따라서 ERR_NAME_NOT_RESOLVED 오류를 피드백하게 되었던 것이다.

실제로 우분투 컨테이너 내부가 아닌 MacOS 로컬에서 hosts 파일을 확인해보면

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/4B951977-ECEF-4BC5-94A2-3AC104A401DF_2/Ung5730OGijkwm2z9BSlCMQIEGJccJofqalG5sxifW8z/Image.png)

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/44F64FDD-7128-41E7-93EB-17E111CC312B/2C40FA42-CF0A-4084-A0D1-03A0D2D57296_2/yFyyZrvUbSvqJRdFy05SRI5CDTR4YT1yE3EKpNebRtkz/Image.png)

golang에 대한 DNS 데이터가 없는 것을 확인할 수 있었다.

서비스 디스커버리를 온전히 사용 가능하다는 것이 도커 컴포즈를 사용한 개발환경을 구축의 큰 이유 중 하나였는데, 이것이 불완전해지면서 도커 컴포즈가 아닌 KIND 등의 로컬 쿠버네티스를 통한 개발환경 구축으로 방향을 전환하게 되었다.

요청이 불가능한 게 아니라 그저 frontend에서 backend로 서비스 명을 통해 접근만 불가한 것이라 해결이라는 단어가 부적절한 것 같긴하지만, 해결할 방법이 아예 없는 것은 아니다. 서비스 명을 사용해서 backend로의 접근을 가능하게 하려면 클라이언트 사이드 렌더링이 아닌, 서버 사이드 렌더링을 구현하면 된다. 

frontend 코드가 브라우저에서 실행되어서 네트워크가 달라 서비스 명을 찾지 못하는 것이라면 코드가 서버에서 실행되게 하면 된다. 서버에서 모든 사전 실행을 다 하고 HTML 형식으로 변환해서 틱 던져주면 가능하다.

그러나


1. 서버사이드 렌더링이 애초에 \"frontend의 서비스 디스커버리\"를 위한 목적으로 존재하는 것이 아니다.
2. Go로 서버사이드 렌더링을 구현하기 위해서는 추가적인 복잡성을 감수해야한다.

이러한 이유로 로컬 쿠버네티스를 통한 개발환경 구성을 진행하기로 했다.
