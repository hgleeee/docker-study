# 젠킨스 설치 및 설정
## 젠킨스 설치 by Helm
### 1) 레지스트리 구성
```bash
[root@m-k8s ~]# docker ps -f name=registry
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                             NAMES
83e015b59f7d        registry:2          "/entrypoint.sh /etc…"   13 seconds ago      Up 12 seconds       5000/tcp, 0.0.0.0:8443->443/tcp   registry
```
- 젠킨스로 지속적 통합을 진행하는 과정에서 컨테이너 이미지를 레지스트리에 푸시하는 단계이다.

### 2) NFS 디렉터리를 /nfs_shared/jenkins에 만듦
```bash
[root@m-k8s ~]# ~/_Book_k8sInfra/ch5/5.3.1/nfs-exporter.sh jenkins
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.
```

### 3) 생성된 디렉터리의 사용자 ID와 그룹 ID 확인
```bash
[root@m-k8s ~]# ls -n /nfs_shared
total 0
drwxr-xr-x. 2 →0 0← 6 Jun 22 21:32 jenkins
```
- 0번은 root 사용자에 속해 있다는 의미이다.

### 4) NFS 디렉터리에 대한 접근 ID를 1000번으로 설정
```bash
[root@m-k8s ~]# chown 1000:1000 /nfs_shared/jenkins
[root@m-k8s ~]# ls -n /nfs_shared
total 0
drwxr-xr-x. 2 1000 1000 6 Jun 22 21:32 jenkins
```
- 젠킨스를 헬름 차트로 설치하면 젠킨스의 여러 설정 파일과 구성 파일들이 PVC를 통해 PV에 파일로 저장된다.
- 이 때 PV에 적절한 접근 ID를 부여하지 않으면 PVC를 사용해 파일을 읽고 쓰는 기능에 문제가 발생할 수 있다.
- 이러한 문제를 방지하기 위해 젠킨스 PV가 사용할 NFS 디렉터리에 대한 접근 ID를 1000번으로 설정한다.

### 5) PV와 PVC 구성 후 BOUND 상태인지 확인
```bash
[root@m-k8s ~]# kubectl apply -f ~/_Book_k8sInfra/ch5/5.3.1/jenkins-volume.yaml
persistentvolume/jenkins created
persistentvolumeclaim/jenkins created
[root@m-k8s ~]# kubectl get pv jenkins
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
jenkins   10Gi       RWX            Retain           Bound    default/jenkins                           10s
[root@m-k8s ~]# kubectl get pvc jenkins
NAME      STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
jenkins   Bound    jenkins   10Gi       RWX                           18s
```

### 6) 젠킨스 설치
```bash
[root@m-k8s ~]# ~/_Book_k8sInfra/ch5/5.3.1/jenkins-install.sh
NAME: jenkins
LAST DEPLOYED: Thu Jun 22 21:46:48 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
[생략]
```
- NAME : 설치된 젠킨스의 릴리스 이름
- NAMESPACE : 젠킨스가 배포된 네임스페이스
- REVISION : 배포된 릴리스가 몇 번째로 배포된 것인지 알려준다. 해당 젠킨스는 처음 설치된 것임을 알 수 있다.
- NOTES : 설치와 관련된 안내 사항을 몇 가지 표시한다.

### 7) 디플로이먼트 정상 배포 확인
```bash
[root@m-k8s ~]# kubectl get deployment
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
jenkins   1/1     1            1           2m47s
```

### 8) 서비스 상태 확인
```bash
[root@m-k8s ~]# kubectl get service jenkins
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
jenkins   LoadBalancer   10.101.41.251   192.168.1.11   80:31287/TCP   7m34s
```
- 192.168.1.11 주소로 젠킨스가 외부에 노출되었음을 알 수 있다.

### 9) 젠킨스가 마스터 노드에 있음을 확인
```bash
[root@m-k8s ~]# kubectl get pod -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP              NODE    NOMINATED NODE   READINESS GATES
jenkins-76496d9db7-7dkt4   2/2     Running   0          8m51s   172.16.171.74   m-k8s   <none>           <none>
```

### 10) 왜 마스터 노드에 파드가 배포가 되었을까?
```bash
[root@m-k8s ~]# kubectl get node m-k8s -o yaml | nl
     1  apiVersion: v1
     2  kind: Node
    [중략]
    14      kubernetes.io/arch: amd64
    15      kubernetes.io/hostname: m-k8s
    16      kubernetes.io/os: linux
    [중략]  
   164    taints:
   165    - effect: NoSchedule
   166      key: node-role.kubernetes.io/master
    [생략]
[root@m-k8s ~]# kubectl get deployments jenkins -o yaml | nl
     1  apiVersion: apps/v1
     2  kind: Deployment
     3  metadata:
     4    annotations:
     5      deployment.kubernetes.io/revision: "1"
     6      meta.helm.sh/release-name: jenkins
     7      meta.helm.sh/release-namespace: default
    [중략]
    562        tolerations:
    563        - effect: NoSchedule
    564          key: node-role.kubernetes.io/master
    565          operator: Exists
    [생략]
```

- 위의 테인트(taints)와 톨러레이션(tolerations)이 이런 결과를 만든 설정이다.


#### jenkins-install.sh의 내용
```bash
[root@m-k8s ~]# cat ~/_Book_k8sInfra/ch5/5.3.1/jenkins-install.sh
1 #!/usr/bin/env bash
2 jkopt1="--sessionTimeout=1440"
3 jkopt2="--sessionEviction=86400"
4 jvopt1="-Duser.timezone=Asia/Seoul"
5 jvopt2="-Dcasc.jenkins.config=https://raw.githubusercontent.com/sysnet4admin/_Book_k8sInfra/main/ch5/5.3.1/jenkins-config.yaml"
6 jvopt3="-Dhudson.model.DownloadService.noSignatureCheck=true"
7
8 helm install jenkins edu/jenkins \
9 --set persistence.existingClaim=jenkins \
10 --set master.adminPassword=admin \
11 --set master.nodeSelector."kubernetes\.io/hostname"=m-k8s \
12 --set master.tolerations[0].key=node-role.kubernetes.io/master \
13 --set master.tolerations[0].effect=NoSchedule \
14 --set master.tolerations[0].operator=Exists \
15 --set master.runAsUser=1000 \
16 --set master.runAsGroup=1000 \
17 --set master.tag=2.249.3-lts-centos7 \
18 --set master.serviceType=LoadBalancer \
19 --set master.servicePort=80 \
20 --set master.jenkinsOpts="$jkopt1 $jkopt2" \
21 --set master.javaOpts="$jvopt1 $jvopt2 $jvopt3"
```
- 2~3번째 줄 : 세션의 유효 시간을 1440분(하루)으로 설정하고 세션을 정리하는 시간을 86400초(하루)로 설정한다. <- 기본 설정은 30분이므로 불편
- 4번째 줄 : 젠킨스를 통한 CI/CD 시 시간대를 명확히 맞춰주기 위함
- 5번째 줄 : 가상 머신인 마스터 노드가 재시작한다면 이에 대한 설정이 초기화, 따라서 설정값을 깃허브 저장소에서 받아오도록 설정
- 8번째 줄 : edu 차트 저장소의 jenkins 차트를 사용해 jenkins 릴리스를 설치
- 9번째 줄 : PVC 동적 프로비저닝을 사용할 수 없는 가상 머신 기반 환경이므로 이미 만들어놓은 jenkins라는 이름의 PVC를 사용하도록 설정
- 10번째 줄 : 젠킨스 접속 시 사용할 관리자 비밀번호를 admin으로 설정
- 11번째 줄 : 젠킨스의 컨트롤러 파드를 쿠버네티스 마스터 노드 m-k8s에 배치하도록 선택
     - nodeSelector는 nodeSelector 뒤에 따라오는 문자열과 일치하는 레이블을 가진 노드에 파드를 스케줄링하겠다는 설정
     - kubernetes.io/hostname에 '.'가 포함되어 있기 때문에 \가 없다면 kubernetes.io를 하나의 문자열로 인식하지 못한다. (\ : 이스케이프)
- 12~14번째 줄 : 현재 마스터 노드에는 NoSchedule이라는 테인트가 설정되어 있는 상태이므로, 테이늩가 설정된 노드에 파드를 배치하기 위해서는 tolerations 옵션 필요
- 15~16번째 줄 : 젠킨스를 구동하는 파드가 실행될 때 가질 유저 ID와 그룹 ID 설정
- 17번째 줄 : 젠킨스 버전에 따른 UI 변경을 막기 위한 설정
- 18번째 줄 : 차트로 생성되는 서비스의 타입을 로드밸런서로 설정하여 외부 IP를 받아옴
- 19번째 줄 : 젠킨스가 http 상에서 구동되도록 포트를 80으로 지정
- 20번째 줄 : 젠킨스에 추가로 필요한 설정들을 2~3번째 줄에 변수로 선언
- 21번째 줄 : 젠킨스를 구동하기 위한 환경설정에 필요한 것들을 4~5번째 줄에 변수로 선언


## 젠킨스 접속 및 설정 내용
### 1) 젠킨스 접속
- 로드밸런서 타입의 외부 IP인 192.168.1.11에 접속하면 젠킨스 로그인 화면 확인 가능
- admin/admin으로 접속

### 2) 메인 화면 메뉴
- 새로운 Item : 젠킨스를 통해 빌드할 작업을 아이템이라고 한다.
- 사람 : 사용자 관리
- 빌드 기록 : 젠킨스 작업에 대한 성공, 실패, 진행 내역 확인 가능
- Jenkins 관리 : 젠킨스의 시스템, 보안, 도구, 플러그인 등 각종 설정하는 곳
- My Views : 젠킨스에서 각종 작업을 분류해 모아서 볼 수 있는 대시보드
- Lockable Resources : 젠킨스에서 동시에 여러 작업이 일어날 때 작업이 진행 중이라면 옵션에 따라 다른 작업은 대기해야 하는데, 이 때 동시성 문제를 위한 잠금 장치를 설정할 수 있는 곳
- 대시보드인 View를 생성하는 작업

### 3) 젠킨스 관리 메뉴
- 시스템 설정 : 메인 화면에 표시될 문구, 동시에 실행할 수 있는 실행기 개수, 젠킨스 접속 가능 경로, 관리자 정보, 시스템 전체에 적용할 수 있는 환경변수, 플러그인 파일의 경로, 설정 정보
- Global Tool Configuration : 빌드 과정에서 사용하는 도구의 경로 및 옵션 설정 가능
- 플러그인 관리 : 젠킨스에서 사용할 플러그인을 설치, 삭제, 업데이트 가능
- 노드 관리 : 젠킨스에서 사용하는 노드를 추가, 삭제하거나 노드의 세부 설정 및 상태 모니터링을 할 수 있는 메뉴
- Configuration as Code : 젠킨스의 설정을 내보내거나 불러오기 가능
- Manage Credentials : 젠킨스에서 사용하는 플러그인에 필요한 접근 키, 비밀 키, API 토큰과 같은 접속에 필요한 인증 정보를 관리

### 4) 시스템 설정 메뉴
- 시스템 메시지 : 

## 젠킨스 컨트롤러 설정


















