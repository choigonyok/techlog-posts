[ID: 60]
[Tags: istio]
[Title: Makefile 함수 & 문법]
[WriteTime: 2024-02-15]
[ImageNames: 047c3f3e-5819-4dcc-a173-103c1a29529c.png]
[Subtitle: Istio 빌드에 사용되는 Makefile.core.mk 분석하기 #1]

Istio 빌드에 사용되는 Makefile 분석하기

Istio 이슈의 디버깅을 위해서는 Istio을 로컬에서 빌드할 수 있어야한다. 

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/5B311BC3-D20D-42AD-AC1E-4B1371189BD4/A9FE460F-F894-4A66-9687-B2EEEED4A68F_2/2AdcbC6y7yrvIo4H617BafXIlijpS1Qw1cSN59cH4tEz/Image.png)


- Makefile / Makefile.core.mk / Makefile.overrides.mk

make build 하면 .PHONY로 build 실행. build는 depend 타겟에 대해 의존성이 존재하기 때문에, depend 먼저 실행한 후에 GOOS, GOARCH, LDFLAGS 환경변수가 정의된 common/scripts/gobuild.sh 스크립트를 실행하게 된다.

```go
.PHONY: build
build: depend ## Builds all go binaries.
	GOOS=$(GOOS_LOCAL) GOARCH=$(GOARCH_LOCAL) LDFLAGS=$(RELEASE_LDFLAGS) common/scripts/gobuild.sh $(TARGET_OUT)/ -tags=$(STANDARD_TAGS) $(STANDARD_BINARIES)
	GOOS=$(GOOS_LOCAL) GOARCH=$(GOARCH_LOCAL) LDFLAGS=$(RELEASE_LDFLAGS) common/scripts/gobuild.sh $(TARGET_OUT)/ -tags=$(AGENT_TAGS) $(AGENT_BINARIES)
```


depend 타켓은 init 타겟을 먼저 실행하고 그 결과를 파이프로 넘겨서 TARGET_OUT에 전달한다.

```go
depend: init | $(TARGET_OUT)

DIRS_TO_CLEAN := $(TARGET_OUT)
DIRS_TO_CLEAN += $(TARGET_OUT_LINUX)

$(OUTPUT_DIRS):
	@mkdir -p $@
```


init 타겟은 pilot 도커파일에 정의되어있는 SHA를 기반으로 엔보이를 다운로드한다.

```go
# Downloads envoy, based on the SHA defined in the base pilot Dockerfile
init: $(TARGET_OUT)/istio_is_init init-ztunnel-rs
	@mkdir -p ${TARGET_OUT}/logs
	@mkdir -p ${TARGET_OUT}/release

# I tried to make this dependent on what I thought was the appropriate
# lock file, but it caused the rule for that file to get run (which
# seems to be about obtaining a new version of the 3rd party libraries).
$(TARGET_OUT)/istio_is_init: bin/init.sh istio.deps | $(TARGET_OUT)
	@# Add a retry, as occasionally we see transient connection failures to GCS
	@# Like `curl: (56) OpenSSL SSL_read: SSL_ERROR_SYSCALL, errno 104`
	TARGET_OUT=$(TARGET_OUT) ISTIO_BIN=$(ISTIO_BIN) GOOS_LOCAL=$(GOOS_LOCAL) bin/retry.sh SSL_ERROR_SYSCALL bin/init.sh
	touch $(TARGET_OUT)/istio_is_init
```


그 전에 `$(TARGET_OUT)/istio_is_init`과 `init-ztunnel-rs`에 대한 의존성이 있다.

```go
$(TARGET_OUT)/istio_is_init: bin/init.sh istio.deps | $(TARGET_OUT)
	@# Add a retry, as occasionally we see transient connection failures to GCS
	@# Like `curl: (56) OpenSSL SSL_read: SSL_ERROR_SYSCALL, errno 104`
	TARGET_OUT=$(TARGET_OUT) ISTIO_BIN=$(ISTIO_BIN) GOOS_LOCAL=$(GOOS_LOCAL) bin/retry.sh SSL_ERROR_SYSCALL bin/init.sh
	touch $(TARGET_OUT)/istio_is_init
```


```go
.PHONY: init-ztunnel-rs
init-ztunnel-rs:
	TARGET_OUT=$(TARGET_OUT) bin/build_ztunnel.sh
```


`$(TARGET_OUT)/istio_is_init`은 `bin/init.sh`와 `istio.deps`를 의존성으로 가지고있다. 이 두 파일은 타겟이 아니기 때문에 makefile이 의존성을 체크해서 해당 파일이 존재하는지 여부만 확인을 한다.

이 파일들을 먼저 실행한다거나 하지는 않는다. 

`init-ztunnel-rs`는 TARGET_OUT 환경변수를 초기화해서 `bin/build_ztunnel.sh` 스크립트를 실행시킨다.

## Istio 빌드를 위한 Makefile에서 사용되는 함수들


makefile에서는 여러 함수들을 사용할 수 있다.

istio를 빌드하기 위한 Makefile.core.mk 파일에서는 `findstring`, `shell`, `realpath`, `lastword`, `error`, `filter`, `if`, `origin`, `info`, `foreach`, `eval`, `call`, `strip` 커맨드가 사용되었다.

각 커맨드에 대해 알아보자.

### findstring


`$(findstring A,B)`와 같은 형태로 사용된다.

```go
ifneq ($(findstring google,$(HOSTNAME)),)
```


스트링 B에 A가 포함되어있는지를 찾는다. 찾으면 `find` 스트링을 리턴하고, 못찾으면 NULL을 리턴한다.

Istio Makefile에서 사용된 위 예시는 변수 `HOSTNAME`에 google이 포함되어있는지를 찾고, ifneq문으로 findstring과 NULL을 비교해서 찾은 상황과 찾지 못한 상황을 분리시킨다.

###  shell


shell 함수는 makefile에서 shell 커맨드를 사용할 수 있게 해주는 함수이다.

### realpath


`$(realpath names…)`와 같은 형태로 사용된다.

인자로 받은 여러 파일들의 캐노니컬한 절대 경로를 표시해준다.

### lastword


`$(lastword names...)` 와 같은 형대로 사용된다.

인자로 받은 값들 중 가장 마지막 단어만 추출한다.

```go
$(lastword $(MAKEFILE_LIST))
```


변수 `MAKEFILE_LIST`는 `Makefile Makefile.core.mk Makefile.common.mk`처럼 공백을 기준으로 여러 Makefile의 리스트로 초기화되어있을 것이다. 그 중 마지막 단어인 `Makefile.common.mk`만을 추출하게된다.

### filter


`$(filter A...,B)`와 같은 형태로 사용된다.

스트링 B를 공백을 기준으로 split해서 스트링 A들과 일치하는 단어들만 남기는 것이다. SQL의 LIKE문처럼 `%`를 사용해서 Prefix/Suffix를 찾을 수도 있다.

```go
Q = $(if $(filter 1,$VERBOSE),,@)
```


변수 `VERBOSE`가 `1 5 6`으로 초기화되어있다고 가정하면 공백을 기준으로 나눈 `1`, `5`, `6` 중 `1`이 포함되어있기 때문에 filter 함수는 1을 반환하게되고, if문에 따라서 ture

### origin


`$(origin A)`와 같은 형태로 사용된다.

변수 A의 출처를 리턴한다. 출처는 `undefined`, `default`, `environment`, `environment override`, `file`, `command line`, `override`, `automatic` 으로 구분된다.

정의되지 않은 변수인지, 정의되었다면 어떤 경로(환경변수/Makefile을 통한 오버라이딩/커맨드라인/파일)로 정의된 변수인지를 리턴한다.

```go
ifeq ($(origin DEBUG), undefined)
```


Istio Makefile에서는 `DEBUG` 변수가 정의되어있는지를 체크하기 위해 origin 함수를 사용하였다.

### info


`$(info text…)`와 같은 형태로 사용된다.

인자로 받은 `text` 값들을 그대로 표준출력으로 출력한다. Makefile에서의 `echo`와 같다.

```go
$(info $(H) Build with debugger information)
```


Istio Makefile에서는 로그 출력을 위해 info 함수가 사용했다.

### foreach


`$(foreach var,list,text)`와 같은 형태로 사용된다.

## References


[GNU-make](https://www.gnu.org/software/make/manual/make.html)
