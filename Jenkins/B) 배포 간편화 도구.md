# 배포 간편화 도구

## 개요
- 쿠버네티스 클러스터 환경에서는 이미 이러한 배포 도구가 준비되어 있다.
- 사용자마다 필요한 환경적 요소가 모두 다르기 때문에 필요하다.

## 큐브시티엘 (kubectl)
- 쿠버네티스에 기본으로 포함된 커맨드라인 도구로, 추가 설치 없이 바로 사용 가능하다.
- 오브젝트 생성과 쿠버네티스 클러스터에 존재하는 오브젝터, 이벤트 등의 정보를 확인하는 데에 사용된다.
- 큐브시티엘은 정의된 매니페스트 파일을 그대로 배포하기 때문에 개별적인 오브젝트를 관리하거나 배포할 때 사용하는 것이 좋다.

## 커스터마이즈 (kustomize)
- 오브젝트를 사용자의 의도에 따라 유동적 배포가 가능하다. 별도의 커스터마이즈 실행 파일을 활용해 yaml 파일을 생성할 수 있다.
- yaml 파일이 이미 존재한다면 kubectl로도 배포할 수 있는 옵션 -k 가 있을 정도로 kubectl과 밀접하게 동작한다.

## 헬름 (Helm)
- 오브젝트 배포에 필요한 사양이 이미 정의된 차트라는 패키지를 활용한다.
- 헬름 차트 저장소가 온라인에 존재해 패키지를 검색하고 내려받아 사용하기 편리하다.
- 헬름은 오브젝트를 묶어 패키지 단위로 관리하므로 단순한 1개의 명령어로 애플리케이션에 필요한 오브젝트들을 구성할 수 있다.

|구분|큐브시티엘|커스터마이즈|헬름|
|---|:-------:|:-------:|:-------:|
|설치 방법|쿠버네티스에 기본 포함|별도 실행 파일 또는 쿠버네티스에 통합|별도 설치|
|배포 대상|정적인 야믈 파일|커스터마이즈 파일|패키지(차트)|
|주 용도|오브젝트 관리 및 배포|오브젝트 가변적 배포|패키지 단위 오브젝트 배포 및 관리|
|가변적 환경|대응 힘듦(야믈 수정 필요)|간단 대응 가능|복잡 대응 가능|
|기능 복잡도|단순|보통|복잡|
