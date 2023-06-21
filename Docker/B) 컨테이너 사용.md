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


## 3. 컨테이너 내부 파일 변경
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

#### 5) cp 명령어로 index.html 파일을 /root/html/ 디렉토리에 복사

```bash
[root@m-k8s ~]# cp ~/_Book_k8sInfra/ch4/4.2.3/index-BindMount.html /root/html/index.html
[root@m-k8s ~]# ls /root/html
index.html
```

#### 6) 브라우저에서 192.168.1.10:8081 로 접속
- index.html이 표시됨을 확인할 수 있다.
- 현재 상태는 바인드 마운트로 index.html이 호스트에서 컨테이너로 전달된 상태이다.


### 볼륨으로 호스트와 컨테이너 연결

#### 1) 볼륨 생성
```bash
[root@m-k8s ~]# docker volume create nginx-volume
nginx-volume
```
- 컨테이너에 연결할 볼륨을 호스트에 생성한다.

#### 2) 볼륨 조회
```bash
[root@m-k8s ~]# docker volume inspect nginx-volume
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/nginx-volume/_data",
        "Name": "nginx-volume",
        "Options": {},
        "Scope": "local"
    }
]
```

#### 3) 볼륨으로 생성된 디렉터리 확인
```bash
[root@m-k8s ~]# ls /var/lib/docker/volumes/nginx-volume/_data/
[root@m-k8s ~]#
```

#### 4) 호스트와 컨테이너의 디렉터리를 연결할 컨테이너 구동
```bash
[root@m-k8s ~]# docker run -d -v nginx-volume:/usr/share/nginx/html -p 8082:80 --restart always --name nginx-volume nginx
baefd58b83055044086b7995818690066ebe8c08efff760e4ea5ad5b0437cc34
```
- nginx-volume 이라는 이름의 볼륨을 구동하고 컨테이너 내부의 /usr/share/nginx/html 디렉터리와 호스트의 nginx-volume 볼륨을 연결한다.
- -v [볼륨 이름]:[컨테이너 디렉터리] 옵션 사용

#### 5) 볼륨 디렉터리 재확인
```bash
[root@m-k8s ~]# ls /var/lib/docker/volumes/nginx-volume/_data/
50x.html  index.html
```

#### 6) 브라우저로 접속
- 성공적으로 index.html이 브라우저로 보여지는 것을 알 수 있다.
- 볼륨은 바인드 마운트 방식과는 달리 컨테이너 디렉터리에 덮어쓰는 것이 아닌 양쪽을 서로 동기화시키는 구조이기 때문에 컨테이너 디렉터리의 기존 파일이 보존된다.

#### 7) cp 명령어로 index.html 파일 바꿔치기
```bash
[root@m-k8s ~]# cp ~/_Book_k8sInfra/ch4/4.2.3/index-Volume.html /var/lib/docker/volumes/nginx-volume/_data/index.html
cp: overwrite ‘/var/lib/docker/volumes/nginx-volume/_data/index.html’? y
```
- 바꾼 index.html 파일이 브라우저에 나타난다.
- 이를 통해 볼륨을 사용하면 컨테이너에 존재하는 파일을 보존하거나, 필요할 때 변경해서 사용할 수 있음을 알 수 있다.

## 4. 사용하지 않는 컨테이너 정리
### 컨테이너 정지
> 컨테이너나 이미지를 삭제하기 전에 먼저 선행되어야 할 것이 정지이다.

#### 1) nginx 이미지를 기반으로 생성된 컨테이너 조회
```bash
[root@m-k8s ~]# docker ps -f ancestor=nginx
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
baefd58b8305        nginx               "/docker-entrypoin..."   7 minutes ago       Up 7 minutes        0.0.0.0:8082->80/tcp   nginx-volume
42c4f6694636        nginx               "/docker-entrypoin..."   19 hours ago        Up 2 hours          0.0.0.0:8081->80/tcp   nginx-bind-mounts
7df1d2fe196c        nginx               "/docker-entrypoin..."   20 hours ago        Up 2 hours          0.0.0.0:8080->80/tcp   nginx-exposed
2c6c78709099        nginx               "/docker-entrypoin..."   23 hours ago        Up 2 hours          80/tcp                 inspiring_goldberg
```
- ancestor 키는 컨테이너를 생성하는 데 사용한 이미지를 기준으로 필터링한다.

#### 2) 컨테이너를 이름으로 정지
```bash
[root@m-k8s ~]# docker stop inspiring_goldberg
inspiring_goldberg
```
- docker stop <컨테이너 이름 | ID> 명령으로 컨테이너를 정지할 수 있다.

#### 3) 컨테이너를 ID로 정지
```bash
[root@m-k8s ~]# docker stop 7df
7df
```
- nginx-exposed 정지

#### 4) nginx 이미지를 사용하는 모든 컨테이너를 한꺼번에 정지
```bash
[root@m-k8s ~]# docker ps -q -f ancestor=nginx
baefd58b8305
42c4f6694636
```
- 위의 조회 명령으로 nginx 이미지를 사용하는 모든 컨테이너 ID를 출력할 수 있다.
- -q (--quite) 옵션으로 ID만 출력 가능

```bash
[root@m-k8s ~]# docker stop $(docker ps -q -f ancestor=nginx)
baefd58b8305
42c4f6694636
```
- 위와 같이 $() 안에 docker stop의 인자로 사용하도록 할 수 있다.

#### 5) 정지된 컨테이너를 포함한 모든 컨테이너 조회
```bash
[root@m-k8s ~]# docker ps -a -f ancestor=nginx
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                          PORTS               NAMES
baefd58b8305        nginx               "/docker-entrypoin..."   14 minutes ago      Exited (0) About a minute ago                       nginx-volume
42c4f6694636        nginx               "/docker-entrypoin..."   19 hours ago        Exited (0) About a minute ago                       nginx-bind-mounts
7df1d2fe196c        nginx               "/docker-entrypoin..."   20 hours ago        Exited (0) 3 minutes ago                            nginx-exposed
2c6c78709099        nginx               "/docker-entrypoin..."   23 hours ago        Exited (0) 5 minutes ago                            inspiring_goldberg
```
- 정지된 컨테이너를 다시 구동하고 싶다면 docker start <컨테이너 이름 | ID> 명령어를 사용한다.


### 컨테이너와 이미지 삭제
#### 1) 컨테이너 삭제
```bash
[root@m-k8s ~]# docker rm $(docker ps -aq -f ancestor=nginx)
baefd58b8305
42c4f6694636
7df1d2fe196c
2c6c78709099
```
- docker rm <컨테이너 이름 | ID> 명령어로 컨테이너를 삭제한다.

#### 2) 이미지 삭제
```bash
[root@m-k8s ~]# docker rmi $(docker images -q nginx)
Untagged: docker.io/nginx:latest
Untagged: docker.io/nginx@sha256:593dac25b7733ffb7afe1a72649a43e574778bf025ad60514ef40f6b5d606247
Deleted: sha256:eb4a57159180767450cb8426e6367f11b999653d8f185b5e3b78a9ca30c2c31d
Deleted: sha256:387c6708d068d261ce5b1fe3e67323cbf64d8a37901f3d9742557f4abb830baf
Deleted: sha256:2946620cb422511c62ba67d12b1c16bbf6b85e6ce42e93a4dace94b4a70160b3
Deleted: sha256:f2545115e362a40e5b3fe057ad159aa9824f40a0e9341f4743b4d0c4f5322435
Deleted: sha256:9b3ff8c6f07faac480afaeecc0388a387f8cf92832de656a2d35e890340ac59a
Deleted: sha256:77366f15e73eef5c23ff7bd0be0c09f1b280c9586863232392c2d500eed148e7
Deleted: sha256:7447c8c6be248218804380a22d47c130f7efc16f31550cb446fc3cc91f98a54c
Deleted: sha256:ac4d164fef90ff58466b67e23deb79a47b5abd30af9ebf1735b57da6e4af1323
Untagged: docker.io/nginx:stable
Untagged: docker.io/nginx@sha256:a8281ce42034b078dc7d88a5bfe6d25d75956aad9abba75150798b90fa3d1010
Deleted: sha256:2af0ea4a9556b049337d026dd7df7f9c20661203c634be4f9b976814c05e5c32
Deleted: sha256:e6067ec1fcb755253b636345433596519db188e171c187b77e2406c18b5bddb5
Deleted: sha256:bc1572112f4383964945f8ec3e476dad3a11f2cd5dbefad7e913aa3375b895ac
Deleted: sha256:bfd119cd090e088fbedd6e9a62c6c8be4f6bb80436bd050c7403c974b5582548
Deleted: sha256:46dcd0ba3f14025b1e146ce79dab6b4eec3b63558fb6482ce520470b5d2b4985
Deleted: sha256:a600211c75798c52c7a8a5186af235b1b3e81491dd20d43b744de78f275bed65
Deleted: sha256:0cc1f01656262cc1319655e8570146e4aa190c3fb8c7e81c353760c44a96c13b
```
- rmi 는 remove image 라는 의미이다.
- 이미지는 컨테이너가 정지 상태가 아닌 삭제된 상태일 때 비로소 삭제 가능하다.














