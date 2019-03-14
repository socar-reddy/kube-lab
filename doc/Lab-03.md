# 파드 띄우기 실습

본 랩에서는 파드를 정의하는 매니패스트를 직접 작성해봅니다.

## 1. 클러스터 올리기

### 1-1. project zone 셋팅
```
gcloud config set compute/zone asia-northeast1-c
```

### 1-2. 클러스터 생성
```
gcloud container clusters create lab03 --num-nodes 3 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

### 1-3. alias 설정
```shell
alias k=kubectl
```

## 2. 파드 매니페스트 생성 / 실행

- 본랩에서는 vi를 사용합니다.

### 2-1. nginx 컨테이터 이미지를 Pod으로 띄워보기

참고 : [쿠버네티스 홈페이지 참고 링크](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#step-two-add-a-nodeselector-field-to-your-pod-configuration)

##### 매니페스트 파일 (YAML) 작성

쿠버네티스 오브젝트의 스펙을 기술해 봅니다.

```shell
vi pod1.yaml
```

pod2-1.yaml
```yaml
apiVersion: v1 # 이 오브젝트를 생성하기 위해 사용하고 있는 쿠버네티스 API 버전이 어떤 것인지
kind: Pod # 어떤 종류의 오브젝트를 생성하고자 하는지
metadata:  # 이름, 네임스페이스 를 포함하여 오브젝트를 유일하게 구분지어 줄 데이터
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
```

##### Pod 띄우기
- create 명령어를 안쓰는 이유
- 모디파이 할떄는 create안먹고 apply만 먹는다
- 야믈파일을 수정한 이후에는 apply를 통해서 수정한 사항 반영
```shell
kubectl apply -f pod.yaml
kubectl get po
kubectl get po --show-labels
kubectl get po -l env=test
# 라벨 골라서 get
```

##### 세부사항 조회

- 파드 기본정보
- 실행중인 컨테이너 정보
- 이벤트 정보

```sh
kubectl describe po 파드이름(nginx)
```

##### 파드 삭제
```sh
kubectl delete po 파드이름
kebectl delete po -f pod1.yaml # 이렇게도 지울 수 있다.
```

파드가 삭제될 때는 30초의 유예기간을 갖습니다. 더이상 요청을 받지 않습니다.
또한, 해당 파드의 컨테이너 데이터도 삭제됩니다.

### 2-2. 특정 노드에서 띄워보기

pod2-2.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```
- imagePullPolicy 현재 땡겨온 이미지가 있으면 있는걸 계속 재활용 한다는 의미
- 디스크 타입이 ssd라는 라벨딱지가 붙어있는 노드에만 배포하겠다는 뜻

그런데 파드 생성이 안된다... 계속 펜딩(Pending)상태..
```
kubectl describe po nginx
로 살펴보니 Events 상태를 보니 노드셀렉터와 매치할 수 있는 노드가 없다...
```

```sh
kubectl get node --show-labels
kubectl get node --show-labels | grep ssd
없다...
```

노드에 라벨 붙히기
```sh
kubectl label node 이름(여기서는두번째노드로해보자) disktype=ssd
```
다시 pod이 제대로 running하기 시작했다.

### 2-3. 네임스페이스 지정해보기
- 지금까지는 pod를 만들때 기본 네임스페이스에서 생성했다.
- **네임스페이스** : 클러스터 내부의 객체를 관리.
  - 각 네임스페이스는 객체 집합을 담고 있는 폴더로 생각할 수 있습니다.
  - 명시를 안하면, default namespace

네임스페이스 만들기
```
kubectl create ns kub(네임스페이스명)
kubectl get ns
```

pod2-3.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: kuba
  name: nginx-kuba
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
```

아래에 2개의 명령어로 나오는 pod은 다르다
다른 네임스페이스에 있기 때문에
```
kubectl get po
kubectl get po -n kuba
```

pod제거
```
kubectl delete po nginx
kubectl delete po nginx-kuba -n kuba
```


## 3. 포트 포워딩

참고 : https://github.com/kubernetes-up-and-running/kuard

### 3-1. kuard-pod.yaml
- 쿠버네티스 업 앤 러닝 데모?
- 이번에는 gcr.io레포에서 이미지를 땡겨온다
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  containers:
  - name: kuard
    image: gcr.io/kuar-demo/kuard-amd64:1
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP
```

### 3-2. 파드 실행

```shell
kubectl apply -f kuard-pod.yaml
```

### 3-3. 포트 포워딩

https://i.stack.imgur.com/wmKgd.png 이런 느낌?
```shell
kubectl port-forward kuard 8080:8080
```

확인
```shell
curl localhost:8080
```
http://localhost:8080 확인


### 3-4. 로그 확인

##### 로그 조회

실행중인 인스턴스에서 로그를 다운로드합니다.

```sh
kubectl logs kuard -f(pod 이름)

```

## 4. 하나의 파드, 두개의 컨테이너

https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/#creating-a-pod-that-runs-two-containers

two-container-pod.yaml
- 엔진엑스는 웹서버만
- 데비안 컨테이너는 소스만
- emptyDir : 하드가 날라가면 해당 볼륨도 날라가는 속성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:

  restartPolicy: Never

  volumes:
  - name: shared-data
    emptyDir: {}

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

팟 생성
```
kubectl apply -f two-container-pod.yaml
kubectl get po
kubectl describe po two-containers
```
- 엔진엑스는 running 
- 데비안 컨테이너는 실행되고 터미네이트 된 상태
- html을 만들고 바로 종료하는 것
  - 리눅스는 1번 pid가 종료하면 리눅스 수명이 끝 근데 그 1번 pid가 html을 만드는 거였음


팟속 컨테이너 쉘 접속
- pod 속의 컨테이너에 접속
- c는 컨테이너 지정 옵션
```sh
kubectl exec -it two-containers -c nginx-container -- /bin/bash

root@two-containers:/ apt-get update
root@two-containers:/ apt-get install -y curl procps
root@two-containers:/ ps aux

root@two-containers:/ curl localhost
```

포트포워딩 후 로컬에서 접속
- &는 백그라운드 실행
```
kubectl port-forward two-containers 8080:80 &
curl localhost:8080

jobs
kill %1
```

파드 상태를 확인해봅시다.

```sh
kubectl get po

NAME             READY     STATUS      RESTARTS   AGE
two-containers   1/2       Completed   0          20m
```

```
kubectl delete po two-containers
```

## 5. 클러스터 삭제

```
gcloud container clusters delete lab03
```

## 6. 미니큐브 띄워보기

https://kubernetes.io/ko/docs/tutorials/hello-minikube/
