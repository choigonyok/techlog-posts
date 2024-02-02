[ID: 47]
[Tags: INFRA GOLANG KUBERNETES DOCKER]
[Title: MySQL 마스터-슬레이브 Replication 구성 및 자동화하기]
[WriteTime: 2023-12-27]
[ImageNames: d4ae59bf-ff56-4400-add1-7b56071e8f7f.png 044c3458-5add-4351-ba6d-6919927bb27f.png f8e8c9fa-3991-418e-9648-8dc1573205b2.png e5f4c42e-35f6-4f1e-972f-9c3c486c3893.png e4c5ba93-0c2d-49a4-a863-cd0d63b2134d.png]
		
## Content

1. Preamble 
2. Replication
3. 구현
4. 자동화

## 1. Preamble

백엔드의 코드를 Package Oriented Design으로 리팩토링 하게되면서, 자연스럽게 데이터베이스 모델링도 다시 하게 되었다.

또 기존 백엔드, 프론트엔드, 데이터베이스 서버가 각각 하나의 AWS EC2 인스턴스 위에 구축되어있던 Kubeadm 기반의 싱글 쿠버네티스 클러스터에서 EKS + Multi-AZ 기반 클러스터로 이전하게 되면서 데이터베이스의 고가용성 역시 보장되어야했다.

## 2. Replication


replication은 여러 데이터베이스의 데이터를 일치시키기 위해 하나의 데이터베이스에서 다른 데이터베이스로 데이터를 복제하는 것을 의미한다.

일반적으로 복제를 담당하는 하나의 Master DB가 있고, 나머지 Slave DBs가 있다.

마스터 DB에서 데이터를 변경(생성, 수정, 삭제)하면, 마스터 DB가 슬레이브 DB에 데이터 변경사항을 복제시키고, 마스터, 슬레이브 구분없이 전체 DB는 읽기를 수행한다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/3D38522E-E46A-4A72-808A-CC313028AA73/BC5799BF-39BF-4459-B7B6-E7359AAE1A01_2/O3neFc5SVN0mW5sou8E6g8jZfxlRVMYc9dUmtXKmxxwz/Image.png)

데이터베이스에서 replication은 크게 두 가지 목적으로 사용된다.


- failure tolerance
- load balancing

첫 번째 목적인 failture tolerance가 주 목적이고, 로드밸런싱은 추가적으로 얻을 수 있는 장점에 가깝다.

데이터베이스에서 일반적으로 CRUD중 R의 비율이 압도적으로 큰 경우가 많기 때문에 최대한의 성능을 내기 위해 조회 쿼리 요청은 마스터 DB를 포함한 전체 DB에 로드밸런싱 되도록 한다.

## 3. 구현


MySQL에서는 기본적으로 마스터/슬레이브 replication 기능을 지원한다. 다만 방식이 복잡하다. 요약해서 살펴보면,


- master DB에서 slave DB가 사용할 DB user/password 생성하기 (읽기 권한만 주기 위해)
- 만든 DB user에 자신(master)의 slave가 될 수 있는 replication 권한 부여하기
- 각 slave에서 master의 로그 파일 이름, 로그 위치, 권한을 부여받은 user/password와 함께 slave로 설정하기

이러한 과정을 거친다.

### Master DB


마스터 데이터베이스에서 슬레이브 DB가 복제에 사용할 유저를 생성한다. 이 유저를 통해 슬레이브 DB에서 마스터 DB에 접근할 수 있게된다.

```bash
CREATE USER 'replicas'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replicas'@'%';
FLUSH PRIVILEGES;
```


아래는 슬레이브 DB를 위한 config 파일이다. read_only로 설정했기 때문에 슬레이브 DB는 Write 작업을 수행하지 못한다.

```bash
[mysqld]
character-set-server=utf8
server-id = 2
read_only
slave-skip-errors = 1032, 1452

[mysql]
default-character-set=utf8

[client]
default-character-set=utf8
```


아래는 마스터 DB를 위한 config 파일이다. 참조할 마스터 DB 로그파일의 prefix와 참조할 DB명을 지정해준다.

```bash
[mysqld]
character-set-server=utf8
server-id = 1
log_bin = mysql-log
binlog_do_db = blogdb
slave-skip-errors = 1032

[mysql]
default-character-set=utf8

[client]
default-character-set=utf8
```


마스터 DB를 생성할 때 config 파일, 슬레이브 DB를 생성할 때 config 파일을 주입해준다.

마스터 DB에서 `show master status` 커맨드를 실행하면 마스터 DB의 로그 파일 이름과, 로그 파일의 현재 위치인 Position 등의 데이터가 출력된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/3D38522E-E46A-4A72-808A-CC313028AA73/76DEEB02-37AE-463D-B989-70CC8740C669_2/2ff1bkp96NJcIbhKQwLyBIxl7RvKtyyaqmqnSHUpxkEz/Image.png)

이 데이터를 슬레이브 DB에 입력해주어야한다. 두 슬레이브 DB에 각각 접속해서 아래 커맨드를 실행해준다.

```bash
STOP REPLICA IO_THREAD FOR CHANNEL '';
CHANGE MASTER TO MASTER_HOST='MASTER_DB_IP', MASTER_PORT=3306, MASTER_USER='replicas', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-log.000003', MASTER_LOG_POS=567355, GET_MASTER_PUBLIC_KEY=1;
SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;
START SLAVE
```


그리고 슬레이브 DB에 접속해서 `show slave status;` 커맨드를 실행하면,

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/3D38522E-E46A-4A72-808A-CC313028AA73/82A7327B-8C20-4E3B-A68B-04733445CB77_2/BHmC5gYy157vJyNcV4x5bbxyAy9WlPLrkGvI37qt0Y0z/Image.png)

이런 식으로 `Slave_IO_Running` 필드와 `Slave_SQL_Running` 필드가 모두 `YES` 인 것을 확인할 수 있게된다. 마스터/슬레이브 간 레플리케이션이 성공적으로 연결되어있다는 것을 의미한다.

## 4. 자동화


마스터/슬레이브 구축 과정은 자동화가 꼭 필요한 부분은 아니다. 한 번 설정해두면 반영구적으로 지속되기 때문이다. 다만 나의 경우에는 자동화가 필요했다.

클라우드 비용으로 인해 서버를 내렸다 올리는 횟수가 빈번헀기 때문에, 매번 블로그 서버를 다시 시작할 때마다 반복적인 과정을 줄이기 위해 자동화를 구축하기로 했다.

### 마스터/슬레이브의 비동기적 실행


쿠버네티스에서는 기본적으로 Pod의 동기적 실행을 지원하지 않는다. Pod가 만약 동기적으로 실행된다면 파드가 Crash 되었을 떄 동기적 조건으로 인해 다시 재실행되지 않을 수 있기 때문이다.

앞서 말했듯이 슬레이브 DB는 마스터 DB가 먼저 구성을 마친 뒤 생성되는 로그 파일 이름, 위치, 유저 이름, 패스워드를 바탕으로 슬레이브 설정이 되기 때문에, 마스터 DB가 정상적으로 실행을 마친 뒤에 실행되어야한다.

쿠버네티스에서 마스터 DB 파드를 먼저 배포한다고 해도 실행만 먼저 될 뿐이지, 마스터 DB의 모든 작업이 완료된 후에 슬레이브 DB 파드가 실행되도록 하는 것은 불가능하다.

그렇다보니 어쩔 때는 마스터/슬레이브 구성이 잘 되고, 어쩔 때는 슬레이브 DB 2개 중 하나만 슬레이브가 되는 식으로 랜덤하게 설정이 되었다.

그래서 구성 관리를 파드에서 하는 것이 아니라, 별도의 파드를 생성해서 구성하도록 방식을 변경했다.

마스터/슬레이브 파드를 우선 실행만 시킨 후에, 별도의 golang으로 작성된 쿠버네티스 Job를 실행시켜서 동기적으로 각 작업을 수행할 수 있도록 변경했다.

이를 위해서 기존 cmd 디렉토리에 백엔드 실행파일 생성을 위한 blog.go 이외에 init.go가 추가되었다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/3D38522E-E46A-4A72-808A-CC313028AA73/B89E14F9-AC13-49D7-A4F4-B53B1F78F4A4_2/SDaJRBBb5veIxoF1V1OlfFYj4FSpUH9X3XJxdoiHauwz/Image.png)

init.go의 코드는 아래와 같다.

```go
package main

import (
	"database/sql"
	"fmt"
	"net"
	"os"

	_ "github.com/go-sql-driver/mysql"
	"github.com/joho/godotenv"
)

var (
	databaseDriver            = "mysql"
	masterDatabaseServiceName = "mysql-ha-0"
	slave1DatabaseServiceName = "mysql-ha-1"
	slave2DatabaseServiceName = "mysql-ha-2"
	masterSVCName             = "mysql-ha-0.default.svc.cluster.local"
)

func main() {
	godotenv.Load(".env")
	var rootPassword = os.Getenv("DB_PASSWORD")
	var databaseName = os.Getenv("DB_NAME")

	var file string
	var position uint32
	var t1 string
	var t2 string
	var t3 string

	addresses, err := net.LookupHost(masterSVCName)
	if err != nil {
		fmt.Println("Error:", err)
		return
	}

	master, err := sql.Open(databaseDriver, "root:"+rootPassword+"@tcp("+masterDatabaseServiceName+")/"+databaseName)
	if err != nil {
		fmt.Println(err.Error())
		return
	}
	defer master.Close()

	err = master.QueryRow(`SHOW MASTER STATUS`).Scan(&file, &position, &t1, &t2, &t3)
	if err != nil {
		fmt.Println(err.Error())
		return
	}

	slave1, err := sql.Open(databaseDriver, "root:"+rootPassword+"@tcp("+slave1DatabaseServiceName+")/"+databaseName)
	if err != nil {
		fmt.Println(err.Error())
		return
	}
	defer slave1.Close()

	_, err = slave1.Exec(`STOP REPLICA IO_THREAD FOR CHANNEL ''`)
	if err != nil {
		fmt.Println(err.Error())
		return
	}

	_, err = slave1.Exec(fmt.Sprintf(`CHANGE MASTER TO MASTER_HOST='`+addresses[0]+`', MASTER_PORT=3306, MASTER_USER='replicas', MASTER_PASSWORD='password', MASTER_LOG_FILE='%s', MASTER_LOG_POS=%d, GET_MASTER_PUBLIC_KEY=1`, file, position))
	if err != nil {
		fmt.Println(err.Error())
		return
	}

	_, err = slave1.Exec(`SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;`)
	if err != nil {
		fmt.Println(err.Error())
		return
	}

	_, err = slave1.Exec(`START SLAVE`)
	if err != nil {
		fmt.Println(err.Error())
		return
	}
 
	slave2, err := sql.Open(databaseDriver, "root:"+rootPassword+"@tcp("+slave2DatabaseServiceName+")/"+databaseName)
	if err != nil {
		fmt.Println(err.Error())
		return
	}
	defer slave2.Close()

	_, err = slave2.Exec(`STOP REPLICA IO_THREAD FOR CHANNEL ''`)
	if err != nil {
		fmt.Println(err.Error())
		return
	}

	_, err = slave2.Exec(fmt.Sprintf(`CHANGE MASTER TO MASTER_HOST='`+addresses[0]+`', MASTER_PORT=3306, MASTER_USER='replicas', MASTER_PASSWORD='password', MASTER_LOG_FILE='%s', MASTER_LOG_POS=%d, GET_MASTER_PUBLIC_KEY=1`, file, position))
	if err != nil {
		fmt.Println(err.Error())
		return
	}

	_, err = slave2.Exec(`SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;`)
	if err != nil {
		fmt.Println(err.Error())
		return
	}

	_, err = slave2.Exec(`START SLAVE`)
	if err != nil {
		fmt.Println(err.Error())
		return
	}
}
```


```go
addresses, err := net.LookupHost(masterSVCName)
if err != nil {
    fmt.Println("Error:", err)
    return
}
```


golang에서 쿠버네티스의 서비스 디스커버리를 활용하기 위해 net 패키지의 LookupHost 함수를 사용했다. DNS를 검색해주는 함수이다.

이 함수를 통해 마스터 DB인 mysql-ha-0의 IP 주소를 찾는다.

sql.Open()으로 DB와 TCP 연결을 맺을 때는 서비스 이름만 입력해도 쿠버네티스의 서비스 디스커버리를 통해 MySQL DB와 커넥션이 가능한데, 이 주소는 그 용도가 아니라 이후에 슬레이브 DB에서 마스터 DB를 지정해줄 때 사용하기 위한 IP 주소이다.

```go
master, err := sql.Open(databaseDriver, "root:"+rootPassword+"@tcp("+masterDatabaseServiceName+")/"+databaseName)
if err != nil {
    fmt.Println(err.Error())
    return
}
defer master.Close()
```


가져온 주소로 마스터 DB와 tcp 연결을 맺는다.

```go
err = master.QueryRow(`SHOW MASTER STATUS`).Scan(&file, &position, &t1, &t2, &t3)
if err != nil {
    fmt.Println(err.Error())
    return
}
```


마스터 슬레이브의 로그 파일 이름과 위치를 전달해야하기 때문에, database/sql 패키지의 QueryRow 메서드를 사용해서 질의한다.

Query와 달리 QueryRow는 레코드 하나만 조회할 때 사용한다.

```go
_, err = slave2.Exec(fmt.Sprintf(`CHANGE MASTER TO MASTER_HOST='`+addresses[0]+`', MASTER_PORT=3306, MASTER_USER='replicas', MASTER_PASSWORD='password', MASTER_LOG_FILE='%s', MASTER_LOG_POS=%d, GET_MASTER_PUBLIC_KEY=1`, file, position))
if err != nil {
    fmt.Println(err.Error())
    return
}
```


슬레이브 DB도 마찬가지로 tcp 연결을 맺어주고, 가져온 IP 주소와, file, position 변수에 저장되어있는 로그 파일 이름, 위치를 통해 슬레이브 설정을 해준다.

```go
_, err = slave2.Exec(`SET GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;`)
if err != nil {
    fmt.Println(err.Error())
    return
}
```


이 부분은 특정 MySQL 에러 코드가 발생했을 때, 그 에러를 한 차례 무시하고 진행시켜주는 커맨드이다. 이게 없으면 MySQL끼리 Replication을 진행하다가 버그로 인한 에러가 발생하면 마스터/슬레이브 커넥션이 종료되어서 더 이상 Replication이 정상적으로 이루어지지 않는다.

```go
_, err = slave2.Exec(`START SLAVE`)
if err != nil {
    fmt.Println(err.Error())
    return
}
```


그리고 슬레이브를 시작해준다.

### SQL 파일 & Dockerfile & .cnf 파일

> 여러번 배포를 하다보니 마스터 DB가 두 개 생성되거나 슬레이브 DB만 3개 생성되는 이슈가 랜덤하게 생겼다.


원인을 살펴보니 Dockerfile 구성에 문제가 있었다. 

`Dockerfile.mysqls`

```dockerfile
FROM --platform=linux/amd64 mysql:latest

COPY ./hack/mysql-slave.cnf /cfg/mysql-slave.cnf
COPY ./hack/mysql-master.cnf /cfg/mysql-master.cnf

COPY ./hack/init-slave.sql /cfg/init-slave.sql
COPY ./hack/init-master.sql /cfg/init-master.sql
```


하나의 마스터 MySQL 파드, 두 개의 슬레이브 MySQL 파드를 배포하려면 각각 다른 MySQL 설정파일을 전달해야한다.

하나의 도커파일로 마스터와 슬레이브 모두를 구성하려다보니 마스터용/슬레이브용 구성 파일들이 함께 포함되어있어서 매번 배포할 때마다 랜덤하게 마스터/슬레이브가 구성되는 것으로 판단했다.

그렇다고 도커파일을 분리할 수는 없었다. 도커파일을 분리한다는 것은 파드의 배포역시 마스터/슬레이브를 분리해야한다는 것인데, 그렇게 되면 마스터 DB 파드에 장애가 생겨 슬레이브 DB 중 하나가 마스터 DB로 선정되어야할 때, 마스터 DB로 구성되기 위한 설정파일들이 존재하지 않기 때문에, 새로운 마스터 DB를 선정할 수가 없어진다.

즉, 모든 DB는 마스터용/슬레이브용 구성파일을 모두 갖고있되, 처음 실행시에는 마스터 1개, 슬레이브 2개가 정확히 실행되어야한다.

이를 해결하기 위해 쿠버네티스의 `StatefulSet` 리소스와 `initContatiner` command를 활용했다.

StatefulSet 리소스를 활용해서 replicas를 생성하면 파드 이름 뒤에 랜덤스트링이 붙지 않고 0부터 increment 되면서 이름이 생성된다.
> StatefulSet으로 파드를 생성했을 때


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/3D38522E-E46A-4A72-808A-CC313028AA73/CE446106-3627-4402-9452-8EA7A1C28C87_2/d8fjCfCol5jeAmTP3DmPUFxFx51zyl0VgMfjw7LrGZwz/Image.png)
> StatefulSet이 아닌 일반 Deployment로 파드를 생성했을 때


![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/3D38522E-E46A-4A72-808A-CC313028AA73/B0F9D568-93C7-4946-A5B9-CF756B38CDF0_2/WyMkIMGay1dV6hlxiC5QBqrVBdKxW3q491D6aCaez0Ez/Image.png)

이렇게 되면 매번 파드의 이름이 고정되어있기 때문에 initContainer를 통해 스크립트를 실행해서 필요한 파일만 사용할 수 있다.

MySQL에서 .cnf 파일이 위치해야하는 디렉토리가 /etc/mysql/conf.d에 위치해야하는데, 마스터용/슬레이브용 .cnf 파일을 모두 임시로 /cnf에 저장해두고, 이름에 따라서 initContainer를 통해 하나의 파드만 마스터용 설정파일을 /cnf에서 /etc/mysql/conf.d로, 나머지 두 개의 파드는 슬레이브용 설정파일을 /cnf에서 /etc/mysql/conf.d로 이동시키는 방식이다.

이렇게되면 이후에 슬레이브 DB가 새로운 마스터 DB로 선정되었을 때, /etc/mysql/conf.d에 있던 슬레이브용 설정파일을 삭제하고, /cnf에 저장되어있는 마스터용 설정파일을 이동시켜서 재시동하면 마스터 DB의 역할을 수행할 수 있게 될 것이다.

`database-deployment.yml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-ha
spec:
  serviceName: mysql-ha
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      initContainers:
        - name: init-mysql
          image: achoistic98/blog_mysqls:latest
          command:
          - bash
          - "-c"
          - |
            set -ex
            [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
            ordinal=${BASH_REMATCH[1]}
            if [[ $ordinal -eq 0 ]]; then
              cp /cfg/mysql-master.cnf /mnt/
              cp /cfg/init-master.sql /mnt/
            else
              cp /cfg/mysql-slave.cnf /mnt/
              cp /cfg/init-slave.sql /mnt/
            fi
          volumeMounts:
          - name: local-vol
            mountPath: /mnt/
      containers:
        - name: database
          image: mysql:latest
          ports:
            - containerPort: 3306
          volumeMounts:
          - name: local-vol
            mountPath: /etc/mysql/conf.d/
          - name: local-vol
            mountPath: /docker-entrypoint-initdb.d/
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
            - name: MYSQL_DATABASE
              value: "blogdb"
            - name: MYSQL_USER
              value: "slave"
            - name: MYSQL_PASSWORD
              value: "password"
      volumes:
        - name: database-volume
          hostPath:
            path: /volumes
  volumeClaimTemplates:
  - metadata:
      name: local-vol
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-storage"
      resources:
        requests:
          storage: 1Gi
```


## HAProxy 구성


MySQL의 Replication 설정은 끝났고, 로드밸런서인 HAProxy 설정을 추가로 해주어야한다. Write 요청이 오면 마스터 DB에만 요청이 전달되어야하고, Read 요청이 오면 모든 DB에 균일하게 요청이 전달되어야하기 때문이다.

총 두 개의 쿠버네티스 `Service` 오브젝트를 생성될 것인데, 하나는 마스터 DB인 mysql-ha-0으로만 라우팅하는 `Write` 요청만 받는 서비스이고, 다른 하나는 각 AZ마다 배포되는 HAProxy 파드로 라우팅하는 `Read` 요청만 받는 서비스이다.

그럼 read 요청을 받은 HAProxy 파드는 설정된 로드밸런싱 알고리즘(아래 예시에서는 라운드로빈)을 통해 모든 MySQL 파드가 골고루 쿼리를 요청하게 된다.

![image](https://res.craft.do/user/full/6deb5b3a-d995-5f97-e85b-e7c3c5f9702a/doc/3D38522E-E46A-4A72-808A-CC313028AA73/E8F43EAC-AE64-4E30-BB11-49C653308B6B_2/1M025kfIUCx5lDJBQlGOxgeJnF2GQ10dhCpycQJoW7gz/Image.png)

`haproxy-mysql.cfg`

```yaml
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

listen  stats
        bind  *:9000
        mode  http
        stats enable
        stats uri /haproxy/mysql
        stats auth admin:admin

listen  web
        bind  *:3306
        mode  tcp
        balance roundrobin
        option  tcp-check
        server  node1  mysql-ha-0.default.svc.cluster.local:3306  check
        server  node2  mysql-ha-1.default.svc.cluster.local:3306  check
        server  node3  mysql-ha-2.default.svc.cluster.local:3306  check
```

----

이렇게 복잡한 구성을 통해 Read/Write를 분리하면서 Multi-AZ에 배포된 여러 파드들 간의 데이터 일관성을 유지시키는 Replication을 구현하게 되었다.
