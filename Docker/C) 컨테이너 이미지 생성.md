# 컨테이너 이미지 생성
> 컨테이너 인프라 환경을 구성할 때 제공된 이미지를 사용할 수 있지만, 직접 만든 App으로 컨테이너를 만들 수도 있다.

## 기본 방법으로 빌드
### 1) 기본적인 컨테이너 빌드 도구 & 파일 확인
```bash
[root@m-k8s ~]# cd ~/_Book_k8sInfra/ch4/4.3.1/
[root@m-k8s 4.3.1]# ls
Dockerfile  mvnw  pom.xml  src
```
- Dockerfile : 컨테이너 이미지를 빌드하기 위한 정보
- mvnw : 메이븐 래퍼라는 이름의 리눅스 스크립트, 메이븐 실행을 위한 환경 설정 자동화
- pom.xml : 메이븐 래퍼가 작동할 때 필요한 절차와 빌드 정보
- src(디렉터리) : 메이븐으로 빌드할 자바 소스 디렉터리

### 2) 자바 개발 도구 (JDK, Java Development Kit) 설치
- 소스 코드가 자바로 작성되어 있으므로 실행 가능한 바이너리(JAR)로 만들려면 JDK가 필요하다.

### 3) 자바 빌드
- 자바를 빌드할 때 메이븐을 사용한다.
- 메이븐은 빌드를 위한 의존성과 여러 설정을 자동화하는 도구이다.
- mvnw clean package 명령으로 메이븐을 실행하는데, 빌드를 진행할 디렉터리를 비우고 (clean) JAR (package)를 생성하라는 의미이다.

### 4) JAR 파일 확인
```bash
[root@m-k8s 4.3.1]# ls target
app-in-host.jar  app-in-host.jar.original  classes  generated-sources  maven-archiver  maven-status
```

### 5) 컨테이너 이미지 빌드
```bash
[root@m-k8s 4.3.1]# docker build -t basic-img .
```
- -t (tag) : 만들어질 이미지를 가리킴
- . (dot) : 원하는 내용을 추가하거나 변경하는 데에 필요한 작업 공간을 현재 디렉터리로 지정하겠다는 의미
- 빌드 내용을 이해하기 위해서는 Dockerfile을 뜯어봐야 한다.


### 6) 5단계에서 생성한 이미지 확인
```bash
[root@m-k8s 4.3.1]# docker images basic-img
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
basic-img           latest              0d85f7eeca24        55 seconds ago      544 MB
```

### 7) 5단계와 태그 정보만 바꿔서 이미지 빌드
```bash
[root@m-k8s 4.3.1]# docker build -t basic-img:1.0 -t basic-img:2.0 .
Sending build context to Docker daemon  17.7 MB
Step 1/6 : FROM openjdk:8
 ---> b273004037cc
Step 2/6 : LABEL description "Echo IP Java Application"
 ---> Using cache
 ---> d5640afc18f2
Step 3/6 : EXPOSE 60431
 ---> Using cache
 ---> 74d6f9e7a54e
Step 4/6 : COPY ./target/app-in-host.jar /opt/app-in-image.jar
 ---> Using cache
 ---> 3c8a3bf51588
Step 5/6 : WORKDIR /opt
 ---> Using cache
 ---> d79e85c6c0d6
Step 6/6 : ENTRYPOINT java -jar app-in-image.jar
 ---> Using cache
 ---> 0d85f7eeca24
Successfully built 0d85f7eeca24
```
- 캐시를 사용해 빠르게 빌드된다.

### 8) 생성된 이미지 확인
```bash
[root@m-k8s 4.3.1]# docker images basic-img
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
basic-img           1.0                 0d85f7eeca24        3 minutes ago       544 MB
basic-img           2.0                 0d85f7eeca24        3 minutes ago       544 MB
basic-img           latest              0d85f7eeca24        3 minutes ago       544 MB
```
- 태그 정보만 다를 뿐 모두 같은 이미지이며, 한 공간을 사용한다.

### 9) Dockerfile 내용 중 일부 변경
```bash
[root@m-k8s 4.3.1]# sed -i 's/Application/Development/' Dockerfile
[root@m-k8s 4.3.1]# docker build -t basic-img:3.0 .
```

### 10) 생성된 이미지 확인
```bash
[root@m-k8s 4.3.1]# docker images basic-img
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
basic-img           3.0                 bb22c35ade3c        33 seconds ago      544 MB
basic-img           1.0                 0d85f7eeca24        27 minutes ago      544 MB
basic-img           2.0                 0d85f7eeca24        27 minutes ago      544 MB
basic-img           latest              0d85f7eeca24        27 minutes ago      544 MB
```
- 위에서 볼 수 있듯 IMAGE ID가 전혀 다른 것을 확인할 수 있다.

### 11) 컨테이너 실행 후 docker ps로 컨테이너 상태 출력
```bash
[root@m-k8s 4.3.1]# docker run -d -p 60431:80 --name basic-run --restart always basic-img
6cba1fb40bb8774e11500560884e4360becda8f9a425bd4f717e08abac1e436a
[root@m-k8s 4.3.1]# docker ps -f name=basic-run
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                              NAMES
6cba1fb40bb8        basic-img           "java -jar app-in-..."   10 seconds ago      Up 10 seconds       60431/tcp, 0.0.0.0:60431->80/tcp   basic-run
```

### 12) curl을 이용해 자기 자신의 IP에 60431 포트로 요청을 보내고 응답이 오는지 확인
```bash
[root@m-k8s 4.3.1]# curl 127.0.0.1:60431
src: 172.17.0.1 / dest: 127.0.0.1
```

## 컨테이너 용량 줄이기
### 1) 앞에서 진행한 이미지 빌드 과정을 스크립트로 작성
```bash
[root@m-k8s 4.3.2]# cat build-in-host.sh
#!/usr/bin/env bash
yum -y install java-1.8.0-openjdk-devel
./mvnw clean package
```

### 2) Dockerfile 변경점 확인
```bash
[root@m-k8s 4.3.2]# cat Dockerfile
FROM gcr.io/distroless/java:8
LABEL description="Echo IP Java Application"
EXPOSE 60432
COPY ./target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]
```
- 사용하는 기초 이미지가 openjdk에서 gcr.io/distroless/java:8 로 변경
- 기본 방법은 openjdk 이미지를 설치할 때 자바 개발 도구 또한 함께 설치하게 되는데, 이 때 openjdk 이미지에 포함된 자바 개발 도구는 불필요하게 낭비되는 공간이다.

### 3) 경량화 이미지 빌드
```bash
[root@m-k8s 4.3.2]# chmod 700 mvnw
[root@m-k8s 4.3.2]# ./build-in-host.sh
```

### 4) 새로 빌드된 이미지와 기본 방법으로 빌드된 이미지 비교
```bash
[root@m-k8s 4.3.2]# docker images | head -n 3
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
optimal-img                           latest              30e51d32fac4        53 seconds ago      148 MB
basic-img                             latest              97037275bfc8        18 minutes ago      544 MB
```
- head -n 3 옵션 : 첫 3줄만 출력

### 5) 컨테이너 실행 후 정상 동작 확인
```bash
[root@m-k8s 4.3.2]# docker run -d -p 60432:80 --name optimal-run --restart always optimal-img
65543cb929c93747791dae96e355631210dd0073907e392fedd2b71dac330059
[root@m-k8s 4.3.2]# curl 127.0.0.1:60432
src: 172.17.0.1 / dest: 127.0.0.1
```

## 컨테이너 내부에서 컨테이너 빌드
### 1) Dockerfile 확인
```bash
[root@m-k8s 4.3.3]# cat Dockerfile
FROM openjdk:8 # 자바 개발 도구가 포함된 이미지
LABEL description="Echo IP Java Application"
EXPOSE 60433 
RUN git clone https://github.com/iac-source/inbuilder.git # RUN으로 이미지 내부에서 소스코드 실행
WORKDIR inbuilder # git clone으로 내려받은 디렉터리를 현 작업공간으로 설정
RUN chmod 700 mvnw # mvnw 실행 권한 변경
RUN ./mvnw clean package # 메이븐 래퍼로 JAR 빌드
RUN mv target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]
```
- 이미지 내부에 소스 코드를 내려받기 위해 깃을 사용
- 내려받은 소스 코드를 이미지 내부에서 실행하기 위해 RUN을 추가

### 2) 이미지 내부에 내려받은 inbuilder 저장소 확인
```bash
[root@m-k8s 4.3.3]# git clone https://github.com/iac-source/inbuilder.git
Cloning into 'inbuilder'...
remote: Enumerating objects: 53, done.
remote: Counting objects: 100% (53/53), done.
remote: Compressing objects: 100% (36/36), done.
remote: Total 53 (delta 10), reused 38 (delta 2), pack-reused 0
Unpacking objects: 100% (53/53), done.
[root@m-k8s 4.3.3]# ls inbuilder/
mvnw  mvnw.cmd  pom.xml  README.md  src
```
- mvnw.cmd는 윈도용 스크립트, README.md는 깃허브용 안내 파일이다.

### 3) 컨테이너 이미지 빌드
```bash
[root@m-k8s 4.3.3]# docker build -t nohost-img .
```

### 4) 컨테이너 이미지 확인 및 비교
```bash
[root@m-k8s 4.3.3]# docker images | head -n 4
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
nohost-img                            latest              dfeb703555ba        38 seconds ago      633 MB
optimal-img                           latest              30e51d32fac4        2 hours ago         148 MB
basic-img                             latest              97037275bfc8        3 hours ago         544 MB
```
- 왜 컨테이너 내부에서 빌드한 이미지가 가장 클까?
- 컨테이너 내부에서 빌드를 진행하기 때문에 빌드 중간에 생성한 파일들과 내려받은 라이브러리 캐시들이 최종 이미지인 nohost-img에 그대로 남는다.

### 5) 컨테이너 실행 및 curl로 확인
```bash
[root@m-k8s 4.3.3]# docker run -d -p 60433:80 --name nohost-run --restart always nohost-img
10362387a1e4de279ac18f2427a44d1c66735d0c18a70c36b18081e8a24c3f19
[root@m-k8s 4.3.3]# curl 127.0.0.1:60433
src: 172.17.0.1 / dest: 127.0.0.1
```


## 최적화해 컨테이너 빌드
> 멀티 스테이지 빌드 : 최종 이미지의 용량을 줄일 수 있고, 호스트에 어떠한 빌드 도구도 설치할 필요 X

### 1) Dockerfile 확인
```bash
# 1단계 : JAVA 소스를 빌드해 JAR로 만듦
[root@m-k8s 4.3.4]# cat Dockerfile
FROM openjdk:8 AS int-build
LABEL description="Java Application builder"
RUN git clone https://github.com/iac-source/inbuilder.git
WORKDIR inbuilder
RUN chmod 700 mvnw
RUN ./mvnw clean package

# 2단계 : 빌드된 JAR을 경량화 이미지에 복사함
FROM gcr.io/distroless/java:8
LABEL description="Echo IP Java Application"
EXPOSE 60434
COPY --from=int-build inbuilder/target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]
```

- 멀티 스테이지의 핵심은 빌드하는 위치와 최종 이미지를 '분리'하는 것이다.

### 2) 컨테이너 이미지 빌드
```bash
[root@m-k8s 4.3.4]# docker build -t multistage-img .
```

### 3) 빌드된 이미지 확인
```bash
[root@m-k8s 4.3.4]# docker images | head -n 3
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
multistage-img                       latest              3ca9b776a684        12 seconds ago      148MB
<none>                               <none>              0dfa8c1af7bc        25 seconds ago      615MB
```
- optimal-img 의 용량과 같은데, 둘은 빌드 단계가 서로 같다.
- 차이점은 자바 소스를 호스트에서 빌드했는지, 컨테이너 내에서 빌드했는지 차이밖에 없다.

### 4) 댕글링 이미지 삭제
```bash
[root@m-k8s 4.3.4]# docker rmi $(docker images -f dangling=true -q)
Deleted: sha256:0dfa8c1af7bca11bdaac743be98fd386674220f8c15ec6521c3c9b2a248fac7c
Deleted: sha256:d526532d1e32835d859aca0278b6ffdf01912e26af975219a789b05c678c25aa
Deleted: sha256:5a1c6c433459855d8f7d13f25b0110a8555b962d2735839dd8c3cce30d3eec36
Deleted: sha256:25bda325b9002914e204e0b0625cc21fb0d111aba31c37d64fbd624b9b02de05
Deleted: sha256:97762ec134806a1bac07507616ec6971a20accea23ab4c2bb82c26fb08d4a3e3
Deleted: sha256:eab3531ae91065dabc8700f229e24c0bd3c9908547b1c311e7b1daec9190af0d
Deleted: sha256:43e58c7414ffd953d16d5a582822cc60d4c7c6198d61cc8e39cbcee569ce606d
Deleted: sha256:82b8fc04bb65a364f2022eb045acf6061421ba3679a6540d13958eeaebc21bb5
```
- <none>으로 표시되는 이미지에 해당된다.
- 이름이 없는 해당 이미지를 댕글링 이미지라고 하며, 멀티 스테이지 과정에서 자바 소스를 빌드할 때 생성된 이미지이다.

### 5) 컨테이너 실행 및 작동 확인
```bash
[root@m-k8s 4.3.4]# docker run -d -p 60434:80 --name multistage-run --restart always multistage-img
a229a6b0279c82e752012eb92447ffa39467cfd05a67ae55f6e9c21058007c94
[root@m-k8s 4.3.4]# curl 127.0.0.1:60434
src: 172.17.0.1 / dest: 127.0.0.1
```

## 쿠버네티스에서 직접 만든 컨테이너 사용



















