# 서비스 디스커버리
## cAdvisor
### 1) 쿼리 입력기에 container_memory_usage_bytes 입력 후 Execute 버튼
- 3개의 출력되는 결과가 존재한다.
- Load time : PromQL이 동작하는데 걸린 시간
- Resolution : 수집된 데이터로 지정된 초 단위의 그래프를 그린다.
- Total time series : PromQL로 수집된 결과의 개수

### 2) 메트릭 수집 확인
- PromQL에 추가할 파드의 이름으로 검색하도록 {container="nginx"} 추가 (이와 같은 구문을 레이블이라고 함)
- nginx 디플로이먼트가 설치되기 전이므로 no data이다.

### 3) nginx 배포
```bash
[root@m-k8s ~]# kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
```

### 4) 메트릭 수집 재확인
- nginx 디플로이먼트에 대한 메트릭이 자동으로 수집되는 것을 확인할 수 있음
- 메트릭이 자동으로 수집되는 것은 컨피그맵에 수집 대상이 지정되었기 때문이다.

```bash
[root@m-k8s ~]# kubectl get configmap prometheus-server -o yaml | nl
    56        job_name: kubernetes-nodes-cadvisor
    57        kubernetes_sd_configs:
    58        - role: node
    59        relabel_configs:
    60        - action: labelmap
    61          regex: __meta_kubernetes_node_label_(.+)
    62        - replacement: kubernetes.default.svc:443
    63          target_label: __address__
    64        - regex: (.+)
    65          replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
    66          source_labels:
    67          - __meta_kubernetes_node_name
    68          target_label: __metrics_path__
```

### 5) 디플로이먼트 삭제 후 메트릭 수집 재확인
- 디플로이먼트를 삭제하면 더 이상 메트릭이 수집되지 않음을 확인할 수 있다.
- 메트릭 자체가 지워진 것이 아닌 웹 UI에서 쿼리로 검색되지 않을 뿐이다.
- 이처럼, cAdvisor는 각 노드에 컨테이너 시스템 정보를 담고 있는 특정 파일들을 읽어들여 메트릭을 수집하고, 프로메테우스가 수집할 수 있도록 컨테이너 메트릭을 프로메테우스 메트릭으로 변환해 공개한다.


## 익스포터
> 사전 준비 작업 필요 : API 서버 등록(경로 알 수 있게), 익스포터가 데이터를 프로메테우스 타입으로 노출해야 함

### 1) API 서버에 등록될 구성이 포함된 deployment 배포
```bash
[root@m-k8s ~]# kubectl apply -f ~/_Book_k8sInfra/ch6/6.2.3/nginx-status-annot.yaml
deployment.apps/nginx created
```

### 2) 애너테이션 설정 확인
- 프로메테우스 서버가 배포된 애플리케이션을 API 서버에서 찾아 매트릭을 수집하기 위해서는 애너테이션 설정이 가장 중요하다.
- 이는 매니페스트에 적용되어 있다.
```bash
[root@m-k8s ~]# cat ~/_Book_k8sInfra/ch6/6.2.3/nginx-status-annot.yaml | nl
    13        annotations:
    14          prometheus.io/port: "80"
    15          prometheus.io/scrape: "true"
```

#### 애너테이션으로 매트릭 수집
1. 프로메테우스에 메트릭을 공개할 애플리케이션은 애너테이션이 매니페스트에 추가되어 있어야 한다. 이때 애너테이션에 추가되는 구문은 prometheus.io/로 시작한다.
2. 작성된 매니페스트를 쿠버네티스 클러스터에 배포해 애너테이션을 포함한 정보를 API 서버에 등록한다.
3. 프로메테우스 서버가 prometheus.io/로 시작하는 애너테이션 정보를 기준으로 대상 주소를 만든다.
4. 프로메테우스 서버가 애너테이션을 기준으로 만든 대상의 주소로 요청을 보내 메트릭 데이터를 수집한다.


### 3) API 서버를 통해 배포된 nginx 디플로이먼트 정보가 프로메테우스 서버에 등록되었는지 확인
- nginx 디플로이먼트는 등록되었으나 메트릭이 수집되지 않는다.
- 메트릭이 수집되지 않는 것은 메트릭이 공개되지 않았기 때문이다.
- 두 가지 방법으로 메트릭을 공개할 수 있다.
  - 프로그래밍 언어의 프로메테우스 SDK를 사용해 직접 메트릭을 공개하도록 작성
  - 이미 만들어 둔 익스포터를 활용해 메트릭을 공개하는 방법
 
### 4) deployment 재배포
```bash
[root@m-k8s ~]# kubectl apply -f ~/_Book_k8sInfra/ch6/6.2.3/nginx-status-metrics.yaml
deployment.apps/nginx configured
```
- NGINX에서 제공하는 nginx-prometheus-exporter를 추가로 구성한 야믈 파일을 배포한다.
- 해당 익스포터는 멀티 컨테이너 패턴 중 하나인 사이드카 패턴이다.

















