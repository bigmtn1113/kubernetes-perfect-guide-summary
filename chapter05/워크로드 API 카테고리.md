# 워크로드 API 카테고리

<br>

**각 워크로드 API 카테고리로 분류된 리소스와 관계**  
| **Tier 1** | **Tier 2** | **Tier 3** |
|:---:|:---:|:---:|
| Pod | ReplicationController ||
| Pod | ReplicaSet | Deployment |
| Pod | DaemonSet ||
| Pod | StatefullSet ||
| Pod | Job | CronJab |

<br>

## Pod
워크로드 리소스의 최소 단위  
한 개 이상의 컨테이너로 구성되며, 같은 파드에 포함된 컨테이너끼리는 네트워크적으로 격리되어 있지 않고 IP 주소를 공유  
즉, 파드 내 컨테이너들의 IP는 모두 동일하므로 서로 localhost 통신 가능

일반적으로 하나의 파드에 하나의 컨테이너를 가지지만, 추가로 서브 컨테이너 구성 가능  

### Pod Design Pattern
#### Sidecar Pattern
메인 컨테이너 외에 보조적인 기능을 추가하는 서브 컨테이너를 포함하는 패턴  
데이터 영역을 공유할 수 있다는 파드의 특징과 관련된 패턴

ex)  
- Config Updater -> Application
  - 특정 변경 사항을 감지하여 동적으로 설정을 변경하는 컨테이너
- Pod 외부 -> Git Syncer -> Application
  - 깃 저장소와 로컬 스토리지를 동기화하는 컨테이너
- Application -> S3 Transfer -> Pod 외부
  - 애플리케이션의 로그 파일을 오브젝트 스토리지로 전송하는 컨테이너

#### Ambassador Pattern
메인 컨테이너가 외부 시스템과 접속할 때 대리로 중계해 주는 서브 컨테이너를 포함한 패턴  
메인 컨테이너에서 목적지에 localhost를 지정하여 앰배서더 컨테이너로 접속 가능

메인 컨테이너는 항상 localhost를 지정하여 앰배서더 컨테이너로만 접속하고 앰배서더 컨테이너가 여러 목적지에 중계하여 연결하도록 구성하면 느슨한 결합 유지 가능.
즉, 단일 데이터베이스나 분산된 데이터베이스를 사용하는 서비스 환경 모두에서 특별한 변경 사항 없이 애플리케이션 사용 가능

ex)  
- Application -> CloudSQL Proxy -> Pod 외부
  - Application이 localhost를 통해 DB에 접속하면 CloudSQL로 전송하는 컨테이너
- Application -> Redis Proxy -> Pod 외부
  - Application이 localhost를 통해 DB에 접속하면 샤딩된 Redis에 전송하는 컨테이너

#### Adapter Pattern
서로 다른 데이터 형식을 변환해 주는 컨테이너를 포함하는 패턴

ex)  
- MySQL <-> MySQL Exporter <-> Pod 외부
  - MySQL 자체 형식 메트릭을 소프트웨어 형식에 맞게 변환해 주는 컨테이너
- Redis <-> Redis Exporter <-> Pod 외부
  - Redis 자체 형식 메트릭을 소프트웨어 형식에 맞게 변환해 주는 컨테이너

### Pod 생성
sample-pod.yaml  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-pod
sepc:
  containers:
  - name: nginx-container
    image: nginx:1.16
```
```bash
# 파드 생성
kubectl apply -f sample-pod.yaml
```

### 두 개의 컨테이너를 포함한 Pod 생성
sample-2pod.yaml  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-2pod
spec:
  containers:
  - name: nginx-container
    image: nginx:1.16
  - name: redis-container
    image: redis:3.2
```
```bash
# 두 개의 컨테이너를 포함한 파드 생성
kubectl apply -f sample-2pod.yaml

# 파드 확인(컨테이너 수가 2개인 것 확인)
kubectl get pods
```

### 컨테이너 로그인과 명령어 실행
```bash
# 컨테이너 로그인. 컨테이너에서 /bin/bash 실행
kubectl exec -it sample-pod -- /bin/bash

# 특정 컨테이너에서 ls 명령어 실행
kubectl exec -it sample-2pod -c nginx-container -- /bin/ls
```

### ENTRYPOINT/CMD 명령과 command/args
도커 이미지의 ENTRYPOINT = K8s yaml의 command  
도커 이미지의 CMD = K8s yaml의 args

sample-entrypoint.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-entrypoint
spec:
  containers:
  - name: nginx-container
    image: nginx:1.16
    command: ["/bin/sleep"]
    args: ["3600"]
```

### Pod명 제한
- 영문 소문자와 숫자
- '-' 또는 '.'
- 시작과 끝은 영문 소문자

### 호스트의 네트워크 구성을 사용한 Pod 기동
Pod에 할당된 IP 주소는 K8s 노드의 호스트 IP 주소와 범위가 달라 외부에서 볼 수 없는 IP 주소가 할당됨  
호스트의 네트워크를 사용하는 설정(spec.hostNetwork)을 활성화하면 호스트상에서 프로세스를 기동하는 것과 같은 네트워크 구성(IP 주소, DNS 설정, host 설정 등)으로 파드 기동 가능  
호스트 측의 네트워크 감시 또는 제어와 같은 특수한 애플리케이션 등에서만 사용 권장

sample-hostnetwork.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-hostnetwork
spec:
  containers:
  - name: nginx-container
    image: nginx:1.16
```
```bash
# 파드의 IP 주소 확인
kubectl get pod sample-hostnetwork -o wide

# 파드가 기동 중인 노드의 IP 주소 확인(Pod의 IP주소와 같은 것 확인)
kubectl get node <node명> -o wide

# 파드의 호스트명 확인
kubectl exec -it sample-hostnetwork -- hostname

# 파드의 DNS 설정 확인
kubectl exec -it sample-hostnatwork -- cat /etc/resolv.conf
```

### Pod DNS 설정과 Service Discovery
DNS 서버에 관한 설정은 spec.dnsPolicy에서 설정  
일반적으로 파드는 클러스터 내부 DNS를 사용하여 이름을 해석  
서비스 디스커버리나 클러스터 내부 로드 밸런싱에서 사용 목적

#### ClusterFirst(기본값)
클러스터 내부의 DNS 서버에 질의를 하고, 해석이 안 되는 도메인에 대해서는 업스트림 DNS 서버에 질의

sample-dnspolicy-clusterfirst.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-dnspolicy-clusterfirst
spec:
  containers:
  - name: nginx-container
    image: nginx:1.16
```
```bash
# 컨테이너 내부의 DNS 설정 파일 표시
kubectl exec -it sample-dnspolicy-clusterfirst -- cat /etc/resolv.conf

# 클러스터 내부의 DNS Service에 할당된 IP 주소(컨테이너 내부의 DNS 설정 파일에 표시된 nameserver IP와 동일) 확인
kubectl get service kube-dns -n kube-system
```

#### None
클러스터 외부 DNS 서버 참조  
정적으로 외부 DNS 서버만 설정하면 클러스태 내부 DNS를 사용한 서비스 디스커버리는 사용 불가

sample-dnspolicy-none.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-dnspolicy-none
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - example.com
    options:
    - name: ndots
      value: "5"
  containers:
  - name: nginx-container
    image: nginx:1.16
```
```bash
# 컨테이너 내부의 DNS 설정 파일 표시. 설정값 확인
kubectl exec -it sample-dnspolicy-none -- cat /etc/resolv.conf
```

#### Default
K8s 노드의 DNS 설정을 그대로 상속  
dnsPolicy의 기본값은 Default가 아닌 ClusterFirst  
클러스터 내부의 DNS를 사용한 서비스 디스커버리 사용 불가

sample-dnspolicy-default.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-dnspolicy-default
spec:
  dnsPolicy: Default
  containers:
  - name: nginx-container
    image: nginx:1.16
```
```bash
# 컨테이너 내부의 DNS 설정 파일 표시. K8s 노드의 /etc/resolv.conf의 내용과 동일
kubectl exec -it sample-dnspolicy-default -- cat /etc/resolv.conf
```

#### ClusterFirstWithHostNet
hostNetwork를 사용한 파드에 클러스터 내부의 DNS를 참조하고 싶을 경우에 설정  
hostNetwork를 사용하는 경우 K8s 노드의 네트워크 설정이 사용되므로 명시적으로 ClusterFirstWithHostNet을 지정

sample-dnspolicy-clusterfirstwithhostnet.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-dnspolicy-clusterfirstwithhostnet
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  containers:
  - name: nginx-container
    image: nginx:1.16
```
```bash
# 컨테이너 내부의 DNS 설정 파일 표시. ClusterFirst로 설정했을 때와 동일
kubectl exec -it sample-dnspolicy-clusterfirstwithhostnet -- cat /etc/resolv.conf
```

### /etc/hosts
리눅스 운영체제에서는 DNS로 호스트명을 해석하기 전에 /etc/hosts 파일로 정적 호스트명을 해석  
spec.hostAliases로 지정해 파드 내부 모든 컨테이너의 /etc/hosts 변경

sample-hostaliases.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-hostaliases
spec:
  containers:
  - name: nginx-container
    image: nginx:1.16
  hostAliases:
  - ip: 8.8.8.8
    hostnames:
    - google-dns
    - google-public-dns
```
```bash
# Entries added by HostAliases 내용 부분 확인
kubectl exec -it sample-hostaliases -- cat /etc/hosts
```

### 작업 디렉터리 설정
컨테이너에서 동작하는 애플리케이션의 작업 디렉터리는 도커 파일의 WORKDIR 명령 설정을 따르지만 spec.containers[].workingDir로 덮어쓰기 가능

sample-workingdir.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-workingdir
spec:
  containers:
  - name: nginx-container
    image: nginx:1.16
    workingDir: /tmp
```
```bash
# 출력 결과 /tmp 확인
kubectl exec -it sample-workingdir -- pwd
```

<br>

## ReplicaSet/ReplicationController
파드의 레플리카를 생성하고 지정한 파드 수를 유지하는 리소스  
레플리케이션 컨트롤러가 레플리카셋으로 이름이 변경되면서 일부 기능이 추가됐고 레플리카셋을 주로 사용하는 추세

### ReplicaSet 생성
sample-rs.yaml
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: sample-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```
```bash
# 세 개의 파드 기동 확인
kubectl get replicaset sample-rs -o wide

# 레플리카셋이 파드 관리에 사용되는 레이블을 지정하여 파드 목록 표시
# 세 개의 파드 기동 확인. 각각 다른 노드에 파드가 배치돼 노드 장애 발생 시 서비스에 미치는 영향 최소화
kubectl get pods -l app=sample-app -o wide
```

### Pod 정지와 self-healing
레플리카셋에서는 노드나 파드에 장애가 발생했을 때 지정한 파드 수를 유지하기 위해 다른 노드에서 파드를 기동시켜 주므로 장애 시 많은 영향을 받지 않음

```bash
# sample-rs에 기동 중인 파드 하나 종료
kubectl delete pod <pod명>

# 파드가 새로 생성되는 것 확인
kubectl get pods -o wide

# 레플리카셋 파드 수 증감 이력 확인
kubectl describe replicaset sample-rs
```

### ReplicaSet과 Label
레플리카셋은 K8s가 파드를 모니터링하여 파드 수를 조정  
모니터링은 특정 레이블을 가진 파드 수를 계산하는 형태

레플리카 수가 부족한 경우 매니페스트에 기술된 spec.template로 파드를 생성  
레플리카 수가 많을 경우 레이블이 일치하는 파드 중 하나를 삭제

### ReplicaSet과 Scaling
- 매니페스트를 수정하여 kubectl apply -f 명령어 실행
- kubectl scale 명령어를 사용하여 스케일 처리

#### kubectl apply
IaC를 구현하기 위해서라도 이 방법을 권장

```bash
# 레플리카 수를 3에서 4로 변경
sed -i -e 's|replicas: 3|replicas: 4|' sample-rs.yaml

kubectl apply -f sample-rs.yaml
kubectl get replicaset sample-rs
```

#### kubectl scale
ReplicationController/Deployment/StatefulSet/Job/CronJob에서도 사용 가능

```bash
kubectl scale replicaset sample-rs -- replicas 5
kubectl get replicaset sample-rs
```

<br>

## Deployment
여러 레플리카셋을 관리하여 롤링 업데이트나 롤백 등을 구현하는 리소스  
디플로이먼트가 레플리카셋을 관리하고 레플리카셋이 파드를 관리하는 관계

**디플로이먼트의 롤링 업데이트 구조**  
1. 신규 레플리카셋 생성
2. 신규 레플리카셋의 레플리카 수를 단계적으로 증가
3. 신규 레플리카셋의 레플리카 수를 단계적으로 감소
4. 2, 3을 반복
5. 이전 레플리카셋의 레플리카 수를 0으로 유지

신규 레플리카셋에 컨테이너가 기동되었는지와 헬스 체크는 통과했는지를 확인하면서 전환 작업이 진행  
레플리카셋의 이행 과정에서 파드 수에 대한 상세한 지정도 가능

※ 파드로만 배포한 경우엔 파드에 장애가 발생하면 자동으로 파드가 다시 생성 되지 않으며, 레플리카셋으로만 배포한 경우에는 롤링 업데이트 등의 기능을 사용할 수 없으므로 디플로이먼트 사용 권장

### Deployment 생성
sample-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```
```bash
# 업데이트 이력을 저장하는 옵션을 사용하여 디플로이먼트 기동
kubectl apply -f sample-deployment.yaml --record

# 이력 및 revision 정보 등 확인
kubectl get replicasets -o yaml | head

# 리소스들 확인
# 이름을 통해 디플로이먼트가 레플리카셋을, 레플리카셋이 파드를 생성한다는 것 확인 가능
kubectl get deployments,replicasets,pods
```

--record을 사용하여 어떤 명령어를 실행하고 업데이트했는지 이력을 저장하면 kubectl rollout에서 롤백 등을 실시할 때 참고 정보로 활용 가능  
이력은 각 레플리카셋의 metadata.annotations[kubernetes.io/change-cause]에 저장  
레플리카셋의 수정 버전 번호는 metadata.annotations[deployment.kubernetes.io/revision]에 저장

```bash
# 컨테이너 이미지 업데이트
kubectl set image deployment sample-deployment nginx-container=nginx:1.17 --record

# 디플로이먼트 업데이트 상태 확인
kubectl rollout status deployment sample-deployment

# 리소스들 확인
# 신규 및 이전 레플리카셋, 신규 파드 확인
kubectl get deployments,replicasets,pods
```

※ --record 옵션은 deprecated 됐으므로 사용 권장하지 않음. 대체 flag 나오기 전까진 다음 방법([디플로이먼트의 롤아웃 기록 확인](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#checking-rollout-history-of-a-deployment)) 사용 권장  
- 디플로이먼트에 kubectl annotate deployment/sample-deployment kubernetes.io/change-cause="update content"로 주석 작성
- 수동으로 리소스 매니페스트 편집

### Deployment 업데이트 조건
디플로이먼트는 변경이 발생하면 레플리카셋을 생성하는데 '생성된 파드의 내용 변경'이 조건  
spec.template에 변경이 있으면 생성된 파드의 설정이 변경되므로 레플리카셋을 신규로 생성하고 롤링 업데이트를 진행

매니페스트를 K8s에 등록한 후 레플리카셋의 정의를 보면 Pod Template Hash를 계산하고 이 값을 사용한 레이블로 관리

```bash
# pod-template-hash 값은 레플리카셋 이름에 있는 문자열과 동일
kubectl get replicaset <레플리카셋명> -o yaml
```

### Rollback
디플로이먼트가 생성한 기존 레플리카셋은 레플리카 수가 0인 상태로 남아 있으므로 레플리카 수를 변경시켜 다시 사용 가능

변경 이력을 확인할 때는 kubectl rollout history 명령어를 사용  
CHANGE-CAUSE 부분은 디플로이먼트를 생성할 때 --record 옵션을 사용하면 표시되지만, 아니라면 \<none\> 등으로 표시
  
```bash
# 변경 이력 확인
kubectl rollout history deployment sample-deployment
```

해당 수정 버전에 대한 상세한 정보를 가져오려면 --revision 옵션 지정
  
```bash
# 초기 상태의 디플로이먼트
kubectl rollout history deployment sample-deployment --revision 1

# 한 번 업데이트된 후의 디플로이먼트
# pod-template-hash, change-cause, image 등 달라진 점 확인
kubectl rollout history deployment sample-deployment --revision 2
```

롤백하려면 kubectl rollout undo를 사용  
명령어의 인수로 버전 번호 지정이 가능하며, 0으로 지정하거나 지정하지 않을 경우(0이 기본값)엔 바로 이전 버전으로 롤백
  
```bash
# 버전 번호를 지정하여 롤백
kubectl rollout undo deployment sample-deployment --to-revision 1

# 바로 이전 버전으로 롤백
kubectl rollout undo deployment sample-deployment

# 이전 레플리카셋에서 파드가 기동됨을 확인
kubectl get replicasets
```

CI/CD 파이프라인에서 롤백을 하는 경우 kubectl rollout 명령어보다 이전 매니페스트를 다시 kubectl apply 명령어로 실행하여 적용하는 것이 호환성 면에서 더 우수.
이때 spec.template를 같은 내용으로 되돌렸을 경우 pod-template-hash 값이 같으므로 kubectl rollout처럼 기존에 있었던 레플리카셋의 파드가 기동

### Deployment 변경 일시 중지
일반적으로 디플로이먼트를 업데이트하면 바로 적용되어 업데이트 처리가 실행  
즉시 적용을 일시 정지하고 싶을 때는 kubectl rollout pause 실행  
다시 시작할 때는 kubectl rollout resume 실행

```bash
kubectl rollout pause deployment sample-deployment

kubectl rollout resume deployment sample-deployment

# pause 상태에서 컨테이너 이미지 업데이트
kubectl set image deployment sample-deployment nginx-container=nginx:1.16

# 업데이트가 대기 중
kubectl rollout status deployment sample-deployment

# pause 상태에선 롤백 불가. error 발생
kubectl rollout undo deployment sample-deployment
```

### Deployment 업데이트 전략
#### Recreate
모든 파드를 한 번에 삭제하고 재생성하므로 다운타임이 발생하지만, 추가 리소스를 사용하지 않고 전환이 빠른 것이 장점  
기존 레플리카셋의 레플리카 수를 0으로 하고 리소스를 반환. 이후 신규 레플리카셋을 생성하고 레플리카 수 증가

sample-deployment-recreate.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-deployment-recreate
spec:
  strategy:
    type: Recreate
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```
```bash
# 컨테이너 이미지 업데이트
kubectl set image deployment sample-deployment-recreate nginx-container=nginx:1.17

# 레플리카셋 목록 표시(상태 변화 있을 시 계속 출력)
# 모든 파드가 존재하지 않는 기간 확인. 이때 일시적 서비스 중단 발생
kubectl get replicas --watch
```

#### RollingUpdate
maxUnavailable, maxSurge 설정 가능

- maxUnavailable
  - 업데이트 중에 동시에 정지 가능한 최대 파드 수
- maxSurge
  - 업데이트 중에 동시에 생성할 수 있는 최대 파드 수

두 개 동시에 0으로 설정 불가  
백분율로도 지정 가능. default 값은 각각 25%

sample-deployment-rollingupdate.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-deployment-rollingupdate
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```
```bash
# 컨테이너 이미지 업데이트
kubectl set image deployment sample-deployment-rollingupdate nginx-container=nginx:1.17

# 레플리카셋 목록 표시(상태 변화 있을 시 계속 출력)
# 신규 레플리카셋의 레플리카 수 증가 -> 기존 레플리카셋의 레플리카 수 감소. 이 과정을 반복
# maxUnavailable=1/maxSurge=0일 경우 반대로 진행
kubectl get replicas --watch
```

### 상세 업데이트 파라미터
Recreate나 RollingUpdate를 사용할 때 다른 파라미터를 설정하여 사용 가능

- minReadySeconds
  - 최소 대기 시간(초)
  - 파드가 Ready 상태가 된 다음부터 디플로이먼트 리소스에서 파드 기동이 완료되었다고 판단(다음 파드의 교체가 가능하다고 판단)하기까지의 최소 시간
- revisionHistoryLimit
  - 수정 버전 기록 제한
  - 디플로이먼트가 유지할 레플리카셋 수
  - 롤백이 가능한 이력 수
- progressDeadlineSeconds
  - 진행 기한 시간(초)
  - Recreate/RollingUpdate 처리 타임아웃 시간
  - 타임아웃 시간이 경과하면 자동으로 롤백

sample-deployment-params.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-deployment-params
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 2
  progressDeadlineSeconds: 3600
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```

### Deployment Scaling
디플로이먼트가 관리하는 레플리카셋의 레플리카 수는 레플리카셋과 같은 방법으로 스케일 가능

```bash
# 매니페스트 이용
sed -i -e 's|replicas: 3|replicas: 4|' sample-deployment.yaml
kubectl apply -f sample-deployment.yaml

# kubectl scale 명령어 이용
kubectl scale deployment sample-deployment --replicas=5
```

<br>

## DaemonSet
각 노드에 파드를 하나씩 배치하는 리소스  
레플리카 수를 지정할 수 없고 하나의 노드에 두 개의 파드 배치 불가  
단, 파드를 배치하고 싶지 않은 노드가 있을 경우 nodeSelector나 노드 안티어피니티를 사용한 스케줄링으로 예외 처리 가능

노드가 늘어나면 데몬셋의 파드는 자동으로 늘어난 노드에서 기동  
그렇기 때문에 데몬셋은 각 파드가 출력하는 로그를 호스트 단위로 수집하는 Fluentd나 각 파드 리소스 사용 현황 및 노드 상태를 모니터링하는 Datadog 등 모든 노드에서 반드시 동작해야 하는 프로세스를 위해 사용하는 것이 유용

### DaemonSet 생성
레플리카 수 등을 지정할 수 있는 항목이 존재하지 않음

sample-ds.yaml
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sample-ds
spec:
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```
```bash
# 각 노드에 하나씩 파드가 기동됨을 확인
kubectl get pods -o wide
```

### DaemonSet 업데이트 전략
#### OnDelete
데몬셋 매니페스트를 수정하여 이미지 등을 변경했더라도 기존 파드는 업데이트 되지 않음  
모니터링이나 로그 전송과 같은 용도로 많이 사용하므로 업데이트는 파드를 다시 생성할 때나 수동으로 임의의 시점에 진행

운영상의 이유로 정지하면 안 되거나 업데이트가 급하지 않은 파드의 경우 OnDelete 설정으로 사용해도 되지만, 이전 버전이 장기간 사용된다는 점 주의

sample-ds-ondelete.yaml
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: smaple-ds-ondelete
spec:
  updateStrategy:
    type: OnDelete
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```

#### RollingUpdate
디플로이먼트와 달리 하나의 노드에 동일한 파드를 여러 개 생성할 수 없으므로 maxSurge 설정 불가  
maxUnavailable만 지정하여 업데이트 수행. maxUnavailable 기본값은 1이며, 0으로 지정 불가

sample-ds-rollingupdate.yaml
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: smaple-ds-rollingupdate
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```

<br>

## StatefulSet
데이터베이스 등과 같은 스테이트풀한 워크로드에서 사용하기 위한 리소스

레플리카셋과의 주된 차이점
- 생성되는 파드명의 접미사는 숫자 인덱스가 부여된 것
  - ex) sample-statefulset-0, sample-statefulset-1, ...
  - 파드명이 변하지 않는다는 것이 특징
- 데이터를 영구적으로 저장하기 위한 구조
  - PersistentVolume을 사용하는 경우엔 파드 재기동 시 같은 디스크를 사용하여 재생성

### StatefulSet 생성
spec.volumeClainTemplates를 지정함으로써 각 파드에 영구 볼륨 클레임 설정 가능  
영구 볼륨 클레임을 사용하면 클러스터 외부의 네트워크를 통해 제공되는 영구 볼륨을 파드에 연결 가능하므로,  파드를 재기동할 때나 다른 노드로 이동할 때 같은 데이터를 보유한 상태로 컨테이너 생성  
영구 볼륨은 하나의 파드가 소유할 수도 있고, 영구 볼륨 종류에 따라 여러 파드에서 공유도 가능

sample-statefulset.yaml
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sample-statefulset
spec:
  serviceName: sample-statefulset
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
        volumeMounts:
        - name: www
          mountPath: /ver/share/nginx/html
        volumeClaimTemplates:
        - metadata:
            name: www
          spec:
            accessModes:
            - ReadWriteOnce
            resources:
              requests:
                storage: 1G
```
```bash
# 파드명 인덱싱 확인
kubectl get pod -o wide

# 스테이트풀셋에서 사용하고 있는 영구 볼륨 클레임 및 volume 확인
kubectl get persistentvolumeclaims

# 스테이트풀셋에서 사용하고 있는 영구 볼륨 확인
kubectl get persistentvolumes
```

### StatefulSet Scaling
레플리카셋과 같은 방법인 kubectl apply -f 또는 kubectl scale을 사용하여 스케일링  
레플리카셋이나 데몬셋 등과 달리 파드를 하나씩만 생성하고 삭제하므로 시간을 상대적으로 더 소모

#### Scale Out
- 인덱스가 가장 작은 것부터 파드를 하나씩 생성  
- 이전에 생성된 파드가 Ready 상태가 되고 나서 다음 파드 생성

#### Scale In
- 인덱스가 가장 큰 파드부터 삭제  

0번째 파드가 항상 가장 먼저 생성되고 가장 늦게 삭제  
-> 0번째 파드를 마스터 노드로 사용하는 이중화 구조 애플리케이션에 적합. 레플리카셋은 파드를 무작위로 삭제하므로 부적합

### StatefulSet Lifecycle
spec.podManagementPolicy를 Parallel로 설정하여 레플리카셋 등처럼 병렬로 동시에 파드 기동 가능  
기본값은 orderedReady

sample-statefulset-parallel.yaml
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sample-statefulset-parallel
spec:
  podManagementPolicy: Parallel
  serviceName: sample-statefulset-parallel
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-controller
        image: nginx:1.16
```

### StatefulSet 업데이트 전략
디플로이먼트나 데몬셋처럼 업데이트 전략 두 가지 중에서 선택 가능

#### OnDelete
스테이트풀셋에서 OnDelete는 영속성 영역을 가진 데이터베이스나 클러스터 등에서 많이 사용

sample-statefulset-ondelete.yaml
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sample-statefulset-ondelete
spec:
  updateStrategy:
    type: OnDelete
  serviceName: sample-statefulset-ondelete
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.16
```

#### RollingUpdate
디플로이먼트처럼 RollingUpdate를 사용한 스테이트풀셋 업데이트가 가능하나 차이점 존재  
- 스테이트풀셋에서는 영속성 데이터가 있으므로 디플로이먼트와 다르게 추가 파드를 생성해서 롤링 업데이트 불가  
- maxUnavailable 지정하여 롤링 업데이트 불가하므로 파드마다 Ready 상태인지 확인 후 업데이트 진행  
- spec.podManagementPolicy가 Parallel로 설정되어 있어도 병렬로 동시에 처리되지 않고 하나씩 업데이트 진행
- partition 설정 가능
  - 전체 파드 중 어떤 파드까지 업데이트할지 지정 가능
  - 전체에 영향을 주지 않고 부분적으로 업데이트 적용 및 확인할 수 있어 안전한 업데이트 가능
  - OnDelete와 달리 수동으로 재기동한 경우에도 partition 값보다 작은 인덱스를 가진 파드는 업데이트 되지 않음

sample-statefulset-rollingupdate.yaml
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sample-statefulset-rollingupdate
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 3
  serviceName: sample-statefulset-rollingupdate
  replicas: 5
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx-controller
        image: nginx:1.16
```
```bash
# 이미지 수정 후, 인덱스가 3이상인 파드만 업데이트 되는지 확인
kubectl get pods --watch

# partition 값을 3에서 1로 변경 후, 인덱스가 1이상인 파드만 업데이트 되는지 확인
kubectl get pods --watch
```

### 영구 볼륨 데이터 저장 확인
```bash
# 볼륨 마운트 여부 확인
kubectl exec -it sample-statefulset-0 -- df -h

# 영구 볼륨에 sample.html 생성
kubectl exec -it sample-statefulset-0 -- touch /usr/share/nginx/html/sample.html

# 테스트를 위해 파드 삭제
kubectl delete pod sample-statefulset-0

# 파드 정지, 복구 이후에도 파일이 저장되어 있는지 확인
kubectl exec -it sample-statefulset-0 -- ls /usr/share/nginx/html/sample.html
```

### StatefulSet 및 영구 볼륨 삭제
스테이트풀셋을 생성하면 파드에 영구 볼륨 클레임을 설정할 수 있어 영구 볼륨도 동시에 확보 가능  
이렇게 확보한 영구 볼륨은 스테이트풀셋이 삭제되어도 유지되므로 스테이트풀셋을 다시 생성하면 영구 볼륨 데이터도 보존되어 있는 상태

```bash
kubectl delete statefulset sample-statefulset

# 스테이트풀셋에 연결되는 영구 볼륨 클레임 확인
kubectl get persistentvolumeclaims

# 스테이트풀셋에 연결되는 영구 볼륨 확인
kubectl get persistentvolumes

kubectl apply -f statefulset.yaml

# 영구 볼륨 데이터 확인
kubectl exec -it sample-statefulset-0 -- ls /var/share/nginx/html/sample.html
```

스테이트풀셋을 삭제해도 영구 볼륨이 확보된 상태일 경우
- 관리형 서비스의 종량 과금 방식인 경우 볼륨을 그대로 유지하면 비용 발생
- 온프레미스 환경인 경우 디스크 공간이 확보된 상태로 유지

스테이트풀셋을 삭제한 후에는 확보된 영구 볼륨 해제 필요
```bash
kubectl delete persistentvolumeclaims www-sample-statefulset-{0..2}
```

<br>

## Job
N개의 병렬로 실행하면서 지정한 횟수의 컨테이너 실행(정상 종료)을 보장하는 리소스

### ReplicaSet과의 차이점과 Job의 용도
#### ReplicaSet과의 차이점
기동 중인 파드가 정지되는 것을 전제로 만들어졌는지 여부

#### Job 용도
배치 처리(일괄적으로 대량 건 처리)인 경우에 사용

ex)  
- 특정 서버와의 rsync
- S3 등의 오브젝트 스토리지에 파일 업로드

### Job 생성
레플리카셋처럼 레이블과 셀렉터를 명시적으로 지정 가능하지만, K8s가 유니크한 uuid를 자동으로 생성하므로 권장하지 않음

성공 횟수를 지정하는 completions, 병렬성을 지정하는 parallelism, 실패 허용 횟수를 지정하는 backoffLimit  
completions와 parallelism의 기본값은 1

sample-job.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sample-job
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 10
  template:
    spec:
      containers:
      - name: tools-container
        image: amsy810/tools:v2.0
        command: ["sleep"]
        args: ["60"]
      restartPolicy: Never
```
```bash
# 잡 목록 표시(생성 직후)
# 정상 종료한 파드 수(COMPLETIONS) 표시
kubectl get jobs

# 잡이 생성한 파드 확인
kubectl get pods --watch

# 잡 목록 표시(파드 실행 완료 후)
# 파드가 성공적으로 실행되고 종료 되었는지 확인. READY가 0/1, STATUS는 Completed
kubectl get jobs
```

### restartPolicy에 따른 동작 차이
#### Never
잡 용도로 생성한 파드 내부 프로세스가 정지되면 신규 파드를 생성하여 잡을 계속 실행

sample-job-never-restart.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sample-job-never-restart
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 10
  template:
    spec:
      containers:
      - name: tools-container
        image: amsy810/tools:v2.0
        command: ["sh", "-c"]
        args: ["$(sleep 3600)"]
      restartPolicy: Never
```
```bash
kubectl get pods

# 컨테이너상의 sleep 프로세스 정지
kubectl exec -it sample-job-never-restart-5vhgd -- sh -c 'kill -9 `pgrep sleep`'

# 생성된 파드 확인
kubectl get pods
```

#### OnFailure
잡 용도로 생성한 파드 내부 프로세스가 정지되면 RESTARTS 카운터가 증가하고, 사용했던 파드를 다시 사용하여 잡 실행  
파드가 기동하는 노드나 파드 IP 주소는 변경되지 않지만, 영구 볼륨이나 K8s 노드의 hostPath를 마운트하지 않은 경우라면 데이터 유실

sample-job-never-restart.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sample-job-onfailure-restart
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 10
  template:
    spec:
      containers:
      - name: tools-container
        image: amsy810/tools:v2.0
        command: ["sh", "-c"]
        args: ["$(sleep 3600)"]
      restartPolicy: OnFailure
```
```bash
kubectl get pods

# 컨테이너상의 sleep 프로세스 정지
kubectl exec -it sample-job-onfailure-restart-cgzr9 -- sh -c 'kill -9 `pgrep sleep`'

# 재시작 파드 확인
kubectl get pods
```

### 일정 기간 후 Job 삭제
잡은 종료 후에도 삭제되지 않고 계속 유지되므로 spec.ttlSecondsAfterFinished를 설정하여 잡이 종료한 후에 일정 기간(초) 경과 후 삭제되도록 설정 가능

sample-job-ttl.yaml  
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sample-job-ttl
spec:
  ttlSecondsAfterFinished: 30
  completions: 1
  parallelism: 1
  backoffLimit: 10
  template:
    spec:
      containers:
      - name: tools-container
        image: amsy810/tools:v2.0
        command: ["sleep"]
        args: ["60"]
      restartPolicy: Never
```
```bash
# Job 상태 모니터링
# 60초 후에 잡이 종료, 종료 이후 30초가 지나면 잡이 삭제 되는지 확인
# ADDED -> MODIFIED -> DELETED
kubectl get job sample-job-ttl --watch --output-watch-events
```

<br>

## CronJob
디플로이먼트와 레플리카셋의 관계와 비슷  
크론잡이 잡을 관리하고 잡이 파드를 관리하는 3계층 구조

### 크론잡 생성
spec.schedule에 Cron과 같은 형식으로 시간 지정 가능  
다음 예제는 '1분마다 50% 확률로 성공하는 한 번만 실행되는 잡'을 생성하는 크론잡 생성 예제

sample-cronjob.yaml
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sample-cronjob
spec:
  schedule: "*/1 * * * *
  concurrencyPolicy: Allow
  startingDeadlineSeconds: 30
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  suspend: false
  jobTemplate:
    spec:
      completions: 1
      parallelism: 1
      backoffLimit: 0
      template:
        spec:
          containers:
          - name: tools-container
            image: amsy810/random-exit:v2.0
          restartPolicy: Never
```
```bash
# 아직 잡이 생성되지 않아 ACTIVE한 잡이 존재하지 않은 상태
kubectl get cronjobs

# 스케줄 시간이 되지 않은 경우 잡이 존재하지 않음
kubectl get jobs

# 지정한 시간이 되면 크론잡이 잡을 생성. ACTIVE 확인
kubectl get cronjobs

# 크론잡이 생성한 잡 존재
kubectl get jobs
```

### CronJob 일시 정지
점검이나 어떤 이유로 잡 생성을 원하지 않을 경우에 suspend(일시 정지) 가능  
spec.suspend가 true로 설정되어 있으면 스케줄 대상에서 제외(기본값은 false)

```bash
# spec.suspend를 true로 설정하고 진행
# SUSPEND True 확인
kubectl get cronjobs
```
