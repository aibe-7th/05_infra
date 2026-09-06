# 경보 자동화와 클라우드 모니터링 Alertmanager GrafanaCloud 실습 가이드

> 💡 **[핵심 원리]** Prometheus Alerting Rules로 임계치 초과 및 서비스 다운을 자동 감지하고, Alertmanager를 통해 중복 제거·그룹핑하여 Slack/Email로 실시간 전송합니다. 사내망 인바운드 포트(8080, 9090)를 일절 개방하지 않고, Prometheus Remote Write를 활용하여 HTTPS(TCP 443) 단방향 아웃바운드로 글로벌 SaaS인 Grafana Cloud에 지표를 실시간 복제하여 안전한 하이브리드 관제를 실현합니다.

---

## 1. 실습 개요 및 목표

### 1.1 실습 개요
본 실습은 장애 감지의 자동화와 클라우드 기반 원격 통합 관제를 완성하는 단계입니다. Prometheus의 Alerting Rules(`alert_rules.yml`)를 선언하여 애플리케이션 서비스 다운(`InstanceDown`) 및 JVM 힙 메모리 임계치(`HighJvmMemoryUsage`) 감지 규칙을 구성합니다. Prometheus가 감지한 알림은 전용 라우팅 엔진인 Alertmanager(`alertmanager.yml`)로 전달되며, 알림 폭탄(Alert Fatigue)을 방지하기 위한 그룹핑(`group_wait`, `group_interval`, `repeat_interval`)을 거쳐 Slack Incoming Webhook 및 Email로 발송됩니다.

동시에 사내망 방화벽의 인바운드 포트를 열지 않는 강력한 보안 아키텍처를 위해 **Prometheus Remote Write** 파이프라인을 구축합니다. 로컬 컨테이너의 Prometheus가 수집한 모든 시계열 데이터를 HTTPS(TCP 443) 단방향 아웃바운드 터널을 통해 글로벌 클라우드 모니터링 SaaS인 Grafana Cloud로 실시간 전송하고, 로컬 서비스 중단 및 정상 복구 시계열을 클라우드 콘솔에서 확인합니다.

```mermaid
flowchart TB
    subgraph OnPremise["사내망 / 로컬 호스트 (인바운드 포트 8080/9090 완전 차단)"]
        App["spring-app (Spring Boot)\n/actuator/prometheus"]
        Prometheus["prometheus (포트 9090)\n규칙 평가 & Remote Write 전송"]
        Alertmgr["alertmanager (포트 9093)\n중복 제거, 그룹핑, 라우팅"]

        App -->|Pull 스크랩 (10s)| Prometheus
        Prometheus -->|경보 발생 통지 (Inactive -> Pending -> Firing)| Alertmgr
    end

    subgraph ExternalChannels["외부 수신 채널"]
        Slack["Slack 채널\nIncoming Webhook 수신"]
        Email["Gmail 메일함\nSMTP 통지 수신"]
        Alertmgr -->|HTTPS POST Webhook| Slack
        Alertmgr -->|TLS SMTP (587)| Email
    end

    subgraph CloudSaaS["Grafana Cloud 글로벌 SaaS"]
        RemoteEndpoint["Mimir / Cortex 메트릭 수집기\n(Remote Write Endpoint)"]
        CloudDash["Grafana Cloud 원격 대시보드\n(Explore / Alerts)"]
        RemoteEndpoint --> CloudDash
    end

    Prometheus ==>|"HTTPS (TCP 443) 단방향 아웃바운드 푸시
(TLS 암호화 Remote Write)"| RemoteEndpoint
```

### 1.2 실습 목표
- **Grafana Cloud 사전 연동**: Grafana Cloud 계정을 생성하고 조직 포털의 `[Details]` 경로를 통해 Remote Write 토큰을 발급받아 환경변수에 저장한다.
- **Prometheus Alerting Rules 정의**: `InstanceDown` 및 `HighJvmMemoryUsage` 규칙을 작성하고, 오탐 방지를 위한 `for` 유예 기간을 구성한다.
- **Alertmanager 라우팅 및 그룹핑 튜닝**: Slack Incoming Webhook 및 Email 리시버를 정의하고, 알림 폭탄 방지 파라미터(`group_wait`, `group_interval`, `repeat_interval`)를 설정한다.
- **Remote Write 파이프라인 구축**: 사내망 인바운드 개방 없이 HTTPS 단방향 아웃바운드로 Grafana Cloud SaaS에 메트릭을 실시간 전송한다.
- **장애 및 복구 생명주기 검증**: 스프링부트 컨테이너를 강제 중지하여 `Inactive` → `Pending` → `Firing` 전이 과정을 관측하고 Slack 실시간 경보와 정상 복구 시 `[RESOLVED]` 알림을 검증한다.
- **클라우드 원격 관제 시각화**: Grafana Cloud 콘솔에서 로컬에서 발생시킨 장애 및 복구 시계열이 실시간 수집되었는지 확인한다.

---

## 2. 실습 환경 및 준비

### 2.1 실습 환경 요약
- **작업 디렉터리**: `~/workspace/simple-back`
- **모니터링 컨테이너 스택**:
  - `spring-app`: Spring Boot 애플리케이션 (`8080:8080`)
  - `prometheus`: 메트릭 수집, 경보 규칙 지속 평가 및 Remote Write 전송기 (`9090:9090`)
  - `alertmanager`: 경보 중복 제거/그룹핑 및 라우팅 엔진 (`9093:9093`)
  - `grafana`: 로컬 통합 대시보드 (`3000:3000`)
- **외부 연동 채널**:
  - Grafana Cloud SaaS (Prometheus Remote Write HTTPS 엔드포인트)
  - Slack Incoming Webhook
  - Gmail SMTP 및 앱 비밀번호
- **가상 네트워크**: `monitoring-net`

---

## 3. 핵심 실습 절차 (Step-by-Step)

### Step 1. Grafana Cloud 계정 생성 및 Remote Write 토큰 발급 (.env 등록)

Grafana Cloud 인스턴스 초기화(프로비저닝)에는 수 분이 소요되므로 실습 맨 처음에 계정을 생성하고 전송 자격증명을 발급받아 환경변수에 저장합니다. 인스턴스가 준비되는 동안 로컬 경보 규칙 및 Alertmanager 설정을 구성합니다.

1. 웹 브라우저에서 Grafana Cloud(`https://grafana.com/products/cloud/`)에 접속하여 영구 무료(Forever Free) 계정을 생성하고 로그인합니다.
2. 조직 포털 홈(`grafana.com/orgs/<조직명>`)에서 본인 스택 카드의 **[Details]** 버튼을 클릭하여 스택 상세 화면으로 진입합니다.
3. 스택 서비스 목록 중 **Prometheus** 카드의 **Send Metrics**를 클릭합니다.
4. 화면에 표시되는 원격 전송 정보를 확인하고 API 토큰(`Generate Token`)을 발급받아 메모합니다:
   - `Remote Write Endpoint URL`
   - `Username / Instance ID`
   - `API Token / Password`
5. 프로젝트 루트의 `.env` 파일에 Grafana Cloud 자격증명을 추가합니다:

```bash
cd ~/workspace/simple-back

# 루트 .env 파일에 Grafana Cloud 원격 전송 자격증명 등록
cat <<'EOF' >> .env

# Grafana Cloud Remote Write 자격증명
GRAFANA_CLOUD_REMOTE_WRITE_URL=https://prometheus-prod-XX-prod-XX.grafana.net/api/prom/push
GRAFANA_CLOUD_INSTANCE_ID=123456
GRAFANA_CLOUD_API_KEY=glc_ey...
EOF

# 키 등록 확인 (비밀번호 노출 없이 키 목록만 확인)
grep "^GRAFANA_CLOUD_" .env
```

> 💡 **[핵심 원리]** Grafana Cloud 인스턴스가 백그라운드에서 초기화되는 동안 Step 2~6의 경보 설정을 연속 진행함으로써 수업 대기 시간을 완전히 제거합니다. 외부 SaaS 연동 정보는 소스코드에 하드코딩하지 않고 `.env` 파일에 격리 보관합니다.

### Step 2. Prometheus Alerting Rules 작성: 인스턴스 다운 및 힙 메모리 임계치

시스템 장애나 서비스 다운을 자동으로 감지하기 위한 경보 규칙 파일(`prometheus/alert_rules.yml`)을 작성합니다.

```bash
cd ~/workspace/simple-back
mkdir -p prometheus alertmanager

cat <<'EOF' > prometheus/alert_rules.yml
groups:
  - name: spring-boot-alerts
    rules:
      # 1. 애플리케이션 다운 감지 (1분 이상 인스턴스가 응답하지 않을 때)
      - alert: InstanceDown
        expr: up{job="spring-actuator"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "스프링부트 인스턴스 다운 감지 (Instance: {{ $labels.instance }})"
          description: "스프링 애플리케이션 컨테이너가 1분 이상 응답하지 않고 있습니다."

      # 2. 고부하 JVM 메모리 사용률 경보 (힙 사용률 85% 초과 시)
      - alert: HighJvmMemoryUsage
        expr: (jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "JVM 힙 메모리 임계치 초과 (Instance: {{ $labels.instance }})"
          description: "JVM 힙 메모리 사용량이 85%를 초과하여 유지되고 있습니다."
EOF
cat prometheus/alert_rules.yml
```

> 💡 **[핵심 원리]**
> - `up{job="spring-actuator"} == 0`: Actuator 스크랩 실패 상태를 감지합니다.
> - `for: 1m`: 일시적인 네트워크 지연이나 GC 멈춤에 의한 오탐(False Positive)을 방지하기 위해 1분간 대기(`Pending`) 상태를 유지합니다.

### Step 3. Alerting Rule PromQL 조건식 및 Pending 주기 검증

작성된 경보 규칙의 구문과 라벨, 임계치 표현식이 유효한지 점검합니다.

```bash
grep -E "alert:|expr:|for:|severity:" prometheus/alert_rules.yml
```

> 📌 **[기대 출력 확인]**
> `InstanceDown`, `HighJvmMemoryUsage` 두 개의 알림 규칙과 `for: 1m`, `for: 2m` 유예 기간이 올바른 들여쓰기로 출력됩니다.

### Step 4. Alertmanager 글로벌 설정 및 Slack Webhook 리시버 정의

Alertmanager의 기본 라우팅 및 Slack 수신 채널을 정의하는 `alertmanager/alertmanager.yml`을 작성합니다. 민감 정보 하드코딩을 방지하기 위해 루트 `.env`에 정의된 환경변수를 로드하여 동적으로 생성합니다.

```bash
# 1. 루트 .env에 저장된 외부 알림 자격증명 로드
set -a
source .env
set +a

# 2. alertmanager.yml 생성
cat <<EOF > alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: '${GMAIL_USER}'
  smtp_auth_username: '${GMAIL_USER}'
  smtp_auth_password: '${GMAIL_APP_PASS}'

route:
  group_by: ['alertname', 'severity']
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 1h
  receiver: 'slack-notifications'

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'
        send_resolved: true
        title: '[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}'
        text: >-
          *경보 이름*: {{ .CommonLabels.alertname }}
          *심각도*: {{ .CommonLabels.severity }}
          *상세 내용*: {{ .CommonAnnotations.description }}
EOF
cat alertmanager/alertmanager.yml
```

> ⚠️ **[주의 사항]** 최신 Slack Incoming Webhook 사양에서는 `channel: '#monitoring-alerts'` 설정을 **반드시 생략**해야 합니다! 최신 Slack Webhook URL은 앱 생성 시 특정 채널에 고정 바인딩되어 발급되므로, YAML에서 다른 채널로 오버라이드를 시도하면 Slack API가 `404 channel_not_found` 오류를 반환하며 알림 발송이 실패합니다.

### Step 5. Alertmanager 알림 그룹핑, 대기 시간 및 복구 알림 튜닝

알림 폭탄(Alert Fatigue)을 방지하는 그룹핑 파라미터 설정을 점검합니다.

```bash
grep -A 6 "^route:" alertmanager/alertmanager.yml
```

> 💡 **[핵심 원리]**
> - `group_wait: 10s`: 최초 경보 발생 시 연관된 추가 알림을 하나로 병합하기 위해 10초간 대기합니다.
> - `group_interval: 1m`: 동일 그룹에 새로운 경보가 추가되었을 때 알림을 묶어서 전송하는 최소 주기입니다.
> - `repeat_interval: 1h`: 장애가 지속되고 있을 때 동일 알림의 재전송 주기(1시간)입니다.
> - `send_resolved: true`: 서비스가 정상 복구되었을 때 `[RESOLVED]` 녹색 복구 알림을 자동 발송합니다.

### Step 6. Alertmanager Email 발송 설정 및 문법 검증

이메일 통지를 위한 보조 리시버를 추가하고 파일 구조를 확인합니다.

```bash
cat <<EOF >> alertmanager/alertmanager.yml

  - name: 'email-notifications'
    email_configs:
      - to: '${GMAIL_USER}'
        send_resolved: true
EOF
cat alertmanager/alertmanager.yml
```

### Step 7. Prometheus 설정에 Alertmanager 타깃 및 Remote Write 파이프라인 등록

Prometheus가 Alertmanager로 경보를 전달하고 Grafana Cloud로 시계열 지표를 원격 전송하도록 `prometheus/prometheus.yml`을 일괄 구성합니다. Step 1에서 `.env`에 저장한 클라우드 자격증명을 로드하여 동적 주입합니다.

```bash
# 1. 루트 .env 환경변수 로드
set -a
source .env
set +a

# 2. prometheus.yml 생성 (Alertmanager + Remote Write 동시 반영)
cat <<EOF > prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'spring-actuator'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    static_configs:
      - targets: ['app:8080']

remote_write:
  - url: "${GRAFANA_CLOUD_REMOTE_WRITE_URL}"
    basic_auth:
      username: "${GRAFANA_CLOUD_INSTANCE_ID}"
      password: "${GRAFANA_CLOUD_API_KEY}"
EOF
cat prometheus/prometheus.yml
```

> 💡 **[핵심 원리]**
> - `rule_files: ["alert_rules.yml"]`: 주기적으로 평가할 경보 규칙 파일을 등록합니다.
> - `alerting.alertmanagers`: 경보 발생 시 통지할 대상 Alertmanager 컨테이너 내부 주소(`alertmanager:9093`)를 지정합니다.
> - `remote_write`: 로컬에 수집된 시계열 데이터를 HTTPS(TCP 443) 단방향 아웃바운드로 Grafana Cloud SaaS에 전송합니다. 사내망 인바운드 방화벽을 전혀 열지 않고도 안전한 하이브리드 관제를 실현합니다.

### Step 8. compose.yaml에 Alertmanager 서비스 추가 및 전체 스택 기동

멀티 컨테이너 환경에 `alertmanager` 서비스를 추가하고 전체 모니터링 스택을 데몬으로 구동합니다. 기동과 동시에 Prometheus가 Alertmanager와 연결되고 Grafana Cloud로 Remote Write 전송이 즉시 개시됩니다.

```bash
cat <<'EOF' > compose.yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: spring-app
    restart: on-failure
    ports:
      - "8080:8080"
    volumes:
      - ./logs:/app/logs
    networks:
      - monitoring-net

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
    networks:
      - monitoring-net

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: always
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    networks:
      - monitoring-net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - monitoring-net

networks:
  monitoring-net:
EOF

docker compose up -d
docker compose ps
```

> 📌 **[기대 출력 확인]**
> `spring-app`, `prometheus`, `alertmanager`, `grafana` 4개 서비스가 모두 `Up` 상태여야 합니다.

### Step 9. Prometheus 콘솔에서 Alert Rule, Alertmanager 및 Remote Write 상태 확인

Prometheus 웹 인터페이스에서 경보 규칙, Alertmanager 연동 및 원격 전송 상태를 점검합니다.

```bash
curl -s http://localhost:9090/-/ready
curl -s http://localhost:9093/-/ready
docker compose logs --tail 40 prometheus | grep -i "remote_write"
```

1. 브라우저에서 `http://localhost:9090/alerts` 접속
2. `InstanceDown` 규칙이 초록색 **Inactive** 상태로 등록되어 있는지 확인
3. 상단 메뉴 **Status > Alertmanagers**에서 `http://alertmanager:9093/api/v2/alerts` 활성화 확인
4. Prometheus 컨테이너 로그에서 `Starting WAL watcher`가 출력되고 `401 Unauthorized` 오류 없이 Remote Write가 활성화되었는지 확인

### Step 10. 백엔드 서비스 강제 중단 및 Prometheus Pending/Firing 전이 관측

스프링부트 컨테이너를 강제 중단시켜 경보 상태가 전이되는 과정을 관측합니다.

```bash
# 백엔드 애플리케이션 강제 중단
docker compose stop app
docker compose ps
```

1. `http://localhost:9090/alerts` 새로고침:
   - 10~15초 후: `up == 0`이 감지되어 노란색 **Pending** 상태로 변경됨
   - 1분(`for: 1m`) 경과 후: 유예 대기 시간이 충족되어 빨간색 **Firing** 상태로 전이됨
2. 터미널에서 경보 API 응답 조회:
   ```bash
   curl -s http://localhost:9090/api/v1/alerts | grep -o '"state":"[^"]*"'
   ```

### Step 11. Alertmanager 대시보드 확인 및 Slack/Email 실시간 경보 수신

Alertmanager로 전달된 경보 내역과 웹훅 발송 상태를 점검합니다.

1. Alertmanager 대시보드(`http://localhost:9093`) 접속
2. `InstanceDown` 활성 경보 카드 확인
3. Slack 채널에서 수신된 실시간 경보 확인:
   - `[FIRING] 스프링부트 인스턴스 다운 감지` 메시지 확인
4. Alertmanager 컨테이너 로그 확인:
   ```bash
   docker compose logs --tail 30 alertmanager | grep -i "notify"
   ```

### Step 12. 백엔드 서비스 정상 복구 및 [RESOLVED] 알림 수신 검증

중단했던 애플리케이션 컨테이너를 다시 시작하여 정상 복구 절차를 검증합니다.

```bash
# 백엔드 애플리케이션 재시작
docker compose start app
docker compose ps
```

1. 10~15초 후 `http://localhost:9090/alerts`에서 상태가 초록색 **Inactive**로 자동 복귀됨을 확인
2. Slack 채널에 `[RESOLVED] 스프링부트 인스턴스 다운 감지` 복구 알림이 도착하는지 확인합니다.

### Step 13. Grafana Cloud Explore에서 내장 데이터 소스 연동 확인

Step 1에서 계정을 생성한 후 로컬 실습을 진행하는 동안 Grafana Cloud 인스턴스 프로비저닝이 백그라운드에서 완료되었으므로 대기 없이 즉시 콘솔에 진입합니다.

1. Grafana Cloud 대시보드(`https://<조직명>.grafana.net`)에 접속합니다.
2. 좌측 메뉴에서 **Explore**를 클릭합니다.
3. 데이터 소스 선택 드롭다운에서 클라우드 내장 Prometheus 데이터 소스(`grafanacloud-<조직명>-prom`)를 선택합니다.

### Step 14. 실시간 지표 쿼리 실행 및 원격 시각화 검증

로컬에서 Remote Write로 전송된 메트릭을 클라우드 대시보드에서 조회하고, 방금 발생했던 장애 및 복구 시계열을 확인합니다.

1. 메트릭 쿼리 입력창에 `up{job="spring-actuator"}`를 입력하고 **Run query**를 클릭합니다:
   - 애플리케이션이 실행되던 상태(`1`) → 강제 중단 구간(`0`) → 복구된 상태(`1`)가 시계열 그래프에 뚜렷한 골(Trough) 형태로 기록되어 있음을 시각적으로 확인합니다.
2. 추가로 `jvm_memory_used_bytes`를 조회하여 Spring Boot 애플리케이션의 메모리 지표가 사내망 인바운드 포트 개방 없이 클라우드에 실시간 전송되고 있음을 검증합니다.

---

## 4. 실무 트러블슈팅 가이드

### Issue 1: Slack Webhook 전송 시 404 channel_not_found 오류
- **증상**: Alertmanager 로그에 `level=error msg="Cancelling notify retry" err="cancelling notify retry for "slack"[0] due to unrecoverable error: unexpected status code 404: channel_not_found"` 발생하며 Slack 알림 미수신.
- **원인**: 최신 Slack Incoming Webhook URL은 앱 생성 시 특정 채널에 고정 바인딩되어 발급됩니다. `alertmanager.yml`에서 `channel: '#monitoring-alerts'`처럼 채널명을 명시하면 Slack API가 채널 오버라이드 권한을 거부하며 404 오류를 반환합니다.
- **해결책**: `alertmanager.yml`의 `slack_configs` 블록에서 `channel` 옵션을 생략해야 합니다. 생략 시 웹훅 생성 시 지정된 기본 채널로 정상 발송됩니다.

### Issue 2: Grafana Cloud Remote Write 401 Unauthorized 인증 실패
- **증상**: Prometheus 로그에 `remote_write: send error: server returned HTTP status 401 Unauthorized` 에러가 반복 기록됨.
- **원인**: `.env`에 입력된 `GRAFANA_CLOUD_API_KEY` 토큰이 만료되었거나, `GRAFANA_CLOUD_INSTANCE_ID` 또는 Remote Write URL에 공백/오타가 포함된 경우.
- **해결책**: Grafana Cloud 조직 포털 홈(`grafana.com/orgs/<조직명>`)에서 본인 스택의 **[Details]** 버튼을 클릭하고 **Send Metrics** 화면에서 발급받은 Instance ID와 신규 생성한 API 토큰 값을 `.env`에 정확히 갱신한 후 `docker compose restart prometheus`로 재기동합니다.

### Issue 3: 서비스 중지 직후 Alertmanager로 경보가 즉시 가지 않는 현상
- **증상**: `docker compose stop app`을 실행했으나 수 초 동안 Slack 알림이 오지 않음.
- **원인**: 정상 동작입니다. `alert_rules.yml`에 `for: 1m` 유예 기간이 설정되어 있어 1분 동안 `Pending` 상태를 유지한 후 장애가 지속될 때만 `Firing`으로 전이되어 알림을 발송합니다.
- **해결책**: 일시적인 네트워크 지연이나 Full GC 멈춤에 의한 오탐(False Positive)을 방지하는 필수 안전장치임을 이해합니다.

### Issue 4: 외부 SaaS 연동 시 사내 방화벽 인바운드 포트 개방 위험
- **증상**: 보안팀에서 외부 SaaS 연동을 위해 사내망 인바운드 포트(8080, 9090) 개방을 금지하여 모니터링 연동이 불가능해 보임.
- **원인**: 전통적인 모니터링 방식(외부에서 내부로 들어와 데이터를 긁어가는 Inbound Pull)은 인바운드 방화벽 포트 개방이 불가피합니다.
- **해결책**: Prometheus Remote Write는 사내망 내부의 Prometheus가 외부 SaaS로 **HTTPS(TCP 443) 단방향 아웃바운드 푸시**를 수행하므로 인바운드 포트를 전혀 열지 않고도 완벽한 사설망 보안을 유지할 수 있습니다.

---


## 5. 실습 자원 정리 (Cleanup)

실습이 끝난 후 모든 모니터링 컨테이너와 볼륨을 완전히 정리합니다.

```bash
# 1. 전체 컨테이너 중지 및 볼륨 일괄 회수
docker compose down -v

# 2. 잔여 컨테이너 및 볼륨 확인
docker ps -a
docker volume ls | grep simple-back
```

> 📌 **[기대 출력 확인]**
> `docker ps -a` 실행 결과 컨테이너가 남아있지 않아야 합니다.

---

## 6. Appendix: Docker Compose 컨테이너 메모리 제한과 JVM 70~75% 예산 배분 원칙

컨테이너 메모리 제한이 없으면 JVM 힙 메모리 누수나 대용량 트래픽 스파이크 시 단일 컨테이너가 호스트 전체 메모리를 고갈시켜 다른 필수 컨테이너(DB, Prometheus 등)가 호스트 커널 OOM Killer에 의해 강제 종료될 위험이 있습니다.

### 7.1 Docker Compose v2 메모리 제한 문법
Docker Compose v2 사양(`compose.yaml`)에서는 `deploy.resources.limits.memory` 또는 축약형 `mem_limit`을 통해 컨테이너당 최대 메모리 상한을 강제합니다:

```yaml
services:
  app:
    build: .
    container_name: spring-app
    # 방법 1: Compose 표준 사양 (권장)
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: '1.0'
        reservations:
          memory: 256m
    # 방법 2: 단순 개발 환경 축약형
    # mem_limit: 512m
```

### 7.2 JVM 힙(-Xmx)과 컨테이너 메모리 한도의 예산 배분
- **컨테이너 한도 = JVM 힙 + Non-Heap + Native OS 버퍼**: JVM 프로세스는 힙(`Heap`) 메모리 외에도 클래스 메타데이터를 저장하는 메타스페이스(`Metaspace`), 쓰레드당 할당되는 쓰레드 스택(`Thread Stack`, 기본 1MB * 쓰레드 수), C/C++ 네이티브 메모리 및 GC 오버헤드를 함께 소비합니다.
- **1:1 매핑 금지 이유**: 컨테이너 메모리 제한이 `512MB`일 때 JVM 최대 힙(`-Xmx`)을 동일하게 `512m`로 부여하면, 비-힙 메모리가 더해져 컨테이너 메모리가 512MB를 초과하는 즉시 리눅스 커널 cgroups OOM Killer에 의해 컨테이너 프로세스가 강제 종료(`Exit Code 137`)됩니다.
- **실무 권장 배분율**: JVM 최대 힙(`-Xmx`)은 반드시 컨테이너 메모리 한도의 **70~75%** 수준으로 설정해야 합니다:
  - 컨테이너 메모리 제한 `512MB` 기준: `JAVA_TOOL_OPTIONS="-Xms256m -Xmx384m"` (384MB = 75%)
  - 컨테이너 메모리 제한 `1GB` 기준: `JAVA_TOOL_OPTIONS="-Xms512m -Xmx768m"` (768MB = 75%)
- **컨테이너 자동 인식 플래그 (Java 10+)**: 최신 JDK 17은 cgroups 한도를 자동 인식하므로 `-XX:MaxRAMPercentage=75.0` 옵션을 지정하면 컨테이너 제한에 맞춰 JVM 힙 상한을 75%로 자동 동적 계산합니다.
