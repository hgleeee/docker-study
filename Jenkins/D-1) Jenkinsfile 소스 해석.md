# Jenkinsfile 소스 해석
## 구성 요소
### pipeline
- 선언적인 문법이 시작하는 부분
- 선언적인 문법으로 작성된 작업들은 pipeline {}의 사이에 작업 내용을 작성해야 한다.

### agent
- 작업을 수행할 에이전트를 지정하고 필요한 설정을 한다.
- 지정된 에이전트 내부에서 젠킨스 빌드 작업이 수행된다.

#### 여러 가지 방식 지정 가능 
- any : 사용 가능한 에이전트를 젠킨스가 임의로 지정
- label : 특정 레이블과 일치하는 에이전트 노드 지정
- docker : 에이전트 노드의 이미지를 도커로 지정
- kubernetes : 에이전트 노드를 쿠버네티스 파드로 지정

### stages
- stage들을 모아서 정의하고 이를 순서대로 진행하게 해준다.

### stage
- step들을 정의하는 영역
- stage는 괄호 안에 여러 step들을 정의할 수 있는데 이 step들 내부에서 실제로 동작되는 내용들이 정의된다.

### steps
- stage 내부에서 실제 작업 내용을 작성하는 영역
- 내부에서 script, sh, git과 같은 작업(work)을 통해 실제로 동작하게 된다.

## 실제 Jenkinsfile
```bash
# Jenkinsfile
01 pipeline {
02   agent any
03   stages {
04     stage('git scm update') {
05       steps {
06         git url: 'https://github.com/IaC-Source/echo-ip.git', branch: 'main'
07       }
08     }
09     stage('docker build and push') {
10       steps {
11         sh '''
12         docker build -t 192.168.1.10:8443/echo-ip .
13         docker push 192.168.1.10:8443/echo-ip
14         '''
15       }
16     }
17     stage('deploy kubernetes') {
18       steps {
19         sh '''
20         kubectl create deployment pl-bulk-prod --image=192.168.1.10:8443/echo-ip
21         kubectl expose deployment pl-bulk-prod --type=LoadBalancer --port=8080 \
22                                                --target-port=80 --name=pl-bulk-prod-svc
23         '''
24       }
25     }
26   }
27 }
```

### 4~8번째 줄
- 소스 코드 저장소인 깃허브로부터 소스 코드를 내려받은 단계이다.
- git 작업을 사용하며, 인자로 요구하는 git url을 깃허브 저장소의 주소로 지정하고 branch는 main 브랜치로 설정하였다.

### 9~16번째 줄
- 도커 명령을 이용해 컨테이너 이미지를 빌드하고, 빌드한 이미지를 레지스트리에 저장하는 작업을 수행한다.
- sh 작업을 통해 docker 명령을 사용한다.

### 17~24번째 줄
- kubectl 명령으로 전 단계에서 레지스트리에 저장한 이미지를 pl-bulk-prod(디플로이먼트)로 배포
- 배포한 pl-bulk-prod를 kubectl 명령으로 로드밸런서 타입으로 노출하는 작업 수행
- sh 작업을 통해 kubectl 명령을 사용한다.
