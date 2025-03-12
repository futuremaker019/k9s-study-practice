docker image 생성 방법 실습

GCP로 실습함

docker 이미지를 GCP의 Artifact Registry에 push

필요한 이미지를 Artifact Registry에서 pull 하여 사용할 수 있도록 함

docker 이미지 빌드 명령어

```
docker build -t [빌드할 이미지명]
ex) docker build -t asia-northeast3-docker.pkg.dev/ornate-axiom-452414-s4/kube-study-registry/configmap-sample:1.0.0 .

gcp의 Artifact Registry 의 경로를 이용하여 docker 이미지를 Dockerfile에서 build 한다.

-t 는 --tag 의 축약형이다.

build 시 마지막의 Dockerfile의 위치를 명시해 주는거 잊지말자. `.` 을 찍든 위치를 표시하든 하자
```

docker 이미지 올리기

```
docker push [레지스트리 명]
ex) docker push asia-northeast3-docker.pkg.dev/ornate-axiom-452414-s4/kube-study-registry/configmap-sample:1.0.0
```

pod가 동작중인지 포트 포워딩으로 확인 하는 명령어

```
kubectl port-forward [pod 명] [host port:pod port]
ex) kubectl port-forward sample-pod 8000:80

# namespac가 존재하는 pod는 namespace 명령어를 추가하자
kubectl port-forward [pod 명] --namespace [namespace 명] [host port]:[pod port]
```
