# 컨테이너 사용
## 컨테이너 이미지와 컨테이너의 관계
<p align="center"><img src="../images/image_container_relation.png" width="600"></p>

- 컨테이너 이미지는 그대로는 사용할 수 없고 도커와 같은 CRI로 불러들여야 컨테이너가 실제 작동한다.
- 실행 파일과 실행된 파일 관계로 볼 수 있다.

## 1. 컨테이너 이미지 찾기
### 이미지 검색 후 내려받기

#### 레지스트리에서 검색
- 이미지는 __레지스트리__ 라고 하는 저장소에 모여 있다. (레지스트리는 도커 허브처럼 공개된 유명 레지스트리일수도, 내부 구축된 레지스트리일수도 있음)
- docker search <검색어>를 입력하면 검색어를 포함하는 이미지가 있는지 찾는다.

```bash
[root@m-k8s ~]# docker search nginx
INDEX       NAME                                                        DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
docker.io   docker.io/nginx                                             Official build of Nginx.                        18650     [OK]
docker.io   docker.io/linuxserver/nginx                                 An Nginx container, brought to you by Linu...   203     
docker.io   docker.io/bitnami/nginx                                     Bitnami nginx Docker Image                      165                  [OK]
docker.io   docker.io/nginxproxy/acme-companion                         Automated ACME SSL certificate generation ...   116     
docker.io   docker.io/ubuntu/nginx                                      Nginx, a high-performance reverse proxy & ...   96     
[생략]
```
- INDEX : 이미지가 저장된 레지스트리의 이름
- NAME : 검색된 이미지 이름, 공식 이미지를 제외하고 나머지는 '레지스트리 주소/저장소 소유자/이미지 이름'의 형태이다.
- DESCRIPTION : 이미지에 대한 설명
- STARS : 해당 이미지를 내려받은 사용자에게 받은 평가 횟수로써, 숫자가 클수록 신뢰성 ↑
- OFFICIAL : [OK] 표시는 해당 이미지에 포함된 애플리케이션, 미들웨어 등을 개발한 업체에서 공식 제공한 이미지라는 뜻
- AUTOMATED : [OK] 표시는 도커 허브에서 자체적으로 제공하는 이미지 빌드 자동화 기능을 활용해 생성한 이미지라는 뜻

#### 이미지 내려받기
```bash
[root@m-k8s ~]# docker pull nginx
Using default tag: latest
Trying to pull repository docker.io/library/nginx ...
latest: Pulling from docker.io/library/nginx
5b5fe70539cd: Pull complete
441a1b465367: Pull complete
3b9543f2b500: Pull complete
ca89ed5461a9: Pull complete
b0e1283145af: Pull complete
4b98867cde79: Pull complete
4a85ce26214d: Pull complete
Digest: sha256:593dac25b7733ffb7afe1a72649a43e574778bf025ad60514ef40f6b5d606247
Status: Downloaded newer image for docker.io/nginx:latest
```
- 태그 : 이미지를 내려받을 때 사용한 태그를 알 수 있다. 아무런 조건 없이 이미지 이름만으로 pull 하면 latest 태그가 적용되고 가장 최신 이미지를 의미한다.
- 레이어 : 하나의 이미지는 여러 개의 레이어로 이루어져 있으므로 레이어마다 Pull complete 메시지가 발생한다.
- 다이제스트 : 이미지의 고유 식별자로 이미지에 포함된 내용과 이미지의 생성 환경을 식별할 수 있다. 이름이나 태그는 이미지를 생성할 때 임의로 지정하므로 완전 구별이 불가하지만 다이제스트는 고유 값이므로 가능하다.
- 상태 : '레지스트리 이름/이미지 이름:태그'의 형태

### 이미지 태그
- 태그는 이름이 동일한 이미지에 추가하는 식별자이다. 도커 이미지의 버전이나 플랫폼이 다를 수 있으므로 이를 구분하기 위함.
- 이미지를 내려받거나 이미지를 기반으로 컨테이너를 구동할 때 태그를 따로 명시하지 않으면 latest 태그를 기본으로 사용한다.
- 안정화 버전인 stable 태그도 존재한다.

### 이미지 레이어 구조
- 이미지는 같은 내용일 경우 여러 이미지에 동일한 레이어를 공유하므로 전체 용량이 감소한다.

## 2. 컨테이너 실행
### 컨테이너 단순 실행
#### 1) 새로운 컨테이너 실행
```bash
[root@m-k8s ~]# docker run -d --restart always nginx
2c6c787090995a7d2aaa3eefba09a54c62d3d7e32783a010e0fcba0868c9be68
```

- docker run으로 컨테이너를 생성하면 결과값으로 16진수 문자열이 나오는데, 해당 문자열은 컨테이너를 식별할 수 있는 고유 ID이다.
- 컨테이너 생성 명령 형식 : docker run [옵션] <사용할 이미지 이름>[:태그 | @다이제스트], 태그와 다이제스트는 생략 가능하다.
- -d (--detach) : 컨테이너를 백그라운드에서 구동한다는 의미이다. 옵션을 생략하게 되면 컨테이너 내부에서 실행되는 애플리케이션의 상태가 화면에 표시된다.
- --restart always : 가상 머신을 중지한 후 다시 실행해도 자동으로 컨테이너가 기존 상태를 이어갈 수 있다.

    |값|컨테이너 비정상 종료 시|도커 서비스 시작 시|
    |--|--------|--------|
    |no(기본값)|컨테이너를 재시작하지 않음|컨테이너를 시작하지 않음|
    |on-failure|컨테이너를 재시작함|컨테이너를 시작함|
    |always|컨테이너를 재시작함|컨테이너를 시작함|
    |unless-stopped|컨테이너를 재시작함|사용자가 직접 정지하지 않은 컨테이너만 시작함|

#### 2) docker ps 명령으로 컨테이너 상태 확인
```bash
[root@m-k8s ~]# docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
2c6c78709099        nginx                  "/docker-entrypoin..."   7 minutes ago       Up 7 minutes        80/tcp              inspiring_goldberg
[생략]
```
- CONTAINER ID : 컨테이너를 식별하기 위한 고유 ID
- IMAGE : 컨테이너를 만드는 데 사용한 이미지
- COMMAND : 컨테이너가 생성될 때 내부에서 작동한 프로그램을 실행하는 명령어
- CREATED : 컨테이너가 생성된 시각
- STATUS : 컨테이너가 작동을 시작한 시각 (CREATED와 달리 컨테이너를 중지했다 재시작하면 초기화됨)
- PORTS : 컨테이너가 사용하는 포트와 프로토콜 표시
- NAMES : 컨테이너 이름 표시


#### 3) 컨테이너를 지정해 검색
```bash
[root@m-k8s ~]# docker ps -f id=2c6c
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
2c6c78709099        nginx               "/docker-entrypoin..."   11 minutes ago      Up 11 minutes       80/tcp              inspiring_goldberg
```
- -f (--filtering) <필터링 대상> : 검색 결과 필터링할 수 있음
- 필터링 대상 안에는 key(대상)=value(값) 형식으로 입력한다.
- key 항목에는 id, name, label, exited(컨테이너 종료됐을 때 반환되는 숫자 코드), status(컨테이너 작동 상태), ancestor(컨테이너가 사용하는 이미지)가 포함된다.

#### 4) curl 127.0.0.1 명령으로 컨테이너가 제공하는 nginx 웹 페이지 정보 가져오기
```bash
[root@m-k8s ~]# curl 127.0.0.1
curl: (7) Failed connect to 127.0.0.1:80; Connection refused
```
- 이 때 연결할 수 없다는 오류가 발생한다. 왜일까?
- 컨테이너의 PORTS 열에 표시되는 80/tcp는 컨테이너 내부에서 TCP 프로토콜의 80번 포트를 사용한다는 의미이다.
- 그러나, curl 127.0.0.1로 전달한 요청은 로컬호스트의 80번 포트로만 전달될 뿐 컨테이너에 도달하지는 못하는데, 그 이유는 호스트에 도달한 뒤 컨테이너까지 도달하기 위한 추가 경로 설정이 없기 때문이다.

### 추가로 경로를 설정해 컨테이너 실행
#### 1) docker run에 -p 8080:80 옵션을 추가해 새로운 컨테이너 실행
```bash
[root@m-k8s ~]# docker run -d -p 8080:80 --name nginx-exposed --restart always nginx
7df1d2fe196c541925f3365c27d28369c770d926add1163b967252dde60f2fb5
```
- -p (publish) : -p <요청받을 호스트 포트>:<연결할 컨테이너 포트> 형식으로 사용, 외부에서 호스트로 보낸 요청을 컨테이너 내부로 전달하는 옵션이다.

#### 2) docker ps로 확인
```bash
[root@m-k8s ~]# docker ps -f name=nginx-exposed
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
7df1d2fe196c        nginx               "/docker-entrypoin..."   49 minutes ago      Up 49 minutes       0.0.0.0:8080->80/tcp   nginx-exposed
```
##### 0.0.0.0:8080->80/tcp
- 0.0.0.0의 8080번 포트로 들어오는 요청을 컨테이너 내부의 80번 포트로 전달한다는 의미이다.
- 0.0.0.0은 모든 네트워크 어댑터를 의미한다.
- m-k8s 호스트는 자기 자신을 가리키는 127.0.0.1과 외부에 노출된 192.168.1.10 의 IP를 가지고 있는데, 어떤 IP든 컨테이너 내부의 80 포트로 전달된다.
##### nginx-exposed : 현재 작동중인 컨테이너 이름, --name 옵션으로 직접 지정해주었다.

#### 3) 웹 브라우저에서 접근
- 192.168.1.10:8080 으로 접근했을 때 NGINX 초기 화면이 출력되는 것을 확인할 수 있다.


## 컨테이너 내부 파일 변경
### 방법
#### docker cp
- docker cp <호스트 경로> <컨테이너 이름>:<컨테이너 내부 경로> 형식이다.
- 호스트에 위치한 파일을 구동 중인 컨테이너 내부에 복사한다.
#### Dockerfile ADD
- 이미지는 Dockerfile을 기반으로 만들어진다.
- Dockerfile에 ADD라는 구문으로 컨테이너 내부로 복사할 파일을 지정하면 이미지를 빌드할 때 지정한 파일이 이미지 내부로 복사된다.
#### 바인드 마운트
- 호스트의 파일 시스템과 컨테이너 내부를 연결해 어느 한쪽에서 작업한 내용이 양쪽에 동시에 반영되는 방법이다.
- 새로운 컨테이너를 구동할 때에도 호스트와 연결할 파일이나 디렉터리의 경로만 지정하면 다른 컨테이너에 있는 파일을 새로 생성한 컨테이너와 연결할 수 있다.
#### 볼륨
- 호스트의 특정 디렉터리가 아닌 도커가 관리하는 볼륨을 컨테이너와 연결한다.
- 도커가 관리하는 볼륨 공간을 NFS와 같은 공유 디렉터리에 생성한다면 다른 호스트에서도 도커가 관리하는 볼륨을 함께 사용할 수 있다.

### 바인드 마운트로 호스트와 컨테이너 연결

#### 1) 컨테이너 내부에 연결할 /root/html/ 디렉터리를 호스트에 생성
```bash
[root@m-k8s ~]# mkdir -p /root/html
```

#### 2) docker run 명령으로 컨테이너를 구동하고, 컨테이너의 디렉터리와 호스트의 디렉터리 연결
```bash
[root@m-k8s ~]# docker run -d -p 8081:80 -v /root/html:/usr/share/nginx/html --restart always --name nginx-bind-mounts nginx
42c4f6694636303ff7f56eef7784172a9d87d29747682b1bf4865a03f3cb995c
```
- -v (--volume) : 호스트 디렉터리와 컨테이너 디렉터리를 연결하는 옵션으로 -v <호스트 디렉터리 경로>:[컨테이너 디렉터리 경로] 형식으로 사용한다.
- 바인드 마운트의 특성으로, 호스트 디렉터리의 내용을 컨테이너 디렉터리에 덮어쓰므로 기존 컨테이너 디렉터리의 내용은 삭제된다.

#### 3) nginx-bind-mounts 컨테이너를 조회
```bash
[root@m-k8s ~]# docker ps -f name=nginx-bind-mounts
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
42c4f6694636        nginx               "/docker-entrypoin..."   4 minutes ago       Up 4 minutes        0.0.0.0:8081->80/tcp   nginx-bind-mounts
```

#### 4) 사용자 호스트에 생성한 /root/html 디렉터리 확인
```bash
[root@m-k8s ~]# ls /root/html
[root@m-k8s ~]#
```
- 컨테이너 내부와 연결된 /root/html/ 디렉터리가 비어있음을 확인할 수 있고, 따라서 현재 컨테이너의 nginx는 초기 화면으로 보여줄 파일이 없다.

### 볼륨으로 호스트와 컨테이너 연결






















