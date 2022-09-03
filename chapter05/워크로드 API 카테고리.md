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
