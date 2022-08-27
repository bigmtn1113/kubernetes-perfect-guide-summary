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

<hr>

## 참고
- **Kubernetes API** - https://kubernetes.io/docs/reference/kubernetes-api/
