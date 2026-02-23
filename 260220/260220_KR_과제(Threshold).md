# 임계치 설계 연습
- 마케팅 이벤트 설계를 위해 서비스 동시접속 임계치와 처리 한계 기준이 필요합니다.
- 단순 "몇 명까지 버틴다" 수준이 아니라, 아래 관점에서 정리 부탁드립니다.
## 1. 동접 기준 임계치
- 현재 인프라/아키텍처에서 안정적으로 감당 가능한 동접 범위
- 장애/지연이 발생하기 시작하는 구간
### 1) 각 컴포넌트의 하드 리밋 확인(이론적 상한선)
- 현재 운영중인 컴포넌트의 이론적인 한계치를 파악해야 한다.
- AWS Service Quotas 콘솔에서 현재 계정의 리밋을 확인할 수 있습니다.
ALB: 초당 새 연결 수
EC2/ECS: vCPU 수, 메모리, 네트워크 대역폭(인스턴스 타입 확인)
RDS/Aurora: 인스턴스 메모리 기반으로 자동 계산되는 최대 연결 수(max_connections)
ElastiCache: maxmemory, maxclients(연결수)
Lambda: 리전 별 동시 실행 수(기본 1,000)
API Gateway: 기본 10,000 RPS(계정 단위)
### 2) Cloudwatch 메트릭으로 현재 운영 상태 파악
#### 2.1 ALB
- 메트릭 종류: https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html
ActiveConnectionCount: 현재 동시 연결 수
TargetResponseTime: ALB가 요청을 전송한 후, 타겟 서버가 응답 헤더를 전송하기 시작하기 까지 걸린 시간
HTTPCode_Target_5XX_Count: 로드 밸런서에서 생성된 HTTP 5XX 서버 오류 코드 수
#### 2.2 EC2/ECS
- 메트릭 종류: https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/viewing_metrics_with_cloudwatch.html
CPUUtilization: CPU 사용 백분율(통상 70% 넘으면 응답 지연 시작)
MemoryUtilization: 클러스터 또는 서비스에서 사용 중인 메모리의 비율 (ECS는 기본 제공, EC2는 CW Agent 필요)
NetworkIn/Out: 네트워크 포화 여부(In: 수신한 바이트, Out: 송신한 바이트)
#### 2.3 RDS/Aurora
- 메트릭 종류: https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/rds-metrics.html
DatabaseConnections: max_connections 대비 현재 사용량
CPUUtilization: CPU 사용 백분율(통상 70% 넘으면 응답 지연 시작)
ReadLatency / WriteLatency: 지연이 튀기 시작하는 구간
FreeableMemory: 사용 가능한 RAM 용량
#### 2.4 ElastiCache
- 메트릭 종류: https://docs.aws.amazon.com/ko_kr/AmazonElastiCache/latest/dg/CacheMetrics.Memcached.html
CurrConnections: 동시에 캐시에 연결된 수
Evictions: 0보다 커지면 메모리 한계 도달 신호
CacheHitRate: 떨어지기 시작하면 DB 부하 급증

### 3) 부하 테스트로 실측
- 메트릭 확인만으로는 "장애가 시작되는 구간"을 정확히 알기 어려우므로 실제로 부하를 올려봐야 합니다.
#### 3.1 방법
- 도구: k6, Locust, Artillery 등으로 동접 수를 단계적으로 증가
- 패턴: 100 → 500 → 1,000 → 2,000 동접으로 계단식 증가
- 관찰 포인트: 응답시간(P95/P99)이 튀기 시작하는 지점, 에러율이 올라가는 지점
- AWS에서 확인:
    + 부하 테스트 중 CloudWatch 대시보드를 실시간으로 보면서 어느 컴포넌트가 먼저 반응하는지 확인
    + X-Ray 트레이스로 어느 구간에서 레이턴시가 늘어나는지 추적
## 2. 처리량 기준
* 분당 요청 처리량(RPM), 트랜잭션 처리량(TPM)
* 병목 지점 (DB, 외부 API, 큐, 네트워크 등)
### 2.1 분/트랜잭션 처리량
#### 1) ALB(진입점 기준 RPM 확인)
RequestCount: 전체 요청 수(1분 단위로 보면 RPM)
TargetResponseTime: ALB가 요청을 전송한 후, 타겟 서버가 응답 헤더를 전송하기 시작하기 까지 걸린 시간(처리량 증가 시 응답시간 변화 추이)
#### 2) API Gateway
- 메트릭 종류: https://docs.aws.amazon.com/ko_kr/apigateway/latest/developerguide/api-gateway-metrics-and-dimensions.html
Count: 지정된 기간의 총 API 요청 수
Latency / IntegrationLatency: API Gateway가 클라이언트에서 요청을 수신할 때부터 클라이언트에게 응답을 반환할 때까지의 시간(내부 처리 시간 vs 전체 시간 비교)
#### 3) RDS/Aurora
- 메트릭 종류: https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html
Queries (Aurora 전용): 초당 쿼리 수, 60 곱하면 QPM
CommitLatency / CommitThroughput: 엔진 및 스토리지가 커밋 작업을 완료하는 데 걸린 평균 시간 / 초당 커밋 작업의 평균 수
DMLLatency (Insert/Update/Delete 각각): 삽입, 업데이트 및 삭제의 평균 소요 시간
### 2.2 병목 지점별 처리량
#### 1) Database
DatabaseConnections: 커넥션 풀 고갈 여부
ReadLatency / WriteLatency: 지연이 튀는 구간
DiskQueueDepth: 디스크 I/O 대기열, 0보다 지속적으로 크면 I/O 병목
FreeStorageSpace: 스토리지 부족도 처리량 저하 원인 중 하나
#### 2) 외부 API 병목
- AWS에서 직접 메트릭을 제공하지 않으므로, 아래 방법으로 간접 측정
    + X-Ray Trace: 외부 호출 구간의 레이턴시 세그먼트 확인
    + CloudWatch Custom Metrics로 외부 API 응답시간을 앱에서 직접 발행
        * Timeout, ConnectionError 카운트를 앱 레벨에서 메트릭화(Applicatino에서 발생하는 예외 순간을 CloudWatch로 전송)
#### 3) 큐 병목(SQS)
- 메트릭 지표: https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-available-cloudwatch-metrics.html
ApproximateAgeOfOldestMessage: 대기열에서 가장 오래된 메시지의 수명(가장 중요한 지표, 값이 커지면 소비 속도 < 생산 속도)
NumberOfMessagesSent vs NumberOfMessagesDeleted: 생산(대기열에 성공적으로 추가된 메시지 수)/소비(대기열에서 성공적으로 삭제된 메시지 수) 처리량 비교
ApproximateNumberOfMessagesVisible: 큐 적체량
## 3. SLO 관점
- 응답시간, 에러율 기준으로 봤을 때 "정상 서비스"로 정의 가능한 범위
- 이벤트 트래픽 유입 시 유지 가능한 서비스 레벨
### 3.1 정상 서비스를 수치로 정의하는 방법
#### 1) 응답시간 기준
- 평균은 극단값 때문에 사용자 경험이 희석되므로 사용 비추천
- P95/P99를 기준으로
    + P50: 전체의 50%가 이 시간 내에 응답(일반적인 응답)
    + P95: 전체의 95%가 이 시간 내에 응답(대부분의 사용자 경험)
    + P99: 전체의 99%가 이 시간 내에 응답(SLO의 일반적인 기준)
- ALB의 TargetResponseTime을 P99로 사용 가능
#### 2) 에러율 기준
- 에러율: 5xx 응답 수 / 전체 응답 수 * 100
- ALB에서 HTTPCode_Target_5XX_Count / RequestCount 로 계산
- 보통 0.1% ~ 1% 를 SLO 기준으로 설정하는 경우가 많음
- 4xx는 클라이언트 오류이므로 SLO에서 제외하는 게 일반적
### 3.2 이벤트 트래픽 유입 시 유지 가능한 서비스 레벨
#### 1) 확인 사항
- 현재 아키텍처의 사용 수준 확인
- 부하 테스트로 SLO 한계 수치 실측
#### 2) 가능 전략
- Auto Scaling 트리거를 평소보다 낮게 조정 (CPU 70% → 50%)
- RDS Connection Pool 여유 확보, Read Replica 추가
- ALB Warm-up 사전 요청 (대용량 트래픽 예고 시 AWS에 사전 신청 가능)
- Throttling: SLO 위반이 예상되는 시점에 일부 요청을 429로 반환 → 전체 서비스 보호
- Queue 버퍼링: 즉시 처리 대신 SQS에 적재 후 순차 처리 → 응답시간 SLO 완화
- Circuit Breaker: 외부 API 지연 시 빠르게 fallback → 에러율 SLO 보호
## 4. 모니터링 현황
### 4.1 현재 어떤 지표를 수집하고 있는지
- AWS 기본 제공 메트릭
    + 별도의 설정 불필요
    + ALB: RequestCount, TargetResponseTime, 5XX Count
    + EC2: CPUUtilization, NetworkIn/Out
    + RDS: CPUUtilization, DatabaseConnections, ReadLatency
    + SQS: ApproximateAgeOfOldestMessage, NumberOfMessagesVisible
    + Lambda: Invocations, Duration, Errors, Throttles
- 추가 설정이 필요한 메트릭
    + EC2 메모리 사용률: CloudWatch Agent 설치
    + ECS 컨테이너 메모리: Container Insights 활성화
    + RDS Slow QUery: Parameter Group에서 slow_query_log 활성화 필요
    + 외부 API 응답시간/에러: Appolication 코드 수준에서 제작 필요
- 로그 수집 필요
    + ALB Access Log -> S3로 수집 중인지
    + Application log -> CloudWatch Logs로 수집 중인지
    + X-Ray 트레이싱 -> 활성화 여부 확인
### 4.2 동접/처리량 기준 의사결정이 가능한 수준인지
#### 1) 대시보드 관점
- 동접자 수를 실시간으로 한 눈에 볼 수 있는가?
- 레이어별(ALB > WAS > DB) 메트릭이 한 화면에 있는가?
- P99 응답시간이 대시보드에 표시되고 있는가?
- 에러율이 % 단위로 계산되어 표시되는가?
#### 2) 알람 관점
- 임계치 초과 시 알람이 울리는가?
- Warning / Critical 단계가 구분되어 있는가?
- 알람이 너무 많아서 노이즈가 된 상태는 아닌가?
- 알람 > 담당자 호출까지 연결되어 있는가?
#### 3) 판단 가능성 관점
- 지금 동접이 몇 명인지 즉시 알 수 있는가?
- 현재 RPM이 임계치의 몇 %인지 알 수 있는가?
- 장애 발생 시 어느 컴포넌트 문제인지 5분 내 좁힐 수 있는가?