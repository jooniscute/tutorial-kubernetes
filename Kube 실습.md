# Kube 실습

## 계정
user1@15.164.96.255
SKT20220614

## 설치
`curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube`

## 권한
`sudo mkdir -p /usr/local/bin/`
`sudo install minikube /usr/local/bin/`

## 실행
`minikube start --driver=docker`
`docker ps` 로 minikube container 하나 떠있는 거 확인
컨테이너 내부 접속하려면
`minikube ssh` (또는 `docker exec -t 컨테이너ID bash`): 로그인되는 계정의 차이가 있긴 함
`docker ps`로 다양한 컨테이너 확인 가능


## 상태 확인
`minikube status`

## 정지
`minikube stop`

## 삭제
`minikube delete`

## CLI 도구 (kubectl)
`minikube kubectl` ex) `minikube kubectl get nodes` 혹은
`snap install kubectl` 혹은
`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`
`curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"`
`echo "$(<kubectl.sha256) kubectl" | sha256sum --check`
`sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`

`echo 'source <(kubectl completion bash)' >> ~/.bashrc` kubectl 명령어 tap completion 되게 bashrc에 추가

## 확장 설치
목록 확인 : `minikube addons list`
추가 활성화 : `minikube addons enable 확장이름`
비활성화 : `minikube addons disable 확장이름`
dashboard, metrics-server 활성화 하기
대시보드 접속 : `minikube dashboard --url`

## 명령어
정보 조회 : `kubectl get`
생성 : `kubectl create`
생성(pod) : `kubectl run`
생성(service) : `kubectl expose`
로그 : `kubectl logs`
삭제 : `kubectl delete`
- 파드(pod) : pods = po
  - `kubectl get pods`
  - `kubectl get po`
- 리플리카셋(replicaset) : replicatsets = rs
  - `kubectl get replicasets`
  - `kubectl get rs`
- 디플로이먼트(deployment) : deployment = deploy
  - `kubectl get deployments`
  - `kubectl get deploy`
- 서비스(service) : service = svc
  - `kubectl get services`
  - `kubectl get svc`
- 네임스페이스(namespace) : namespace = ns
- 다수의 서비스 조회
  - `kubectl get deploy,rc,pods`

## Namespace
`kubectl get pod namespace`
`kubectl get pod -n kube-system`
`kubectl create namespace hello`
`kubectl get pods -n hello`
`kubectl delete namespace hello`

## 로그 확인
`kubectl get events`

## 클러스터 정보 확인
`kubectl cluster-info`

## Hello World
`kubectl create deploy hello-minikube --image=k8s.gcr.io/echoserver:1.10`: 기본 서버 배포 (-> deployment, replica set, pod 같이 생성)
`kubectl get deploy`
`kubectl get replicasets`
`kubectl get pods`
`kubectl get all`

안에 서버가 생성되어 있는데 (pod 안에 서비스컨테이너의 8080포트)
거기에 접속할 수 있게 노출시켜줌
`kubectl expose deployment hello-minikube --type=NodePort --port=8080`
서비스를 생성하여 node 포트로 외부로 노출시킴 -> node의 internal-ip 에, 포워딩된 포트로 접속 시 서비스 접근 가능: 192.168.49.2:31613
internal ip 확인: `kubectl get nodes -o wide`
노출된 서비스 포트 확인: `kubectl get svc`
즉, pod(8080)에 바로 접근이 불가하니 pod가 있는 node의 port와 연결하여 node를 통해 접속하는 것
node를 통하지 않고 로컬 호스트에서 서비스로 바로 접속하려면 포트 포워딩: `kubectl port-forward svc/hello-minikube 8080:8080`

만들어져 있는 pod 안에서 명령 실행
`kubectl get pod`
(Name: hello-minikube-7bfc84c94b-pb62q)
`kubectl exec hello-minikube-7bfc84c94b-pb62q -- curl localhost:8080` (pod 안에 직접 접속해서 하는 것과 같음)

배포한 서비스 삭제
`kubectl delete deploy,svc hello-minikube`
`kubectl get pods -w`

## Hello NodeJS
`kubectl run nodejs --image=lovehyun/express-app:1.0 --port=8000` pod 형태로 생성 (deploy 랑 다르게 replicaset, deployment 안생김)
`kubectl logs nodejs` 로그 확인
`kubectl exec nodejs -- curl 127.0.0.1:8000 --silent` 동작 확인
`kubectl expose pod/nodejs --type=NodePort --name nodejs-svc` 서비스 생성 (NodePort or LoadBalancer)
`curl 192.168.49.2:31880` 접근 확인
`kubectl port-forward svc/nodejs-svc 8000:8000` 로컬에서 잘 떴나 확인해보기~
`kubectl delete svc nodejs-svc` svc 삭제
`kubectl delete pod nodejs` pod 삭제

## Hello NodeJS2
`kubectl create deployment nodejs --image=lovehyun/express-app:1.1 --port=8000` deploy 형태로 생성
`kubectl expose deploy/nodejs --type=NodePort --name nodejs-svc` 서비스 생성
`curl 192.168.49.2:30221` 접근 확인
이제 확장시킬거임 (scaling - replicaset 컨트롤)
`kubectl get rs` rs 미리 확인
`kubectl scale deploy/nodejs --replicas=3`  rs 3개로 확장
`curl 192.168.49.2:30221` 반복 접근시마다 pod가 바뀌는 걸 확인 (내부 load balancing으로 인함)
`kubectl delete pod nodejs-7b8d7b4bc5-n4bk7` pod 하나 삭제해보기 -> 삭제되면 새로운 pod가 자동으로 하나 또 생김 (self-healing by replicaset)
`kubectl scale deploy/nodejs --replicas=2`  rs 2개로 바꾸면 한개가 알아서 삭제됨
리소스 업데이트 해볼거임
`kubectl describe svc/nodejs-svc` svc 정보 확인
`kubectl describe deploy/nodejs` deploy 정보 확인
`kubectl set image deployment/nodejs express-app=lovehyun/express-app:1.2 --record` pod 새 이미지로 만들고 기존꺼 삭제 (하나씩 진행됨) -> RollingUpdate. replicaset이 하나 더 생기면서 동작
`kubectl rollout history deployment/nodejs` history 살펴보기
`kubectl rollout undo deployment/nodejs` 이전으로 롤백
`kubectl rollout undo deployment/nodejs --to-revision=2` 특정 수정사항으로 롤백
`kubectl delete deployment/nodejs` 삭제
`kubectl delete svc nodejs-svc`

## Hello NodeJS3
부하가 걸렸을 때의 auto-scaling
`kubectl top pods -n kube-system` htop 같은 명령어 -> metric-server 필요
새로 시작
`minikube delete` 삭제
`minikube start --extra-config=kubelet.housekeeping-interval=10s` 시작
`minikube dashboard --url` 대시보드 켜기
`minikube addons enable metrics-server` metrics-server 켜기
`kubectl create deployment nodejs --image=lovehyun/express-app:latest --port=8000` 앱 띄우기
`kubectl expose deployment/nodejs --type=NodePort --name nodejs-svc` 서비스
`kubectl top pods` pod의 메모리, CPU 확인
`kubectl autoscale deployment/nodejs --cpu-percent=70 --min=1 --max=5` CPU 70% 이상 점유 시 pod를 1-> 5까지 자동으로 조정할 수 있게 설정 (hpa 생성)
`kubectl set resources deployment/nodejs --requests=cpu=50m --limits=cpu=50m,memory=64Mi` CPU 사용량 제한 (점유율 빠르게 오를 수 있게끔)
`while true; do curl 192.168.49.2:31936 --silent >/dev/null; done &`
`while true; do curl 192.168.49.2:31936 --silent >/dev/null; done &`
`while true; do curl 192.168.49.2:31936 --silent >/dev/null; done &` CPU 부하를 줘보기
`kill %1 %2 %3` 로 job kill (`jobs`으로 확인)
`kubectl get hpa` hpa 정보를 통해 CPU 증가 확인
70% 넘으면 replicaset이 pod를 늘려가면서 부하를 조정시킴
`kubectl delete hpa nodejs`
`kubectl delete deploy/nodejs`
`kubectl delete svc/nodejs-svc`

## kubectl 설정 파일
기본 양식
  apiVersion: v1
  kind: Pod
  metadata:
  spec:

./examples/01.pod/1.nginx_pod_yaml
`kubectl create -f 1.nginx_pod.yaml` pod 생성 (yaml 내 kind: Pod)
`kubectl apply -f 1.nginx_pod.yaml` 이렇게도 생성 가능. yaml 수정시 apply 하면 몇가지는 자동으로 반영됨
`kubectl delete -f 1.nginx_pod.yaml` pod 삭제

./examples/04.deployment/1.pods.yaml
`kubectl get all -l app=nginx` label 로 원하는 애들 필터링 가능

## Ingress
./examples/05.ingress/1.ingress.yaml
`minikube addons enable ingress` Ingress 활성화

./examples/05.ingress/2.ingress-dashboard.yaml
`kubectl apply -f 2.ingress-dashboard.yaml`
`kubectl get svc -n kubernetes-dashboard`
kubernetes-dashboard라는 외부에서 연결된 서비스를
'my-dashboard.com' 도메인을 통해 접속
`kubectl get ingress -n kubernetes-dashboard` 로 인그레스 확인
호스트 내에 해당 도메인이 없으므로, 추가 설정을 해줘야함
`sudo vim /etc/hosts` 192.168.49.2 my-dashboard.com 추가
`curl my-dashboard.com` 접속 확인
`kubectl delete -f 2.ingress-dashboard.yaml`