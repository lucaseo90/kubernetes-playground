# 실습

Kubernetes에 애플리케이션 배포

* Kubernetes에 배포할 Node.js 애플리케이션 작성
* 애플리케이션을 이미지로 패키징
* 패키징한 이미지를 컨테이너로 실행
* 이미지를 Docker Hub에 등록
  > 실제로 푸시하지는 않음

## 선행 작업

* Docker 설치
* Kubernetes 설치
  * 초보자의 경우 minikube 또는 Docker Desktop 추천
  > Docker Desktop으로 진행
  Docker version 20.10.12, build e91ed57
  Kubernetes version v1.22.5
* kubectl 설치

## Docker

### `$docker run <image>:<tag>`를 실행할 때 일어나는 일

1. Docker는 실행하려고 하는 이미지가 로컬에 존재하는지 확인
2. 존재하지 않는 경우 Docker Hub Registry에서 로드
3. 이미지를 이용해서 컨테이너 실행
     * `<tag>`를 생략하면 항상 `latest` 태그를 참조

## app.js

* 포트 8080을 사용하여 HTTP 서버를 시작
* 서버를 호출하면 HTTP Status Code 200과 `You're hit <hostname>` 메시지로 응답

### 빌드

```bash
$ docker build -t kubia .
```

1. Docker client가 디렉토리에 있는 Dockerfile 과 애플리케이션 파일을 Docker daemon에 로드
2. Docker daemon은 빌드에 필요한 이미지를 로드
3. 새로운 이미지를 빌드

```bash
$ docker images
REPOSITORY  TAG     IMAGE ID       CREATED         SIZE
kubia       latest  840f696332b6   5 days ago      905MB
```

### 실행

```bash
$ docker run --name kubia-container -p 8080:8080 -d kubia
cb8675fa2e1ff394413ae28a5e6f8a00b71aa8fb1bcac26084c59c403b44ffa0
```

* --name: 컨테이너 이름 지정
* -p: 컨테이너가 사용하는 포트를 호스트 포트와 매핑
* -d: 백그라운드로 실행

### 확인

```bash
$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                    NAMES
cb8675fa2e1f   kubia          "node app.js"            2 minutes ago   Up 2 minutes   0.0.0.0:8080->8080/tcp   kubia-container
```

#### curl을 이용해 컨테이너가 사용중인 8080포트에 HTTP GET 요청 호출

```bash
$ curl localhost:8080
You've hit kubia-cjr87
```

#### 컨테이너 정보 확인

```bash
$ docker inspect kubia-container
[
    {
        "Id": "cb8675fa2e1ff394413ae28a5e6f8a00b71aa8fb1bcac26084c59c403b44ffa0",
        "Created": "2022-02-11T12:04:37.6022826Z",
        "Path": "node",
        "Args": [
            "app.js"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 64438,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2022-02-11T12:04:38.7352117Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:840f696332b68f46a02183b2833515c3c1679e1156ef5ef547b0cc3d492a89c3",
        # ... JSON 포맷으로 컨테이너 정보를 출력
```

#### 컨테이너 접근

실행 중인 컨테이너의 bash에 접속

```bash
$ docker exec -it kubia-container bash
root@cb8675fa2e1f:/#
```

* -i: shell에 명령을 입력하기 위한 STDIN이 열려있는지 확인
* -t: pseudo terminal (TTY) 할당

#### 프로세스 확인

```bash
root@cb8675fa2e1f:/# ps aux | grep app.js
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.2 318036 32992 ?        Ssl  12:04   0:00 node app.js
root        21  0.0  0.0   3088   892 pts/0    S+   13:30   0:00 grep app.js

$ ps aux | grep app.js
USER               PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
abc              63227   0.0  0.0 34271284    948 s000  S+   10:33PM   0:00.01 grep app.js
```

* 호스트에서도 실행중인 app.js 애플리케이션을 확인 가능
* 컨테이너는 호스트 머신과 완전히 격리된 프로세스 트리를 갖고 PID를 독립적으로 소유(1 vs. 63227). 

#### 컨테이너 중지 및 제거

```bash
$ docker stop kubia-container
kubia-container

$ docker rm kubia-container
kubia-container
```

* `stop` 명령만 수행한 경우, `$docker ps -a`로 컨테이너를 확인 가능

### 이미지 등록

빌드한 이미지를 로컬 머신이 아니더라도 로드하여 사용하려면 이미지를 레지스트리에 등록해야 한다.  
대중적으로 사용하는 레지스트리는 Docker Hub 인데, 이미지를 푸시하기 위해서 Docker Hub의 태그 규칙에 따라 태그 지정이 필요하다.

#### 태그 설정

```bash
$ docker tag kubia ${USER_NAME}/kubia
```

* 이미지에 대한 태그를 수정하는 것이 아닌, 동일한 이미지에 대해 태그를 추가
* `${USER_NAME}` 은 Docker Hub 계정

#### 이미지 푸시

```bash
$ docker push ${USER_NAME}/kubia
```

이후로는 로컬에서 이미지를 빌드할 필요 없이, 빌드된 이미지를 레지스트리로부터 받아 컨테이너로 실행할 수 있다.

#### 푸시한 이미지를 이용한 컨테이너 실행

```bash
$ docker run -p 8080:8080 -d ${USER_NAME}/kubia
```

## Kubernetes

### Kubernetes 클러스터 구동 확인

클러스터가 작동중인지 확인

```bash
$ kubectl cluster-info
Kubernetes master is running at https://kubernetes.docker.internal:6443
CoreDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

* 더 자세한 정보를 확인하기 위해서는 `dump` 옵션 추가

클러스터의 노드 목록 확인

```bash
$ kubectl get nodes
NAME             STATUS   ROLES                  AGE   VERSION
docker-desktop   Ready    control-plane,master   16d   v1.21.5
```

노드 정보 확인

```bash
$ kubectl describe nodes ${NODE_NAME, e.g., docker-desktop}
```

* 노드 상태
* CPU 및 메모리
* 시스템 정보
* 노드에서 실행 중인 컨테이너

### 애플리케이션 배포

```bash
$ kubectl apply -f kubia.yaml
replicationcontroller/kubia configured

# deprecated 
$ kubectl run kubia --image=luksa/kubia --port=8080 --generator=run/v1
```

* deprecated 명령어로 배포하면 `--generator=run/v1`이 동작하지 않고 `pod/kubia created` 메시지 출력

### 확인

#### Pod 확인

```bash
$ kubectl get pods 
```

Event 항목을 확인함으로써 Pod가 배포되는 과정을 확인 가능

```bash
$ kubectl describe pod
...
Events:
  Type    Reason     Age        From                     Message
  ----    ------     ----       ----                     -------
  Normal  Scheduled  <unknown>                           Successfully assigned default/kubia-ljbpm to docker-desktop
  Normal  Pulling    2m13s      kubelet, docker-desktop  Pulling image "luksa/kubia:latest"
```

#### Pod 액세스

Pod에 액세스하기 위해서는, Pod에 존재하는 고유한 IP에 접근하도록 구성해야 한다. 이때 Loadbalancer 타입의 특수 서비스를 활용해 연결한다.

```bash
$ kubectl expose rc kubia --type=LoadBalancer --name kubia-http
```

```bash
$ kubectl get services
```

```bash
$ curl localhost:8080
```

#### Pod 복제본 수 증가

```bash
$ kubectl get replicationcontroller # or kubectl get rc

$ kubectl scale rc kubia --replicas=3

$ kubectl get replicationcontroller

$ kubectl get pods

$ curl localhost:8080
```

여기까지 실습을 진행하면 Kubernets의 다음 컴포넌트를 확인할 수 있다.

* Pod
  * 컨테이너가 실행되는 구성 요소
  * 다수의 컨테이너 포함 가능
* Service
  * Pod는 컨테이너가 일시적으로 동작하는 공간으로 Pod가 새로 실행되면 IP 주소가 변경 될 수 있음
  * 변경되는 IP 주소 문제를 보완하여 Pod가 호스트에 단일 고정 IP 및 Port로 노출될 수 있도록 함
* Replication Controller
  * Kubernetes 관리자는 `직접` Pod를 생성하지 않고, Replication Controller가 Pod 를 생성
  * Pod가 사라지면 누락된 Pod를 대체할 새 Pod를 생성
