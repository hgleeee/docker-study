# 컨테이너 심화
## 쿠버네티스가 컨테이너를 다루는 과정 (파드 생성 과정을 컨테이너 중심으로)
<p align="center"><img src="../images/kube_container_create.png" width="600"></p>

### 1번
- 사용자는 kube-apiserver의 url로 요청을 전달하거나 kubectl을 통해 kube-apiserver에 파드를 생성하는 명령을 내린다.

### 2번
- 파드 생성 명령은 네트워크를 통해 kubelet으로 전달된다.
- kube-apiserver는 노드에 있는 kubelet과 안전하게 통신하기 위해 인증서와 키로 통신 내용을 암호화해 전달한다.
- 이 때, 키는 마스터 노드의 /etc/kubernetes/pki/ 디렉터리에 보관되고, 인증서 파일인 api-server-kubelet-client.crt와 키 파일인 apiserver-kubelet-client.key를 사용한다.

```bash
[root@m-k8s ~]# ls /etc/kubernetes/pki/apiserver-kubelet-client.*
/etc/kubernetes/pki/apiserver-kubelet-client.crt  /etc/kubernetes/pki/apiserver-kubelet-client.key
```
- kubelet으로 생성 요청이 전달되면 kubelet은 해당 요청이 적절한 사용자로부터 전달된 것인지 검증 과정을 거친다.
- 이 때, /var/lib/kubelet/config.yaml 파일의 clientCAFile 속성에 설정된 파일을 사용한다.

```bash
[root@m-k8s ~]# cat /var/lib/kubelet/config.yaml | grep clientCAFile
    clientCAFile: /etc/kubernetes/pki/ca.crt
```

### 3번
- 컨테이너디에 컨테이너 생성 명령을 내린다. 이 때 명령 형식은 컨테이너 런타임 인터페이스(CRI)를 따른다.
- CRI는 컨테이너 관련 명령인 런타임 서비스와 이미지 관련 명령인 이미지 서비스로 이루어진다.
- 런타임 서비스는 파드의 생성, 삭제, 정지, 목록 조회와 컨테이너 생성, 시작, 정지, 삭제, 목록 조회, 상태 조회 등 다양한 명령을 내린다.
- kubelet이 내린 명령은 컨테이너디에 통합된 CRI 플러그인이라는 구성 요소에 전달되며 컨테이너디가 컨테이너 생성 명령을 직접 호출한다.

### 4번
- 컨테이너디는 containerd-shim이라는 자식 프로세스를 생성해 컨테이너를 관리한다.
```bash
UID        PID  PPID  C STIME TTY          TIME CMD
root       998     1  0 10:03 ?        00:01:03 /usr/bin/containerd
root       999     1  1 10:03 ?        00:12:02 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root      1474   998  0 10:04 ?        00:00:00 containerd-shim -namespace moby [중략]
root      1486   998  0 10:04 ?        00:00:00 containerd-shim -namespace moby [중략]
[생략]
```
- 다음 출력 예시를 보면 PID 998인 containerd 프로세스는 PID 1474, 1486인 자식 프로세스 containerd-shim을 가진다.
- 이 때, PID 999 dockerd는 사용자가 입력한 도커 명령을 컨테이너디에 전달하기 위해 도커 엔진 내부에 생성된 프로세스이다.

### 5번
- containerd가 생성한 containerd-shim 프로세스는 컨테이너를 조작한다.
- 실제로, containerd-shim이 runC 바이너리 실행 파일을 호출해 컨테이너를 생성한다.

## 컨테이너 PID 1의 의미
- PID 1은 커널이 할당하는 첫 번째 PID라는 의미의 특수한 PID이다.
- 일반적으로 init 또는 systemd에 PID 1이 할당되며 시스템 구동에 필요한 프로세스들을 띄우는 매우 중요한 역할을 한다.
- PID 1번 이외에도 PID 0번 -> swapper, PIE 2번 -> kthreadd 등이 예약된 PID를 가진다.

### 왜 컨테이너 PID 1번은 컨테이너에서 실행된 App인 nginx 프로세스일까?
- 해당 PID의 역할은 시스템을 구동하는 역할이고, 컨테이너는 이미 구동된 시스템, 즉 커널 위에서 동작한다.
- 컨테이너는 OS 시스템을 구동시킬 필요 없이 바로 동작하기 때문에 시스템에 예약된 PID 1번이 할당되지 않은 상태이다.
- 따라서, 컨테이너 세계에서는 컨테이너가 실행하는 첫 애플리케이션에, 예시에서는 nginx에게 할당할 수 있게 된다.

### 실습 (도커로 생성한 컨테이너 PID와 컨테이너 내부 PID 1번이 연결되어 있는가)

#### 1) nginx 관련 프로세스가 호스트에 있는지 확인
```bash
[root@m-k8s ~]# ps -ef | grep -v auto | grep nginx
[root@m-k8s ~]#
```

#### 2) nginx 컨테이너 구동 후 nginx 관련 프로세스 재확인
```bash
[root@m-k8s ~]# docker run -d nginx
3b18ce3ea87b60e8947e11d4224e6a5da103f11f0857179aa9a8b7788de5dd9a
[root@m-k8s ~]# ps -ef | grep -v auto | grep nginx
root     23366 23338  1 12:49 ?        00:00:00 nginx: master process nginx -g daemon off;
101      23409 23366  0 12:49 ?        00:00:00 nginx: worker process
101      23410 23366  0 12:49 ?        00:00:00 nginx: worker process
```

- ps -ef | grep -v auto | grep nginx 실행 결과 PID가 6592인 nginx 프로세스가 나타난다.
- 자식 프로세스인 6629번은 nginx의 실제 역할을 담당하며 6592가 생성될 때 같이 생성되므로, nginx 기능을 하는 프로세스는 6592 하나라고 볼 수 있다.
- 현재 생성된 프로세스는 도커가 생성한 것이므로 컨테이너 내부에 격리되어 있는데 호스트에서 ps -ef 명령어를 실행했을 때 프로세스를 확인 가능하다.
- 이는 호스트에서 보이는 프로세스와 컨테이너 프로세스가 연결되어 있음을 추측할 수 있다.

#### 3) 컨테이너 내부의 nginx 프로세스 확인
```bash
[root@m-k8s ~]# docker exec 3b1 ls -l /proc/1/exe
lrwxrwxrwx. 1 root root 0 Jun 27 03:50 /proc/1/exe -> /usr/sbin/nginx
```
- 리눅스는 프로세스와 관련된 정보를 /proc/<PID>/ 디렉터리에 보관하며, /proc/<PID>/exe는 해당 <PID>를 가지는 프로세스를 생성하기 위해 실행된 명령어를 보여준다.
- /proc/1/exe -> /usr/sbin/nginx는 컨테이너 내부에서 PID 1을 가지고 구동 중인 프로세스가 nginx임을 의미한다.

#### 4) 호스트와 컨테이너 내부의 nginx가 동일한 프로세스임을 증명
- 리눅스는 자원을 

```bash
[root@m-k8s ~]# cat /proc/23366/cgroup
[중략]
1:name=systemd:/docker/3b18ce3ea87b60e8947e11d4224e6a5da103f11f0857179aa9a8b7788de5dd9a
[root@m-k8s ~]# docker exec 3b18 cat /proc/1/cgroup
[중략]
1:name=systemd:/docker/3b18ce3ea87b60e8947e11d4224e6a5da103f11f0857179aa9a8b7788de5dd9a
```

#### 5)
```bash
[root@m-k8s ~]# kill -9 23366
[root@m-k8s ~]# cat /proc/23366/cgroup
cat: /proc/23366/cgroup: No such file or directory
[root@m-k8s ~]# docker exec 3b18 cat /proc/1/cgroup
Error response from daemon: Container 3b18ce3ea87b60e8947e11d4224e6a5da103f11f0857179aa9a8b7788de5dd9a is not running
```

## 도커 아닌 runC로 컨테이너 생성

#### 1)
```bash
[root@m-k8s ~]# docker run -d nginx
1f84cd2e470751c52dd0445e8e4e1181a0a0b50ebc0016bd41b733bd5f7965e2
[root@m-k8s ~]# docker export 1f84 > nginx.tar
[root@m-k8s ~]# ls nginx.tar
nginx.tar

```

#### 2)
```bash
[root@m-k8s ~]# mkdir nginx-container
[root@m-k8s ~]# tar -C nginx-container -svf nginx.tar
[생략]
[root@m-k8s ~]# ls nginx-container
bin   dev                  docker-entrypoint.sh  home  lib32  libx32  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint.d  etc                   lib   lib64  media   opt  root  sbin  sys  usr

```

#### 3)
```bash
[root@m-k8s ~]# ~/_Book_k8sInfra/app/D.DeepDiveContainer/ns-create.sh
```

#### 4)
```bash










