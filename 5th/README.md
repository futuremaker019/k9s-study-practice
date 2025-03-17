# 미니 프로젝트 실습



<br>

# PV (Persistent Volume) 와 StatefuleSet

### statful vs Stateless

- Stateless란 네트워크에서는 서버가 클라이언트의 정보를 저장하지 않는것을 말한다. 상태가 없는, 상태가 중요하지 않은 앱을 의미함
- stateful은 반대로 상태를 저장한다는 의미. 쿠버네티스에서는 기본적으로 Stateless로 생각하지만, 상태를 가져야하는 특수한 경우가 있음

### PV 와 PVC 개념

- `PV (Psersistent Volume)`: 상태를 가지기 위한 물리적인 공간
- `PVC (Persistent Volume Claim)`: Pod에서 PV를 쓸수 있도록 하는 논리적인 단위로 정의한것이 PVC이다.
  - PV가 있는 상태에서, PVC를 만들면 적합한 PV가 PVC와 연결되어 사용 할 수 있는 형태가 된다.
  - AWS나 GCP에서는 PVC만 만들어도 PV를 알아서 만들게 할수 있는데, 이를 `동적 프로비저닝`이라고 한다.

### StatefuleSet 이란

- Statefule이라는 오브젝트를 통해 PVC 와 연결된 Pod를 만들수 있다.
- 기존에 상태가 중요하지 않은 Pod는 Node의 스토리지 중 일부를 빌려와 사용함
- 상태가 중요한 Pod(StatefulSet)으로 만들어진 Pod는 PVC와 연결하여 사용하게 된다.

> pod가 종료되면 해당 pod에 연결되있던 PVC는 생성된 다른 pod에 다시 연결된다.

<br>

# Stateful 실습

### StorageClass 오브젝트 생성

`StorageClass` 란 PV(C)를 이용하기 위해 쿠버네티스에서 구분하는 스토리지의 종류를 나타내는 오브젝트이다.

스토리지 클래스 오브젝트를 설정하면, `AWS`의 소토리지 서비스인 EBS와 연결되어 PVC를 생성 할 때 자동으로 물리적인 스토리지를 빌려와 연결시켜주게 된다.

> `GCP`를 사용중이라면 이미 자동으로 `standard`라는 이름의 StorageClass가 생성이 되있기 떄문에 별로로 등록할 필요가 없음
> StorageClass 확인 경로: 스토리지 -> 탭에 스토리지 클래스 -> standard 확인

```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer             # PVC를 생성하면 바로 PV와 연결할건지 설정하는 Immediate(바로 연결) 도 존재함
reclaimPolicy: Delete                               # 
parameters:
  csi.storage.k8s.io/fstype: ext4
  type: gp3
allowedTopologies:                                  # 리전을 선택하여 한국의 물리 스토리지를 사용할 수 있게 만들어준다.
  - matchLabelExpressions:
    - key: topology.ebs.csi.aws.com/zone
      values:
        - ap-northeast-2a
        - ap-northeast-2b
        - ap-northeast-2c
        - ap-northeast-2d
```

StatefulSet 생성

> `GCP` 의 경우에는 storageClassName을 `standard` 로 설정한다.

```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: default
spec:
  serviceName: "mysql"
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "sparta"
            - name: MYSQL_DATABASE
              value: "sparta"
          volumeMounts:
            - name: mysql-persistent-storage        # PVC와 연결하는 옵션
              mountPath: /var/lib/mysql             # 설정한 경로의 파일들은 PVC와 연결하여 저장한다는 의미의 옵션
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "500m"
              memory: "1Gi"
  volumeClaimTemplates:
    - metadata:
        name: mysql-persistent-storage
      spec:
        accessModes:                                # 하나의 Pod에 하나의 PVC를 사용할 수 있도록하는 옵션
          - ReadWriteOnce
        storageClassName: "ebs-sc"                  # "ebs-sc" StorageClass라는 PVC와 연결할 수 있도록 함
        resources:
          requests:
            storage: 20Gi
```

Headless Service 생성

> Headless Service란
> Service에 직접 IP 를 할당하지 않는 것을 Headless Service라고 한다.
> 이러한 Service를 만드는 이유는 StatefulSet으로 만들어진 Pod의 경우에는 일반적으로 랜덤하게 부하 분산이 이루어지는 것이 아닌 특정 Pod와 접속해야하기 때문이다. (로드 밸런싱이 되면 안됨)
> ex) Storage나 Database와 같은 특정해야하는 Pod와 직접 연결할 때
> <br>
> 이러한 경우 Pod에 DNS(도메인 주소)를 붙여서 쉽게 접속할 수 있도록 한다.
> <br>
> 설정방법: `clusterIP`를 `Node` 으로 설정함으로써 Headless Service로 설정함


```yml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: default
  labels:
    app: mysql
spec:
  clusterIP: None                       # ClustorIP를 Node으로 하면 IP를 할당받지 않는 Headless Service가 된다.
  selector:
    app: mysql
  ports:
    - port: 3306
      name: mysql
```

명령어는 statefulset과 headless-service 만 apply 함

쿠버네티스에서 pod의 mysql에 접속해보자. 명령어는 아래와 같다.

```
kubectl exec -it [pod명] -- bash
ex) kubectl exec -it mysql-0 -- bash
-- 는 kubectl 명렁의 옵션이 끝났음을 나타나는 식별자이다. 

# pod 내부에 다른 컨테이너가 존재한다면 -c 를 붙이고 컨테이너명을 넣어주어야 한다.
kubectl exec -it [pod명] -c [container명] -- bash
```

<br>

# Job/Cron Job 개념

### Batch 프로그램이란

Batch 프로그램이란, 여러 데이터를 일괄적으로 가공하는 작업을 뜻한다. 이러한 프로그램 실행단위를 `Job`이라고 함

> 쿠버네티스에서는 Job 오브젝트를 통해 만들어진 Pod는 컨테이너들이 모두 실행이 종료되면, 그대로 사라져버리는 속성을 가짐

일정주기마다 오브젝트를 실행하기 위해 쿠버네티스에서는 `CronJob`이라는 오브젝트가 존재함

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: eightzero-delete-job
  namespace: default
spec:
  templates:
    spec:
      containers:
      - name: mysql-client
        image: mysql:8.0
        env:
        - name: MYSQL_HOST
          valueFrom:
            secretKeyRef:
              name: mini-project-secret
              key: DB_HOST
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mini-project-secret
              key: DB_USER
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mini-project-secret
              key: DB_PASS
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mini-project-secret
              key: DB_DATABASE
        command: ["sh", "-c", "mysql -h$MYSQL_HOST -u$MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE -e \"DELETE FROM users WHERE user_code = '00000000';\""]     # Job 실생시 실행하는 명령어
      restartPolicy: Never      # Job을 실패하면 다시 실행되지 않도록 함
```

job을 실행하자. 실행 명령어는 동일함

```
kubectl apply -f job.yml
```

실행 후 pod 를 확인하면 상태값이 `ContainerCreating`에서 `Complete`으로 변한것을 확인할 수 있음

직접 mysql 에 접속 후 삭제됬는지 확인

```yml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: eightzero-delete-cronjob
  namespace: default
spec:
  schedule: "*/1 * * * *"               # cron 생성
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mysql-client
            image: mysql:8.0
            env:
            - name: MYSQL_HOST
              valueFrom:
                secretKeyRef:
                  name: mini-project-secret
                  key: DB_HOST
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mini-project-secret
                  key: DB_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mini-project-secret
                  key: DB_PASS
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mini-project-secret
                  key: DB_DATABASE
            command: ["sh", "-c", "mysql -h$MYSQL_HOST -u$MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE -e \"DELETE FROM users WHERE user_code = '00000000';\""]
          restartPolicy: Never
  successfulJobsHistoryLimit: 3         # 성공한 job history를 3개만 생성하도록 함.  
  failedJobsHistoryLimit: 3             # 실패한 job history를 3개만 생성하도록 함.
                                        # history에 제한을 두지 않으면 cronjob이 1분주기로 돌며 반복적인 history를 생성하게 됨
```

cronjob 실행하기

```
kubectl apply -f cronjob.yml
```

1분마다 실행할 수 있도록 cron 을 설정했으므로 cronjob은 종료되지 않고 계속 실행됨

종료 명령어는 동일하다. 그냥 delete

```
kubectl delete cronjob [cronjob 명]
```

<br>

# 미니프로젝트 구축

mini-project.yml 실행함

> ConfigMap, Service, Deployment, Ingress가 하나의 yml 파일로 묶어서 실행함

> GCP로 실행할시 Ingress에 ingressClassName은 지우자. -> address 안뜸

이전에 올려둔 prometheus와 grafana를 이용하여 테스트함

미니 프로젝트에 미리 만들어진 부하테스트를 실행하여 부하를 어떻게 분리할지 고민 
- Deployment의 replicas를 3 -> 10으로 증가시켜 pod를 늘리는 방향으로 진행
- Grafana의 대시보드에서 Compute Resources/Workload 에 들어가 부하 확인
- pod가 새로 생성되면서 부하가 분산됨을 확인


다음장에서는 pod를 직접 늘려주는것이 아닌 k8s가 자동으로 늘려주는 방식을 진행함

<br>
