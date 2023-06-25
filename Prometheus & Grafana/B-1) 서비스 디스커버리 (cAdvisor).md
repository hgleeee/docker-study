# 서비스 디스커버리(cAdvisor)
### 1) 쿼리 입력기에 container_memory_usage_bytes 입력 후 Execute 버튼
- 3개의 출력되는 결과가 존재한다.
- Load time : PromQL이 동작하는데 걸린 시간
- Resolution : 수집된 데이터로 지정된 초 단위의 그래프를 그린다.
- Total time series : PromQL로 수집된 결과의 개수

### 2) 디플로이먼트를 추가하면 자동으로 메트릭을 수집하는지 확인
