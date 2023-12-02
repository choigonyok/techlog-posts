[ID: 38]
[Tags: CI/CD SEC KUBERNETES]
[Title: Jenkins Agent 컨테이너 이미지 빌드 방식 비교 및 구현 (DinD, DOoD, S2I, Kaniko)]
[WriteTime: 2023/11/28]
[ImageNames: ]

## Contents

1. Preamble
2. 분산 빌드 방식의 젠킨스
3. Jenkins Agent 컨테이너 이미지 빌드 방식 비교
4. Kaniko를 활용한 Jenkins Agent의 컨테이너 이미지 빌드 구현
5. Summary

## 1. Preamble


지난 글에서 젠킨스 CI 파이프라인을 구축하기 위한 인프라를 모두 구축했다. 이제 깃허브의 레포지토리에서 코드를 checkout하고, 빌드 및 유닛 테스트를 진행한 후 컨테이너 이미지로 빌드해 도커 허브에 푸시하는 GitOps CI 파이프라인을 구성하고 있던 중, 컨테이너 이미지를 빌드하는 과정에 대한 의문이 생겼다.

젠킨스가 쿠버네티스 외부에 별도 서버로 구축되어있다면 그냥 서버에 도커 엔진을 설치해서 이미지를 빌드하면 되는데, 젠킨스를 Controller / Agent로 나눠 분산 빌드 방식으로 구성했기 때문에, Agent가 컨테이너로 배포되어 문제가 있다고 판단했다.

이 글은 분산 빌드 방식의 Jenkins Agent가 도커파일을 바탕으로 컨테이너 이미지를 빌드하고 컨테이너 이미지 레지스트리에 Push할 수 있는 몇 가지 방법들을 비교해보고, Kaniko를 적용한 이유와 그 구현 과정을 정리하려고 한다.

## 2. 분산 빌드 방식의 젠킨스


어플리케이션이 쿠버네티스 위에 배포될 때, 젠킨스로 CI 방식을 구축하기 위한 두 가지 방식이 있다.


1. Jenkins 서버를 쿠버네티스 외부에 별도로 구축
2. Jenkins 서버를 쿠버네티스 내부에 구축

### 쿠버네티스 외부 구축 장단점


2번 방식이 리소스를 효율적으로 사용할 수 있어서 장점이 많다.

### 쿠버네티스 내부 구축 장단점


젠킨스 Controller(Master)와 Agent(Slave)는 각각 별도의 역할을 맡는다. 컨트롤러는 젠킨스 에이전트를 동적으로 배포하고, 젠킨스 잡을 할당시키는 스케줄러 역할, 에이전트는 컨트롤러에서 요청한 젠킨스 잡을 실질적으로 수행하는 역할

GitOps CI 파이프라인의 핵심은 코드를 빌드하고, 테스트하고, 지속적 전달하는 것이다.

지속적 전달을 하려면 컨테이너 이미지로 빌드를 해야 젠킨스가 직접 배포를 하든, ArgoCD 등의 CD 도구가 배포할 수 있도록 전달을 하든 할 수 있다.

도커가 컨테이너 가상화 솔루션으로 가장 유명하기 때문에, 도커파일로 작성된 컨테이너 이미지를 빌드하기 위해서는 일반적으로 도커 데몬이 필요하다. 컨테이너 및 Pod로 배포되는 젠킨스 에이전트가 도커 데몬을 사용하려면 아래와 같은 몇 가지 방법들 중 하나를 선택해야한다.

## 3. Jenkins Agent 컨테이너 이미지 빌드 방식 비교

1. Jenkins Agent 이미지 안에 도커 엔진 설치
2. DinD & DOoD
3. docker. 약식 사용
4. S2I (Source to Image) 스크립트 작성
5. Kaniko & Buildah

### Jenkins Agent Image 내부에 도커 엔진/데몬 설치


이 방식은 너무 느리다. 애자일과 마이크로서비스가 핫한 배경에 빠른 릴리즈가 있는데, Jenkins Agent Image 내부에 도커를 설치하도록 이미지를 수정한다면 매번 배포할 때마다 도커가 새롭게 설치되게 된다.

에이전트 컨테이너 이미지에서 도커가 설치되도록 구성해서 에이전트가 스핀업될 때마다 정말 짧게 잡아서 10초가 추가적으로 걸린다고 가정하자. 아마존 같은 서비스는 15초에 1번 꼴로 배포가 이루어진다고 한다. 매일 4 * 60 * 24 = 5760회의 배포가 이루어지고, 매일 도커를 배포하기 위해 57600초 = 16시간이 허비된다. 1명의 인력을 매일 도커를 배포하는데 의미없게 날려버리는 것이다.

### DinD & DOoD


그래서 일부 데브옵스 팀에서는 DinD 또는 DOoD를 적용하기도 한다. DinD는 Docker in Docker, DOoD는 Docker Outside of Docker의 준말이다.

DinD는 스핀업되는 에이전트 컨테이너 안에서 추가적으로 도커 컨테이너를 배포하는 방식이다. 도커 컨테이너는 말그대로 도커의 이미지로 실행되는 컨테이너다. 이를 통해 에이전트는 상대적으로 가볍게 도커를 사용할 수 있게된다.

DOoD는 볼륨을 통해 호스트의 Docker 엔진을 공유해서 사용하는 방식이다. docker.sock을 공유하면 에이전트 Pod에서도 컨테이너 이미지를 빌드할 수 있게되고, 에이전트가 배포될 때마다 새로 도커를 설치해야할 필요가 없어서 상대적으로 훨씬 성능이 높다.

그러나 두 방식 모두 보안적으로 위험하다.

DinD와 DOoD 모두 도커라는 도구를 테스트해볼 수 있도록 하는 것이 주 목적이었다.

### S2I


S2I는 Souce to Image의 준말이다. 도커파일을 통해 도커 이미지를 빌드하면 도커 데몬이 필요하고, 그래서 위와 같은 문제가 일어났던 것이기 때문에, 도커파일 없이 소스코드를 직접 빌드해서 컨테이너 이미지로 만드는 방식이다. 이 방식은 스크립트를 직접 구현해야하고 복잡하기 때문에 사람의 실수가 많이 생길 수 있다.

### Kaniko & Buildah


Kaniko는 구글에서 도커 데몬 없이 도커파일을 컨테이너 이미지로 빌드하고, 컨테이너 이미지 레지스트리에 푸시까지 할 수 있도록 해주는 기능을 제공한다. Reddit 커뮤니티를 살펴보니 Kaniko를 몰라서 DinD, DOoD를 실무에서 사용하는 사람들 반, DinD와 DOoD는 보안적으로 좋지 않다며 Kaniko나 Bauh... 를 사용한다는 사람들 반 정도로 나뉘는 것 같다.

Kaniko는 아주 간단하게 도커파일을 이미지로 빌드하고 컨테이너 이미지 레지스트리에 푸시할 수 있게 해준다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1CC7720E-2527-40EC-85CC-4D5AF436EED7/F91178B6-6578-481D-82B9-E73BA5953BCF_2/l6QH2VxsonjldYwObymgG5z8KGIPOyRxJHGtEtZWLbkz/Image.png)

### 4. Kaniko를 활용한 Jenkins Agent의 컨테이너 이미지 빌드 구현


### Docker-Compose로 Kaniko 컨테이너 스핀업


우선 Kaniko의 정상 동작을 먼저 확인하기 위해 Docker-compose를 이용해 Kaniko 컨테이너만 생성해서 잘 이미지가 빌드되고 푸시되는지 확인해봤다.

볼륨으로 도커파일을 Kaniko 컨테이너와 공유할 수 있는데, 파이프라인에서 GitOps로 깃허브 레포지토리에 포함된 도커파일을 Pull할거라 볼륨 필요없음

Dockerfile은 /workspace에 보관

/kaniko/.docker/config.json 에 도커허브 크리덴셜 입력

도커허브 크리덴셜은 Pod로 배포된 Kaniko에는 시크릿으로 config.json을 정의해서 볼륨을 /kaniko/.docker에 마운트

```yaml
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "CREDENTIALS"
    }
  }
}
```


CREDENTIALS에는 {도커허브 ID}:{도커허브 PW}를 base64 인코딩한 값을 넣어줘야한다.

파일의 위치(/kaniko/.docker)와 이름(config.json)은 꼭 지켜져야 인식이 가능하다.

```yaml
echo "DOCKER_HUB_ID:DOCKER_HUB_PW" | base64
```


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1CC7720E-2527-40EC-85CC-4D5AF436EED7/C14426E2-60A9-416F-B88E-B17E50CD055E_2/m0kyTZL4yhY2FKkJVlm7dxMJZwFQcg2uLRrv3DoFVMsz/Image.png)

그리고 executor 실행파일에 플래그를 붙여 실행하면 정상적으로 로그가 출력되고, Docker Hub 레포지토리에 접속하면 정상적으로 이미지가 빌드되어서 이미지 레지스트리에 푸시된 것을 확인할 수 있다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1CC7720E-2527-40EC-85CC-4D5AF436EED7/B14AB3F1-572E-44E0-9E07-1D69DB02C411_2/wxjejX3HnKn4ZBIDyUawzI7nNH9ujfWDujrp0b4UCL0z/Image.png)

### Jenkins Pipeline에 적용


## jenkinsfile


```yaml
pipeline {
    agent {
        kubernetes { // should install kubernetes plugin and configure k8s url, jenkins url, cluster secret
            defaultContainer 'kaniko'
            yaml """
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    imagePullPolicy: Always
    command:     
    - sleep
    args:
    - 9999999
    tty: true
    volumeMounts:
    - name: cfg
      mountPath: /kaniko/.docker
  volumes:
  - name: cfg
    configMap:
      name: config.json
"""
        }
    }
    tools {
        go 'go1.21' // should install GO plugin and configure global tool setting
    }
    environment {
        GO114MODULE = 'on'
        CGO_ENABLED = 0 
        // GOPATH = "${JENKINS_HOME}/jobs/${JOB_NAME}/builds/${BUILD_ID}"
    }
    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'github', url: 'https://github.com/choigonyok/techlog' // should configure github token in global configuration
            }
            post {
                failure {
                    slackSend(color: '#F7A200', message: "Checkout FAILED")
                }
                success {
                    slackSend(message: "Checkout Success")
                }
            }
        }
        stage('Test') {
            steps {
                container(name: 'jnlp') {
                    sh 'go test ./... -cover'
                }
            }
            post {
                failure {
                    slackSend(color: '#F7A200', message: "Test FAILED")
                }
                success {
                    slackSend(message: "Test Success")
                }
            }
        }
        stage('Build') {
            environment {
                PATH = "/busybox:$PATH"
            }
            steps {
                container(name: 'kaniko', shell: '/busybox/sh') {
                    sh '''#!/busybox/sh
                    /kaniko/executor -f `pwd`/build/Dockerfile.dev -c `pwd` --cache=true --destination=achoistic98/test333:latest
                    '''
                }
            }
            post {
                failure {
                    slackSend(color: '#F7A200', message: "Container Image Build and Push FAIL")
                }
                success {
                    slackSend(message: "Container Image Build and Push Success")
                }
            }
        }
    }
}
```


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/1CC7720E-2527-40EC-85CC-4D5AF436EED7/56A214B6-F651-4128-95C9-008562ED4148_2/dxwy8IURrPR7vx6fHLCRsvV3o2uY1usotkVCvTnZByEz/Image.png)

Global configuration에서 Github server 생성, credentials는 secret text로 깃허브 토큰값 넣기

## 5. Summary


## References


[Jenkins Official Docs](https://www.jenkins.io/doc/book/security/controller-isolation/) : Jenkins Controller Isolation

[Jenkins Official Docs](https://wiki.jenkins.io/display/JENKINS/Distributed+builds) : Jenkins Distributed Build

[Tech Blog](https://blog.packagecloud.io/3-methods-to-run-docker-in-docker-containers/) : DinD & DOoD (Docker in Docker & Docker Outside of Docker)

[Blog](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/) : DinD 단점

[Reddit](https://www.reddit.com/r/devops/comments/9lmo2p/building_docker_images_within_a_kubernetes_cluster/) : DinD를 사용하면 안되는 이유

[Medium](https://aws.plainenglish.io/build-docker-containers-on-kubernetes-with-jenkins-and-kaniko-45dc35e89da7) : Kaniko 사용 예시
