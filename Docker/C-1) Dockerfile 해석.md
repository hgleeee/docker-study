# Dockerfile 해석
## 해석할 예시 Dockerfile
```bash
FROM openjdk:8
LABEL description="Echo IP Java Application"
EXPOSE 60431
COPY ./target/app-in-host.jar /opt/app-in-image.jar
WORKDIR /opt
ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]
```

## 도커 파일 ↔ 배시 명령
||도커 파일 내용|배시 명령으로 유사 해석|
|--|-------|-------|
|1|FROM openjdk:8|import openjdk:8 image|
|2|LABEL description="Echo IP Java Application"|Label_desc="Echo IP Java Application"|
|3|EXPOSE 60431|EXPOSE=60431|
|4|COPY ./target/app-in-host.jar /opt/app-in-image.jar|scp <HOST>/target/app-in-host.jar <Image>/opt/app-in-image.jar|
|5|WORKDIR /opt|cd /opt|
|6|ENTRYPOINT [ "java", "-jar", "app-in-image.jar" ]|./java -jar app-in-image.jar|


## 항목 당 설명
### 1번째 줄
- FROM <이미지 이름>:[태그] 형식으로 이미지를 가져온다.
- 가져온 이미지 내부에서 컨테이너 이미지를 빌드한다.
### 2번째 줄
- LABEL <레이블 이름>=<값> 형식으로 이미지에 부가적인 설명을 하기 위한 레이블을 추가할 때 사용한다.
### 3번째 줄
- EXPOSE <숫자> 형식으로 생성된 이미지로 컨테이너를 구동할 때 어떤 포트를 사용하는지 알려준다.
- EXPOSE를 사용한다고 해서 컨테이너 구동 시 자동으로 해당 포트를 호스트 포트와 연결하지 않는다.
- 실제 외부에서 접속하려면 docker run 명령어로 이미지를 컨테이너로 빌드할 때 -p 옵션으로 포트를 연결해야 한다.
### 4번째 줄
- COPY <호스트 경로> <컨테이너 경로> 형식이다.
- 호스트에서 새로 생성하는 컨테이너 이미지로 필요한 파일을 복사한다.
- 메이븐을 통해 생성한 app-in-host.jar 파일을 이미지의 /opt/app-in-image.jar로 복사한다.
### 5번째 줄
- 이미지의 현 작업 위치를 opt로 변경한다.
### 6번째 줄
- ENTRYPOINT 뒤에 나오는 대괄호 안에 든 명령을 실행한다.
- 컨테이너를 구동할 때 java -jar app-in-image.jar가 실행된다.
- ENTRYPOINT로 실행하는 명령어는 컨테이너를 구동할 때 첫 번째로 실행되며 프로세스 컨테이너 냅무에서 첫 번째로 실행되었다는 의미로 PID는 1이 된다.
