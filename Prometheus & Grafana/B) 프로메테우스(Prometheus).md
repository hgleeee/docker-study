# 프로메테우스(Prometheus)
## 구성 요소
### 프로메테우스 서버(prometheus-server)
- 프로메테우스의 주요 기능을 수행하는 요소로 3가지 역할을 맡는다.
  1) 노드 익스포터 외 여러 대상에서 공개된 메트릭을 수집해 오는 수집기
  2) 수집한 시계열 메트릭 데이터를 저장하는 시계열 데이터베이스
  3) 저장된 데이터를 질의하거나 수집 대상의 상태를 확인할 수 있는 웹 UI


### 노드 익스포터(node-exporter)
- 노드의 시스템 메트릭 정보를 HTTP로 공개하는 역할
1. 설치된 노드에서 특정 파일들을 읽는다.
2. 이를 프로메테우스 서버가 수집할 수 있는 메트리 서버로 변환
3. 노드 익스포터에서 HTTP 서버로 공개
4. 공개된 내용을 프로메테우스 서버에서 수집해 간다.

### 쿠버 스테이트 메트릭(kube-state-metrics)
- API 서버로 쿠버네티스 클러스터의 여러 메트릭 데이터를 수집한 후, 이를 프로메테우스 서버가 수집할 수 있는 메트릭 데이터로 변환해 공개하는 역할
- 프로메테우스가 쿠버네티스 클러스터의 여러 정보를 손쉽게 획득할 수 있는 것은 이것 덕분이다.

### 얼럿매니저(alertmanager)
- 프로메테우스에 경보 규칙을 설정하고, 경보 이벤트가 발생하면 설정된 경보 메시지를 대상에 전달
- 프로메테우스에 설치하면 프로메테우스 서버에 주기적으로 경보를 보낼 대상을 감시해 시스템을 안정적으로 운영할 수 있다.

### 푸시게이트웨이(pushgateway)
- 배치와 스케줄 작업 시 수행되는 일회성 작업들의 상태를 저장하고 모아서 프로메테우스가 주기적으로 가져갈 수 있도록 공개한다.

## 프로메테우스 설치 (by Helm)
- NFS 디렉터리를 만들고, 만든 NFS 디렉터리를 쿠버네티스 환경에서 사용할 수 있도록 PV와 PVC로 구성한다.
- 접근 ID(사용자 ID, 그룹 ID)는 1000번으로 설정한다.

### 1) 헬름과 MetalLB 구성 확인 및 NFS 디렉터리 생성 후 PV와 PVC로 구성
```bash
[root@m-k8s ~]# ~/_Book_k8sInfra/ch6/6.2.1/prometheus-server-preconfig.sh
[Step 1/4] Task [Check helm status]
[Step 1/4] ok
[Step 2/4] Task [Check MetalLB status]
[Step 2/4] ok
[Step 3/4] Task [Create NFS directory for prometheus-server]
/nfs_shared/prometheus/server created
[Step 3/4] Successfully completed
[Step 4/4] Task [Create PV,PVC for prometheus-server]
persistentvolume/prometheus-server created
persistentvolumeclaim/prometheus-server created
[Step 4/4] Successfully completed
```

### 2) 모니터링에 필요한 3가지 프로메테우스 오브젝트(프로메테우스 서버, 노드 익스포터, 쿠버 스테이트 메트릭) 설치
```bash
[root@m-k8s ~]# cat  ~/_Book_k8sInfra/ch6/6.2.1/prometheus-install.sh
01 #!/usr/bin/env bash
02 helm install prometheus edu/prometheus \
03 --set pushgateway.enabled=false \
04 --set alertmanager.enabled=false \
05 --set nodeExporter.tolerations[0].key=node-role.kubernetes.io/master \
06 --set nodeExporter.tolerations[0].effect=NoSchedule \
07 --set nodeExporter.tolerations[0].operator=Exists \
08 --set server.persistentVolume.existingClaim="prometheus-server" \
09 --set server.securityContext.runAsGroup=1000 \
10 --set server.securityContext.runAsUser=1000 \
11 --set server.service.type="LoadBalancer" \
12 --set server.extraFlags[0]="storage.tsdb.no-lockfile"
```

- 2번째 줄 : edu 차트 저장소의 prometheus 차트를 사용해 prometheus 릴리스 설치
- 3번째 줄 : 푸시게이트웨이를 사용하지 않도록 설정
- 4번째 줄 : 얼럿매니저를 사용하지 않도록 설정
- 5~7번째 줄 : 테인트가 설정된 노드의 설정을 무시하는 톨러레이션 설정 (마스터 노드에 노드 익스포터 배포, 프로메테우스가 마스터 노드의 메트릭 데이터 수집 가능)
- 8번째 줄 : PVC 동적 프로비저닝을 사용할 수 없는 가상 머신 환경이기 때문에 이미 만들어놓은 prometheus-server라는 이름의 PVC를 사용하도록 설정
- 9~10번째 줄 : 컨테이너에 할당할 사용자 ID와 그룹 ID를 1000번으로 설정
- 11번째 줄 : MetalLB로부터 외부 IP를 할당받기 위해 타입을 LoadBalancer로 설정
- 12번째 줄 : lockfile이 생성되지 않게 설정 (프로메테우스 설정을 변경할 때 lockfile이 있으면 변경 작업을 실패할 수 있음)

### 3) 설치 확인
```bash
[root@m-k8s ~]# kubectl get pods --selector=app=prometheus
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-kube-state-metrics-7bc49db5c5-79tdb   1/1     Running   0          74s
prometheus-node-exporter-759px                   1/1     Running   0          74s
prometheus-node-exporter-gb55v                   1/1     Running   0          74s
prometheus-node-exporter-rclkf                   1/1     Running   0          74s
prometheus-node-exporter-wn5zq                   1/1     Running   0          74s
prometheus-server-6d77896bb4-d5q6q               2/2     Running   0          74s
```
- 노드 익스포터가 여러 개인 이유는 노드마다 메트릭을 수집하기 위해 데몬셋으로 설치했기 때문이다.

### 4) 프로메테우스 서버에서 제공하는 웹 UI로 접속하기 위한 IP 주소 확인
```bash
[root@m-k8s ~]# kubectl get service prometheus-server
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
prometheus-server   LoadBalancer   10.110.36.151   192.168.1.12   80:32379/TCP   2m26s
```
- 프로메테우스 서비스의 IP 주소가 192.168.1.12인 것을 확인할 수 있다.

### 5) 웹 브라우저 접속 및 웹 UI 정상 동작 확인



## 웹 UI 분석
### Graph
#### 1) 쿼리 입력기
- 프로메테우스가 적재한 메트릭 데이터를 조회할 수 있는 표현식을 입력하는 곳
- 이 때 사용하는 표현식은 PromQL(Prometheus Query Language)이라는 프로메테우스에서 제공하는 쿼리 언어이다.
- 다른 RDBMS 처럼 쿼리문을 작성해 효과적으로 필요한 메트릭을 추출한다. (시계열 데이터베이스를 사용하므로)

#### 2) Execute
- 쿼리 입력기에 입력한 PromQL을 실행하는 버튼
- 해당 버튼을 누르면 PromQL 표현식에 맞는 메트릭 데이터를 화면에 보여준다.

#### 3) Graph
- PromQL로 프로메테우스가 적재한 메트릭 데이터를 확인할 때 시각적으로 표현해주는 옵션

#### 4) Console
- PromQL로 추출된 메트릭 데이터를 보여주는 기본 옵션 (표 형식)

#### 5) Add Graph
- 쿼리 입력기를 하나 더 추가해 또 다른 메트릭을 확인하는 버튼


### Alert
- 경보 화면에서는 현 프로메테우스 서버에 등록된 경보 규칙과 경보 발생 여부를 확인할 수 있다.

### Status
- Runtime & Build Information : 런타임 관련 정보, 버전을 나타내는 빌드 정보 등 여러 정보를 확인할 수 있다.
- Command-Line Flags : 프로메테우스 서버가 실행될 때 인자로 입력받았던 값을 보여준다.
- Configuration : 프로메테우스 서버가 구동될 때 설정된 값을 표시한다.
- Rules : 프로메테우스 서버에 등록된 다양한 규칙을 확인할 수 있다.
- Targets : 프로메테우스 서버가 수집해오는 대상의 상태를 확인할 수 있다.
- Service Discovery : 프로메테우스 서버가 디스커버리 방식으로 수집한 대상들에 대한 정보를 요약해 보여준다.

## 서비스 디스커버리로 수집 대상 가져오기
- 프로메테우스는 수집 대상을 자동으로 인식하고 필요한 정보를 가져온다.
- 정보를 수집하기 위해 일반적으로 에이전트를 설치해야 하지만, 쿠버네티스는 사용자가 에이전트에 추가로 입력할 필요 X

### 순서
> 프로메테우스 서버와 API 서버가 주기적으로 데이터를 주고받아 수집 대상을 업데이트하고, 수집 대상에서 공개되는 메트릭을 자동 수집한다.
1. 프로메테우스 서버는 컨피그맵에 기록된 내용을 바탕으로 대상을 읽어온다.
2. 읽어온 대상에 대한 메트릭을 가져오기 위해 API 서버에 정보를 요청한다.
3. 요청을 통해 알아온 경로로 메트릭 데이터를 수집한다.

### 대상에 따른 서비스 디스커버리 방법
- cAdvisor : 쿠버네티스 API 서버에 직접 연결되어 메트릭을 수집
- 익스포터(에이전트) : API 서버가 경로를 알려주으로써 메트릭을 수집








