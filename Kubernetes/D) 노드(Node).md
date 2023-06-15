# 노드(Node)
> 클러스터 > 노드 > 파드 > 컨테이너
## 정의
- 노드란 클러스터 내 가상 서버, 즉 컴퓨팅 엔진 단위라고 이해하면 된다.
- 클러스터 다음으로 큰 단위이며, 마스터 노드와 워커 노드로 분리돼 있다.
- 마스터 노드: 전체 쿠버네티스 시스템을 관리 및 통제하는 쿠버네티스 컨트롤 플레인을 관장
- 워커 노드: 배포하고자 하는 어플리케이션의 실제 실행을 수행
- 마스터 노드가 죽으면 클러스터를 관리할 노드가 없기에, 일반적으로 3개 정도의 마스터 노드를 띄워 관리하는 것으로 알려져 있으며, 워커 노드 또한 여러 개 구성할 수 있다.

## 노드 자원 보호
### 개요
- 최근 몇 차례 문제가 생긴 노드에 파드를 할당한다면 문제가 될 가능성이 높다.
- 노드에 문제가 생기더라도 파드의 문제는 최소화하여야 한다.
- 어떻게 문제가 생길 가능성이 있는 노드라는 것을 쿠버네티스에 알릴까? 이 때 cordon 기능을 사용한다.

### cordon으로 노드 관리
#### 1번
```bash
[root@m-k8s ~]# kubectl apply -f ~/_Book_k8sInfra/ch3/3.2.8/echo-hname.yaml
deployment.apps/echo-hname configured
```
- yaml 파일을 apply 해서 파드를 생성한다.
#### 2번
```bash
[root@m-k8s ~]# kubectl scale deployment echo-hname --replicas=9
deployment.apps/echo-hname scaled
```
- scale 명령을 통해 배포한 파드를 9개로 늘린다.

#### 3번
```bash
[root@m-k8s ~]# kubectl get pods \-o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase,NODE:.spec.nodeName
NAME                        IP               STATUS    NODE
echo-hname-7894b67f-8gjpk   172.16.221.135   Running   w1-k8s
echo-hname-7894b67f-l6rz9   172.16.132.8     Running   w3-k8s
echo-hname-7894b67f-ldk89   172.16.103.135   Running   w2-k8s
echo-hname-7894b67f-mjbb5   172.16.103.138   Running   w2-k8s
echo-hname-7894b67f-ncg62   172.16.132.6     Running   w3-k8s
echo-hname-7894b67f-pjq8f   172.16.221.138   Running   w1-k8s
echo-hname-7894b67f-skhtv   172.16.103.137   Running   w2-k8s
echo-hname-7894b67f-wh6qn   172.16.221.137   Running   w1-k8s
echo-hname-7894b67f-wz2gs   172.16.132.9     Running   w3-k8s
```

- kubectl get pods -o=custom-columns 명령을 사용하는데, 이 때 custom-columns는 사용자가 임의로 구성할 수 있는 열을 의미한다.
- 명령에서 .metadata.name 등의 형식은 배포된 파드의 내용을 yaml 형식으로 받아보는 것으로 생각할 수 있다.

#### 4번
```bash
[root@m-k8s ~]# kubectl scale deployment echo-hname --replicas=3
deployment.apps/echo-hname scaled
```
- 파드를 3개로 줄인다.

#### 5번
```bash
[root@m-k8s ~]# kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase,NODE:.spec.nodeName
NAME                        IP               STATUS    NODE
echo-hname-7894b67f-8gjpk   172.16.221.135   Running   w1-k8s
echo-hname-7894b67f-ldk89   172.16.103.135   Running   w2-k8s
echo-hname-7894b67f-ncg62   172.16.132.6     Running   w3-k8s
```
- 각 노드에 파드가 한 개씩만 남았는지를 확인한다.

#### 6번
```bash
[root@m-k8s ~]# kubectl cordon w3-k8s
node/w3-k8s cordoned
```
- w3-k8s 노드에 문제가 자주 발생하여 현 상태를 보존해야 한다고 가정하자.
- w3-k8s 노드에 cordon 명령을 실행한다.

#### 7번
```bash
[root@m-k8s ~]# kubectl get nodes
NAME     STATUS                     ROLES    AGE   VERSION
m-k8s    Ready                      master   9h    v1.18.4
w1-k8s   Ready                      <none>   9h    v1.18.4
w2-k8s   Ready                      <none>   8h    v1.18.4
w3-k8s   Ready,SchedulingDisabled   <none>   8h    v1.18.4
```
- 위 결과에서 볼 수 있다시피 cordon 명령을 실행하면 해당 노드에 파드가 할당되지 않게 SchedulingDisabled 표시를 한다.

#### 8번
```bash
[root@m-k8s ~]# kubectl scale deployment echo-hname --replicas=9
deployment.apps/echo-hname scaled
```
```bash
[root@m-k8s ~]# kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase,NODE:.spec.nodeName
NAME                        IP               STATUS    NODE
echo-hname-7894b67f-4nnwz   172.16.103.139   Running   w2-k8s
echo-hname-7894b67f-8gjpk   172.16.221.135   Running   w1-k8s
echo-hname-7894b67f-b7gn9   172.16.221.139   Running   w1-k8s
echo-hname-7894b67f-jd4sg   172.16.103.140   Running   w2-k8s
echo-hname-7894b67f-ldk89   172.16.103.135   Running   w2-k8s
echo-hname-7894b67f-ncg62   172.16.132.6     Running   w3-k8s
echo-hname-7894b67f-q2s7z   <none>           Pending   w2-k8s
echo-hname-7894b67f-t5t4z   <none>           Pending   w1-k8s
echo-hname-7894b67f-zmm4n   172.16.221.140   Running   w1-k8s
```

- 파드 수를 9개로 늘리고 노드에 배포된 파드를 확인한 결과, 기존에 할당된 파드를 제외하고는 더 이상 w3-k8s에 할당되지 않았다.

#### 9번
```bash
[root@m-k8s ~]# kubectl scale deployment echo-hname --replicas=3
deployment.apps/echo-hname scaled
```
```bash
[root@m-k8s ~]# kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase,NODE:.spec.nodeName
NAME                        IP               STATUS    NODE
echo-hname-7894b67f-8gjpk   172.16.221.135   Running   w1-k8s
echo-hname-7894b67f-ldk89   172.16.103.135   Running   w2-k8s
echo-hname-7894b67f-ncg62   172.16.132.6     Running   w3-k8s
```
- 파드 수를 3개로 다시 줄이고, 각 노드에 파드가 1개씩 공평하게 할당되었는가를 확인할 수 있다.


#### 10번
```bash
[root@m-k8s ~]# kubectl uncordon w3-k8s
node/w3-k8s uncordoned
```
```bash
[root@m-k8s ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
m-k8s    Ready    master   9h    v1.18.4
w1-k8s   Ready    <none>   9h    v1.18.4
w2-k8s   Ready    <none>   9h    v1.18.4
w3-k8s   Ready    <none>   9h    v1.18.4
```
- uncordon 명령을 통해 w3-k8s에 파드가 할당되지 않도록 했던 것을 해제한다.
- cordon 기능으로 문제가 발생할 가능성이 있는 노드를 스케줄되지 않게 설정할 수 있다.


## 노드 유지보수하기
### 개요
- 쿠버네티스는 유지보수를 위해 노드를 꺼야하는 상황이 발생한다.
- 이런 경우를 대비해 drain 기능을 제공하여 지정된 노드의 파드를 전부 다른 곳으로 이동시켜 해당 노드를 유지보수할 수 있게 해준다.

### 1번
```bash
[root@m-k8s ~]# kubectl drain w3-k8s
node/w3-k8s cordoned
error: unable to drain node "w3-k8s", aborting command...

There are pending nodes to be drained:
 w3-k8s
error: cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/calico-node-h75xp, kube-system/kube-proxy-6x6zs
```
- drain 명령을 실행하면 w3-k8s의 데몬셋을 지울 수 없어서 명령을 수행할 수 없다는 에러가 발생한다.
- drain의 동작 과정은 해당 노드에서 파드를 삭제하고 다른 곳에 다시 생성한다는 것을 알 수 있다.
- 데몬셋의 경우는 각 노드마다 한 개씩 존재하는 파드라서 drain으로는 삭제할 수 없다.

### 2번
```bash
[root@m-k8s ~]# kubectl drain w3-k8s --ignore-daemonsets
node/w3-k8s already cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-h75xp, kube-system/kube-proxy-6x6zs
evicting pod default/echo-hname-7894b67f-ncg62
pod/echo-hname-7894b67f-ncg62 evicted
node/w3-k8s evicted
```
- drain 명령과 ignore-daemonsets 옵션을 함께 활용하여 데몬셋을 무시하고 진행한다.

### 3번
```bash
[root@m-k8s ~]# kubectl get pods -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase,NODE:.spec.nodeName
NAME                        IP               STATUS    NODE
echo-hname-7894b67f-8gjpk   172.16.221.135   Running   w1-k8s
echo-hname-7894b67f-ldk89   172.16.103.135   Running   w2-k8s
echo-hname-7894b67f-x5ccs   172.16.103.142   Running   w2-k8s
```
- w3-k8s에 파드가 없는 것을 확인한다.

### 4번
```bash
[root@m-k8s ~]# kubectl get nodes
NAME     STATUS                     ROLES    AGE   VERSION
m-k8s    Ready                      master   9h    v1.18.4
w1-k8s   Ready                      <none>   9h    v1.18.4
w2-k8s   Ready                      <none>   9h    v1.18.4
w3-k8s   Ready,SchedulingDisabled   <none>   9h    v1.18.4
```
- cordon을 실행했을 때와 마찬가지로 SchedulingDisabled 상태이다.
- 이후로 유지보수가 끝났다고 가정하면 uncordon을 통해 스케줄을 다시 받을 수 있는 상태로 되돌릴 수 있다.


















