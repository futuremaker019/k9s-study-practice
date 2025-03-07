# Pod

스크립트

```
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:1.27.0
    ports:
    - containerPort: 80
```


## 명령어

작성한 스크립트로 pod 실행

```
kubectl apply -f sample-pod.yml
```

현재 가동되고 있는 Pods의 목록을 보기

```vi
 kubectl get pods
```

현재 가동중인 pod를 하나 삭제

```
kubectl delete pod [pod 명]
ex) kubectl delete pod sample-pod
```

<br>

# ReplicaSet

ReplicaSet
    - Pod 를 관리하는 쿠버네티스 오브젝트가 `ReplicaSet` 이라고 함
    - Pod 하나가 다운되면 다운된 Pod를 폐기하고 동일한 새로운 Pod를 생성함

ReplocaSet 실행 스크립트

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: sample-replicaset
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.0
        ports:
        - containerPort: 80
```

## 명령어

작성한 replica 스크립트 실행

```
kubectl apply -f sample-replicaset.yml
```

현재 실행중인 replicaset 목록 보기

```
kubectl get replicasets
```

실습으로 replicaset에 의해 실행된 pod를 하나 지우면 지운 즉시 다시 시행된다.

<br>

# Node

쿠버네티스 클러스터에 실제로 존재하는 서버 컴퓨터이다.

노드는 실제 물리적인 컴퓨터로써 논리적인 단위인 pod들을 프로세스와 같이 실행한다. 


