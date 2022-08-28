# API 리소스와 kubectl

<br/>

## 쿠버네티스 기초
- **쿠버네티스**
  - 쿠버네티스 마스터 + 쿠버네티스 노드
- **쿠버네티스 마스터**
  - API 엔드포인트 제공, 컨테이너 스케줄링, 컨테이너 스케일링 등 담당
- **쿠버네티스 노드**
  - 도커 호스트에 해당하며 실제로 컨테이너를 기동시키는 노드

쿠버네티스 클러스터를 관리하려면 CLI 도구인 **kubectl과** **매니페스트 파일**을 사용하여 쿠버네티스 마스터에 '리소스' 등록 필요

매니페스트 파일은 가독성을 고려하여 YAML 형식으로 작성하는 것이 일반적이며 확장자(.yaml, .yml, .json)로 구분  
kubectl은 매니페스트 파일 정보를 바탕으로 쿠버네티스 마스터가 가진 API에 요청을 보내어 쿠버네티스를 관리  

쿠버네티스 API는 일반적으로 RESTful API로 구현  
kubectl을 사용하지 않고 각종 프로그램 언어의 RESTful API 라이브러리나 curl 등의 명령어 도구를 사용하여 직접 API를 호출함으로써 쿠버네티스 관리 가능

<br/>

## 쿠버네티스와 리소스
쿠버네티스가 취급하는 리소스 목록 다섯 가지 카테고리

- **Workload API**
  - 클러스터 위에서 컨테이너를 기동하기 위해 사용되는 리소스
- **Service API**
  - 컨테이너 서비스 디스커버리와 클러스터 외부에서도 접속이 가능한 엔드포인트 등을 제공하는 리소스
- **Config and Storage API**
  - 설정과 기밀 데이터를 컨테이너에 담거나 영구 볼륨을 제공하는 리소스
- **Cluster API**
  - 클러스터 자체 동작을 정의하는 리소스
- **메타데이터 API**
  - 클러스터 내부의 다른 리소스 동작을 제어하기 위한 리소스

<br/>

## 네임스페이스로 가상적인 클러스터 분리
쿠버네티스에는 네임스페이스라는 가상적인 쿠버네티스 클러스터 분리 기능이 존재  
완전한 분리 개념이 아니므로 용도는 제한되지만, 하나의 쿠버네티스 클러스터를 여러 팀에서 사용하거나 서비스/스테이징/개발 환경으로 구분하는 경우 사용 가능

기본 설정에서 생성되는 네 가지 네임 스페이스
- **kube-system**
  - 쿠버네티스 클러스터 구성 요소와 애드온이 배포될 네임스페이스
- **kube-public**
  - 모든 사용자가 사용할 수 있는 컨피그맵 등을 배치하는 네임스페이스
- **kube-node-lease**
  - 노드 하트비트 정보가 저장된 네임스페이스
- **default**
  - 기본 네임스페이스

관리형 서비스나 구축 도구로 구축된 경우 대부분의 쿠버네티스 클러스터는 RBAC(Role-Based Access Control)이 기본값으로 활성화된 상태  
RBAC은 클러스터 조작에 대한 권한을 네임스페이스별로 구분할 수 있고 네트워크 정책과 함께 사용하여 네임스페이스 간의 통신을 제어할 수 있는 구조

ex)  
클러스터 관리자 - default, kube-system, prd  
애플리케이션 개발자 - prd

<br/>

## kubectl
수동으로 쿠버네티스를 조작하는 경우엔 CLI 도구인 kubectl을 사용하는 것이 일반적

### 인증 정보와 컨텍스트(config)
kubectl이 쿠버네티스 마스터와 통신할 때는 접속 대상의 서버 및 인증 정보 등이 필요  
kubectl은 kubeconfig(~/.kube/config)에 쓰여 있는 정보를 사용하여 접속

kubeconfig에서 구체적으로 설정이 이루어지는 부분은 clusters/users/contexts 세 가지  
이 세 가지 설정 항목은 모두 배열로 되어 있어 여러 대상 등록 가능
- clusters
  - 접속 대상 클러스터 정보
- users
  - 인증 정보
- contexts
  - cluster와 user, namespace

context를 전환하여 여러 환경을 여러 권한으로 조작 가능  
```bash
# context 목록 표시
kubectl config get-contexts

# context 전환
kubectl config use-context prd-admin

# 현재 context 표시
kubectl config current-context
```

### 매니페스트와 리소스 생성/삭제/갱신
```bash
# 리소스 생성
kubectl create -f sample-pod.yaml

# Pod 목록 표시
kubectl get pods

# 리소스 삭제
kubectl delete -f sample-pod.yaml

# 특정 리소스만 삭제
kubectl delete pod sample-pod

# 특정 리소스 종류 모두 삭제
kubectl delete pod --all

# 리소스 삭제(삭제 완료 대기)
kubectl delete -f sample-pod.yaml --wait

# 리소스 즉시 강제 삭제
kubectl delete -f sample-pod.yaml --grace-period 0 --force

# 리소스 업데이트
kubectl apply -f sample-pod.yaml
```

### Pod 재기동
Deployment 등의 리소스와 연결되어 있는 모든 파드는 재기동이 가능하나, 리소스와 연결되어 있지 않은 단독 파드는 재기동 불가

```bash
# 리소스 생성
kubectl apply -f sample-deployment.yaml
kubectl apply -f sample-pod.yaml

# Deployment 리소스의 모든 파드 재기동
kubectl rollout restart deployment sample-deployment

# Pod 재기동 실패
kubectl rollout restart pod sample-pod
```

### 매니페스트 파일 설계
한 개의 매니페스트 파일에 여러 리소스를 정의하거나 여러 매니페스트 파일을 동시에 적용 가능

#### 하나의 매니페스트 파일에 여러 리소스 정의
실행 순서를 정확히 지켜야 하거나 리소스 간의 결합도를 높이고 싶을 때 사용  
공통으로 사용하는 설정 파일(ConfigMap)이나 패스워드(Secret) 등은 별도 매니페스트로 분리하는 것이 용이

```yaml
apiVersion: apps/v1
kind: Deployment
...
---
apiVersion: v1
kind: Service
...
```

#### 여러 매니페스트 파일을 동시에 적용
디렉터리 안에 적용하고 싶은 여러 매니페스트 파일을 배치해 두고 kubectl apply 명령어를 실행할 때 그 디렉터리를 지정  
파일명순으로 적용되므로 순서 제어를 원할 시 파일명 앞에 인덱스 번호 지정과 같은 방법 필요  
-R 옵션을 사용하면 재귀적으로 디렉터리 안에 존재하는 매니페스트 파일 적용 가능

```bash
# 디렉터리를 지정하여 리소스 생성
kubectl apply -f ./dir

# 지정한 디렉터리 내의 파일을 재귀적으로 적용
kubectl apply -f ./dir -R
```

#### 매니페스트 파일 설계 방침
- **시스템 전체를 한 개의 디렉터리로 통합하는 패턴**
  - 규모가 크지 않을 경우엔 시스템 전체를 구성하기 위한 모든 마이크로서비스의 매니페스트 파일을 하나의 디렉터리로 통합
  - ex) ./whole-system
- **시스템 전체를 특정 서브 시스템으로 분리하는 패턴**
  - 거대한 시스템인 경우엔 분리가 가능하다면 서브 시스템이나 부서별로 디렉터리를 나눠서 사용
  - ex) ./whole-system/subsystem-A
- **마이크로서비스별로 디렉터리를 나누는 패턴**
  - 가독성이 높아진다는 장점이 있지만, 적용 순서 제어가 어렵다는 단점 존재
  - ex) ./microservice-A, ./microservice-B, ./microservice-C

### Annotation과 Label
#### Annotation
metadata.annotations로 설정할 수 있는 메타데이터  
리소스에 대한 메모 같은 것

#### Label
metadata.labels에 설정할 수 있는 메타데이터  
리소스를 구분하기 위한 정보 같은 것  
실수로 레이블을 지정하면 레이블 값이 충돌하여 예상하지 못한 문제 발생 가능성 존재  
- ReplicaSet - 레이블에 부여된 파드 수를 계산하여 레플리카 수 관리
  - 문제: 기존에 생성된 Pod 정지
- LoadBalancer - 레이블 기준으로 목적지 파드 결정
  - 문제: 다른 Pod에 트래픽 전송

### diff
로컬 매니페스트와 쿠버네티스 등록 정보 비교 출력  
'실제로 쿠버네티스 클러스터에 등록된 정보'와 '로컬에 있는 매니페스트' 내용의 차이 파악 필요
```bash
# 클러스터 등록 정보와 매니페스트의 차이점 확인
kubectl diff -f sample-pod.yaml
```

### api-resources
모든 리소스 종류 표시
```bash
kubectl api-resources
```

### get
리소스 정보 가져오기
```bash
# Pod 목록 상세히 표시
kubectl get pods -o wide

# YAML 형식으로 Pod의 상세 정보 목록 출력
kubectl get pods -o yaml

# JSON Path로 Pod명 표시
kubectl get pods sample-pod -o jsonpath="{.metadata.name}"

# 생성된 거의 모든 종류의 리소스 표시
kubectl get all

# 리소스 상태 변화를 출력
kubectl get pods --watch
```

### describe
리소스 상세 정보 가져오기
```bash
# Pod 상세 정보 표시
kubectl describe pod sample-pod
```

### top
실제 리소스 사용량 확인

kubectl describe 명령어로 확인할 수 있는 리소스 사용량은 쿠버네티스가 Pod에 확보한 값을 표시  
실제 Pod 내부의 컨테이너가 사용하고 있는 리소스 사용량은 kubectl top 명령어를 사용하여 확인 가능  
리소스 사용량은 Node와 Pod 단위
```bash
# Node 리소스 사용량 확인
kubectl top node

# Pod 리소스 사용량 확인
kubectl top pod

# Container 리소스 사용량 확인
kubectl top pod --containers
```

### exec
컨테이너에서 명령어 실행

가상 터미널을 생성(-t)하고, 표준 입출력을 패스스루(-i)하면서 /bin/sh를 기동하면 마치 컨테이너에 SSH로 로그인한 것과 같은 상태로 접근 가능
```bash
# Pod 내부의 컨테이너에서 /bin/bash 실행
kubectl exec -it sample-pod -- /bin/bash

# 여러 컨테이너에 존재하는 Pod의 특정 컨테이너에서 /bin/ls 실행
kubectl exec -it sample-pod -c nginx-container -- /bin/ls
```

### port-forward
로컬 머신에서 Pod로 포트 포워딩

```bash
kubectl apply -f sample-pod.yml

# localhost:8888에서 Pod의 80/TCP 포트로 전송
kubectl port-forward sample-pod 8888:80

# 다른 터미널에서 접속 확인
curl -I localhost:8888
```

Pod명이 아닌 Deployment 리소스나 Service 리소스에 연결되는 Pod에도 포트 포워딩 가능  
포트 포워딩에 의한 통신은 여러 Pod에 분산하여 전송되는 것이 아니라 하나의 Pod로만 전송
따라서 kubectl port-forward 실행 중에 통신할 수 있는 Pod는 항상 같은 하나의 Pod
```bash
# sample-deployment에 연결된 Pod 중 하나의 Pod로 포트 포워딩
kubectl port-forward deployment/sample-deployment 8888:80

# sample-service에 연결된 Pod 중 하나의 Pod로 포트 포워딩
kubectl port-forward service/sample-service 8888:80
```

<hr>

## 참고
- **Kubernetes API** - https://kubernetes.io/docs/reference/kubernetes-api/
