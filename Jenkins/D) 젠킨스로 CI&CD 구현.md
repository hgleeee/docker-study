# 젠킨스로 CI&CD 구현
- 아이템(Item) : 새롭게 정의할 작업
- 프로젝트 : 모든 작업의 정의와 순서를 모아둔 전체 작업

## 새로운 Item 옵션
### Freestyle project 
- 스타일의 자유도 ↑, 브라우저에서 사용자가 직접 설정값과 수행할 동작 입력
- 과정이 복잡한 작업을 구성하기 어렵고 Freestyle로 생성한 아이템은 입력 항목의 명세서를 별도 저장 과정이 없으므로 작성한 내용의 공유가 어렵다.
### Pipeline 
- 젠킨스가 지원하는 고유의 Pipeline 문법으로 코드를 작성해 작업을 정의하는 프로젝트
- 변수 정의, 반복문, 조건문 등의 프로그래밍 기법을 사용할 수 있어 비교적 복잡한 방식의 작업 정의 가능
- Pipeline 코드로 작성한 파일을 함께 올리면 애플리케이션 코드와 배포 방법을 함께 관리할 수 있음

### Multi-configuration project 
- 하나의 소스 코드를 여러 조건의 조합으로 해당되는 환경에 동시에 배포하는 프로젝트

### Folder
- 젠킨스의 작업이 늘어날수록 관리가 어려움.
- 분류 가능한 디렉터리를 생성하는 것
### Multibranch Pipeline
- 하나의 소스 코드 저장소 내에 존재하는 각 브랜치에서 젠킨스 파이프라인 코드가 작성된 파일을 불러와 한 번에 여러 브랜치에 대해 품질 검증, 테스트, 빌드 등의 작업을 가능하게 함.

## Freestyle로 echo-ip (Nginx 웹 서버) 배포
### 순서 요약
1. 깃허브에서 echo-ip를 빌드할 정보가 담긴 파일들을 내려(pull)받는다.
2. 받은 파일들을 이용해 컨테이너 이미지를 빌드한다.
3. 빌드한 이미지를 레지스트리(192.168.1.10:8443)에 저장(push)한다.
4. 레지스트리에 저장한 이미지를 쿠버네티스 클러스터에 디플로이먼트로 생성하고 로드밸런서 서비스로 노출한다.

## 실습
### 1) New Item에서 Freestyle project 아이템 선택
- 이름은 dpy-fs-dir-prod로 지정한다.

### 2) Restrict where this project can be run 체크 해제
- 해당 설정은 젠킨스의 에이전트가 특정 레이블을 가지고 있을 때 해당 레이블을 가진 에이전트에서만 실행될 수 있도록 제한하는 옵션

### 3) 소스 코드 관리 탭에서 지정
- Repository URL에 깃허브 소스코드 주소 입력
- Branch Specifier에 깃허브 저장소에 존재하는 main 브랜치 변경 내용에 대해서만 CI를 진행하도록 */MAIN 지정

### 4) Build 단계 추가
- Execute Shell을 선택함으로써 이 항목에 입력한 셸 명령어로 빌드 작업 수행

### 5) 젠킨스에서 빌드에 사용할 명령어를 확인하고 입력
- 명령어는 도커 이미지 빌드, 도커 이미지 푸시, 디플로이먼트 생성, 로드밸런서를 통한 노출의 4단계로 이루어짐

```bash
[root@m-k8s ~]# cat ~/_Book_k8sInfra/ch5/5.4.1/echo-ip-101.freestyle 
docker build -t 192.168.1.10:8443/echo-ip .                          # 도커 빌드 / CI 작업
docker push 192.168.1.10:8443/echo-ip                                # 도커 이미지 저장 / CI 작업
kubectl create deployment fs-echo-ip --image=192.168.1.10:8443/echo-ip  # 쿠버네티스 디플로이먼트 배포 / CD 작업
kubectl expose deployment fs-echo-ip --type=LoadBalancer --name=fs-echo-ip-svc --port=8080 --target-port=80  # 쿠버네티스 서비스 노출 / CD 작업
```

### 6) 저장한 후 Build Now 버튼으로 저장한 프로젝트 실행

### 7) CI/CD 작업 성공 여부 확인
- Build History에 파란색 원이 생성되었는지 확인
- 빨간색 원이라면 CI/CD 작업 중 문제가 발생한 것이다.

### 8) 쿠버네티스 클러스터에 디플로이먼트와 로드밸런서가 정상 배포되었는지 확인
```bash
[root@m-k8s ~]# kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
fs-echo-ip   1/1     1            1           9m58s
jenkins      1/1     1            1           37h
[root@m-k8s ~]# kubectl get services
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
fs-echo-ip-svc   LoadBalancer   10.109.99.230   192.168.1.12   8080:31602/TCP   10m
jenkins          LoadBalancer   10.101.41.251   192.168.1.11   80:31287/TCP     37h
jenkins-agent    ClusterIP      10.99.232.19    <none>         50000/TCP        37h
kubernetes       ClusterIP      10.96.0.1       <none>         443/TCP          2d17h
```

- 디플로이먼트 fs-echo-ip와 로드밸런스 fs-echo-ip-svc가 정상 배포된 것을 확인할 수 있다.

### 9) 브라우저에서 배포된 파드 정보가 화면에 출력되는지 확인
- 192.168.1.12:8080 접속 -> 성공적


## Pipeline로 echo-ip 배포
> 연속적인 작업을 코드 또는 파일로 정의해주는 젠킨스 기능 (코드로써의 파이프라인, Pipeline-As-Code)

### 2가지의 문법
#### 스크립트 문법 (Scripted pipeline)
- 젠킨스 에이전트를 설정할 때 익숙하지 않은 젠킨스의 고유 문법으로 작성해야 함.
#### 선언적인 문법 (Declarative pipeline)
- 익숙한 야믈을 그대로 사용할 수 있음.

### 순서 요약
1. 깃허브와 같은 소스 코드 저장소에서 빌드할 소스 코드와 젠킨스 내부의 작업을 선언적인 문법으로 정의한 Jenkinsfile을 내려받는다.
2. 내려받은 Jenkinsfile을 해석해 작성자의 의도에 맞는 작업 자동 수행 (앞에서 Freestyle로 했던 것과 같음)

### 실습
#### 1) New Item에서 Pipeline 아이템 선택
- 이름은 dpy-pl-bulk-prod로 지정한다.

#### 2) Build Triggers 탭
- Build after other projects are built : 다른 프로젝트를 빌드한 후 이 프로젝트 빌드 (여러 프로젝트를 빌드할 때 순서에 따른 의존 관계 있을 경우 유용)
- Build periodically : 주기적으로 프로젝트 빌드 수행 (일정 주기로 빌드를 수행하는 경우 사용, 주기를 설정할 경우 Cron이라는 스케줄 도구 문법 활용)
- Poll SCM : 깃허브 등의 소스 코드 저장소에서 주기적으로 내용을 검사해 빌드
- 빌드 안함 : 빌드를 사용하지 않음 (임시로 사용하지 않을 프로젝트 등에 설정)
- Quiet Period : 빌드를 실행할 때 약간의 지연 시간을 주는 옵션
- 빌드를 원격으로 유발 : 젠킨스 빌드 작업을 외부에서 URL을 호출해 시작할 때 사용 (깃허브의 푸시 또는 메신저의 웹훅과 같이 주소를 이용해 빌드 시작하는 곳에서 사용)

#### 3) Advanced Project Options 탭
- 젠킨스의 플러그인 설치에 따라 생성

#### 4) Pipeline 탭
- Pipeline script를 선택할 경우 해당 화면에서 직접 입력한 내용 사용 (선언적 문법 사용)
- Pipeline script from SCM을 선택할 경우 외부 소스 코드 저장소에서 선언적 문법으로 작성된 파일을 가져와 실행

#### 5) Jenkinsfile 소스 해석
- D-1 참조.


#### 6) 젠킨스의 빌드 및 배포 작업
- Build Now 버튼 클릭

#### 7) 쿠버네티스 클러스터에서 디플로이먼트와 로드밸런서 정상 배포 확인
```bash
[root@m-k8s ~]# kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
jenkins        1/1     1            1           38h
pl-bulk-prod   1/1     1            1           98s
[root@m-k8s ~]# kubectl get services
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
jenkins            LoadBalancer   10.101.41.251   192.168.1.11   80:31287/TCP     38h
jenkins-agent      ClusterIP      10.99.232.19    <none>         50000/TCP        38h
kubernetes         ClusterIP      10.96.0.1       <none>         443/TCP          2d18h
pl-bulk-prod-svc   LoadBalancer   10.103.18.61    192.168.1.12   8080:31576/TCP   102s
```











