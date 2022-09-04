# 워크로드 API 카테고리

<br/>

**각 워크로드 API 카테고리로 분류된 리소스와 관계**  
| **Tier 1** | **Tier 2** | **Tier 3** |
|:---:|:---:|:---:|
| Pod | ReplicationController ||
| Pod | ReplicaSet | Deployment |
| Pod | DaemonSet ||
| Pod | StatefullSet ||
| Pod | Job | CronJab |

<br/>

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

<br/>

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

<br/>

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

※ --record 옵션은 deprecated될 예정

```bash
# 컨테이너 이미지 업데이트
kubectl set image deployment sample-deployment nginx-container=nginx:1.17 --record

# 디플로이먼트 업데이트 상태 확인
kubectl rollout status deployment sample-deployment

# 리소스들 확인
# 신규 및 이전 레플리카셋, 신규 파드 확인
kubectl get deployments,replicasets,pods
```

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
