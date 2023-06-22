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

























