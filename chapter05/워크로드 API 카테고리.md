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
