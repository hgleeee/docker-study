# Jenkinsfile 소스 해석-2

```bash 
pipeline {                                                 
  agent {
    kubernetes {                                           # 3
      yaml '''                                             # 4
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          app: blue-green-deploy
        name: blue-green-deploy
      spec:
        containers:
        - name: kustomize
          image: sysnet4admin/kustomize:3.6.1
          tty: true
          volumeMounts:
          - mountPath: /bin/kubectl
            name: kubectl
          command:
          - cat
        serviceAccount: jenkins
        volumes:
        - name: kubectl
          hostPath:
            path: /bin/kubectl
      '''                                                   # 26
    }
  }
  stages {                                                  # 29
    stage('git scm update'){
      steps {
        git url: 'https://github.com/IaC-Source/blue-green.git', branch: 'main'
      }
    }                                                        # 34
    stage('define tag'){                                     # 35   
      steps {
        script {
          if(env.BUILD_NUMBER.toInteger() % 2 == 1){
            env.tag = "blue"
          } else {
            env.tag = "green"
          }
        }
      }
    }                                                         # 45
    stage('deploy configmap and deployment'){                 # 46
      steps {
        container('kustomize'){
          dir('deployment'){
            sh '''
            kubectl apply -f configmap.yaml
            kustomize create --resources ./deployment.yaml
            echo "deploy new deployment"
            kustomize edit add label deploy:$tag -f
            kustomize edit set namesuffix -- -$tag
            kustomize edit set image sysnet4admin/dashboard:$tag
            kustomize build . | kubectl apply -f -
            echo "retrieve new deployment"
            kubectl get deployments -o wide
            '''
          }
        }
      }    
    }                                                          # 64
    stage('switching LB'){                                     # 65
      steps {
        container('kustomize'){
          dir('service'){
            sh '''
            kustomize create --resources ./lb.yaml
            while true;
            do
              export replicas=$(kubectl get deployments \
              --selector=app=dashboard,deploy=$tag \
              -o jsonpath --template="{.items[0].status.replicas}")
              export ready=$(kubectl get deployments \
              --selector=app=dashboard,deploy=$tag \
              -o jsonpath --template="{.items[0].status.readyReplicas}")
              echo "total replicas: $replicas, ready replicas: $ready"
              if [ "$ready" -eq "$replicas" ]; then
                echo "tag change and build deployment file by kustomize" 
                kustomize edit add label deploy:$tag -f
                kustomize build . | kubectl apply -f -
                echo "delete $tag deployment"
                kubectl delete deployment --selector=app=dashboard,deploy!=$tag
                kubectl get deployments -o wide
                break
              else
                sleep 1
              fi
            done
            '''
          }
        }
      }                                                          # 95
    }                      
  }
}
```

### 3번째 줄
- 쿠버네티스의 파드를 젠킨스 작업이 수행되는 에이전트로 사용한다.
- kubernetes {} 내부에서는 에이전트로 사용할 파드에 대한 명세를 야믈 형태로 정의할 수 있다.

### 4~26번째 줄
- 젠킨스의 에이전트로 만들어지는 파드의 명세이다.
- 해당 야믈은 블루그린 배포를 위해 필요한 kustomize가 호스트에 설치되어 있지 않아도 사용할 수 있도록 kustomize가 설치된 컨테이너를 에이전트 파드에 포함한다.
- 또한, 호스트에 설치된 kubectl 명령어를 사용하기 위해 호스트와 연결된 볼륨, 에이전트 파드가 쿠버네티스 클러스터에 오브젝트를 배포하기 위해 사용할 서비스 어카운트인 jenkins가 미리 설정되어 있다.

### 29~34번째 줄
- 깃허브로부터 대시보드 소스 코드를 내려받는 단계이다.

### 35~45번째 줄
- 젠킨스 빌드 횟수마다 부여되는 번호에 따라 블루와 그린이 전환되는 것을 구현하기 위해 젠킨스 스크립트(script)를 사용한다.
- 젠킨스 빌드 번호가 홀수일 때 tag 환경변수값을 blue로 설정하고, 짝수일 때는 green으로 설정한다.

### 46~64번째 줄
- 대시보드를 배포하기 위해 필요한 ConfigMap을 배포한 후 디플로이먼트를 배포하는 단계이다.

### 65~95번째 줄
- 블루그린 배포 전략을 위한 디플로이먼트 배포가 끝난 후 쿠버네티스 클러스터 외부로부터 들어온 요청을 로드밸런서에서 보내줄 대상을 다시 설정하는 단계이다.
- 로드밸런서 설정에 필요한 야믈 파일이 깃허브 저장소 하위 service 디렉터리에 위치해 있기 때문에 dir('service') 작업으로 service 디렉터리로 이동해 작업을 수행한다.
- service의 selector 값들을 명령으로 처리하기 위해 kustomize 명령을 사용한다.
















