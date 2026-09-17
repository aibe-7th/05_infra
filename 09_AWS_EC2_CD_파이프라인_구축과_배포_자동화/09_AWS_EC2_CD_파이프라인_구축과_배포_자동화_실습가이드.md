# AWS EC2 CD 파이프라인 구축과 배포 자동화 실습 가이드

> 💡 **[핵심 원리]**
> 지속적 통합(CI)이 "합쳐도 되는 코드인가"를 판정한다면, 지속적 배포(CD)는 "검증된 이미지를 서비스 중단 없이 운영 서버에 올리는" 단계입니다. 이 차시에서는 단일 EC2 인스턴스 안에 Blue(`8081`)와 Green(`8082`) 두 컨테이너 자리를 두고, Nginx가 `include`로 읽는 한 줄짜리 설정 조각(`service-url.inc`)만 바꿔 `nginx -s reload`로 트래픽 대상을 교체합니다. 새 컨테이너가 L7 헬스체크를 통과하기 전까지 기존 컨테이너가 계속 응답하므로 전환 순간에도 요청이 끊기지 않습니다.
> 배포 트리거는 두 방식을 모두 다룹니다. **Push 방식**은 GitHub Actions 러너가 CI 성공 직후(`workflow_run`) EC2에 SSH로 접속해 `deploy.sh`를 실행하고, **Pull 방식**은 EC2 안에 상주하는 데몬 컨테이너(Watchtower)가 도커 소켓을 통해 레지스트리의 새 이미지를 감지해 교체합니다. 마지막으로 배포 결과를 Slack Incoming Webhook 카드로 전송해 Git Push부터 알림까지 사람의 개입 없이 이어지는 E2E 파이프라인을 완성합니다.

---

## 1. 실습 개요 및 목표

### 1.1 실습 개요
선행 차시에서 중지해 둔 EC2 인스턴스를 재시작하고, 배포 대상 저장소(`simple-back-ghcr`)에 테스트 게이트 CI와 SSH 기반 CD를 차례로 연결합니다.

1. **Blue-Green 기반 구성**: EC2에 Nginx 동적 프록시 설정, `~/.env`, `compose.blue-green.yaml`을 작성하고 Blue 스택을 기동한 뒤, 헬스체크와 프록시 전환을 수행하는 `deploy.sh`로 수동 Blue → Green 전환을 검증합니다.
2. **CI → CD 체이닝**: 테스트 없이 이미지를 발행하던 `docker-publish.yml`을 테스트 게이트 `ci.yml`로 교체하고, CI 성공 시에만 실행되는 `cd.yml`을 연결합니다. 이미지 발행 Job은 ARM 네이티브 러너(`ubuntu-24.04-arm`)와 GitHub Actions 빌드 캐시로 빌드 시간을 줄입니다.
3. **Pull 방식 체험**: Watchtower를 폴링 모드와 웹훅 모드로 차례로 구동해 두 트리거 경로의 차이를 확인하고, Push 파이프라인과 충돌하지 않도록 제거합니다.
4. **알림과 E2E 검증**: `cd.yml`에 Slack 알림 스텝을 붙이고, 애플리케이션 메시지를 `v2.0`으로 바꿔 푸시한 뒤 0.5초 간격 트래픽 감시 루프로 무중단 전환을 관찰합니다.

```mermaid
flowchart LR
    Dev["로컬 터미널<br/>git push"] --> CI["Continuous Integration & Package<br/>test → docker-build-push"]
    CI -->|linux/arm64 이미지| GHCR[("GHCR<br/>latest · SHA 태그")]
    CI -->|workflow_run success| CD["CD Pipeline<br/>appleboy/ssh-action"]

    subgraph EC2["EC2 t4g.small (Ubuntu ARM)"]
        Deploy["deploy.sh"]
        Nginx["nginx-proxy :80<br/>include service-url.inc"]
        Blue["app-blue :8081"]
        Green["app-green :8082"]
        DB[("mysql-db")]
        WT["Watchtower<br/>(Pull 실습 후 제거)"]
    end

    CD -->|SSH 22| Deploy
    Deploy -->|pull · up -d · 헬스체크| Green
    Deploy -->|service-url.inc 교체 · reload| Nginx
    Nginx --> Blue
    Nginx --> Green
    Blue --> DB
    Green --> DB
    GHCR -.->|다이제스트 감시| WT
    CD -->|Webhook POST| Slack["Slack 채널<br/>성공/실패 카드"]
    User["트래픽 감시 루프"] -->|HTTP 80| Nginx
```

### 1.2 실습 목표
- **단일 인스턴스 Blue-Green 무중단 배포**: Nginx 동적 `include`와 Blue/Green 포트 격리 스택을 구성하고, L7 헬스체크 후 `nginx -s reload`로 전환하는 `deploy.sh`를 구현한다.
- **테스트 게이트 CI와 CD 체이닝**: 배포 대상 저장소에 `needs: test` 기반 `ci.yml`을 도입하고, `workflow_run`으로 CI 성공 시에만 SSH 배포를 실행하는 `cd.yml`을 연결한다.
- **Pull 방식 데몬 컨테이너 운용**: 도커 소켓을 마운트한 Watchtower를 폴링·웹훅 모드로 구동해 트리거 차이를 확인하고 안전하게 제거한다.
- **배포 결과 알림과 E2E 관찰**: Slack Incoming Webhook 카드로 배포 결과를 전송하고, 코드 변경 푸시부터 전환까지 오류 응답 0건의 무중단 전환을 직접 확인한다.
- **보안·비용 통제**: 러너용 SSH 임시 허용 규칙을 실습 종료 시 회수하고 인스턴스를 중지한다.

---

## 2. 실습 환경 및 준비

### 2.1 실습 환경 요약

| 구분 | 내용 |
| --- | --- |
| AWS 자원 | 선행 차시에서 중지해 둔 EC2 인스턴스(`${STUDENT_ID}-managed-ec2` 또는 `${STUDENT_ID}-compose-ec2`, Ubuntu ARM `t4g.small`), 웹 보안 그룹(`${STUDENT_ID}-web-sg`), 키 페어(`${STUDENT_ID}-key.pem`) |
| 이번 차시 신규 자원 | 웹 보안 그룹의 러너용 22번 임시 허용 규칙(`RUNNER_SSH_RULE_ID`, 당일 회수), EC2의 `~/.env`·`~/nginx.conf`·`~/service-url.inc`·`~/compose.blue-green.yaml`·`~/deploy.sh`, 저장소의 `ci.yml`·`cd.yml`, Slack 앱 `Deploy-Bot` |
| 작업 디렉터리 | 로컬 AWS CLI·SSH는 `~/workspace/aws-lab`(키 페어 `.pem` 보관 위치), Git 작업은 배포 대상 저장소 클론 `~/workspace/simple-back` |
| CLI 환경변수 | `STUDENT_ID` 하나에서 `AWS_PROFILE`·`MY_KEY_NAME`·`MY_SG_NAME`·`MY_INSTANCE_NAME`을 파생, `AWS_REGION="ap-northeast-2"`·`AWS_PAGER=""` 병행 선언. Step 1·2에서 `INSTANCE_ID`·`PUBLIC_IP`·`MY_SG_ID`·`RUNNER_SSH_RULE_ID`를 설정 |
| 원격 명령 권한 | `ubuntu` 계정은 `docker` 그룹 비소속. EC2 안의 모든 Docker 명령을 `sudo docker`·`sudo docker compose`로 실행 |
| GitHub 저장소 | `simple-back-ghcr`(공개 권장), Repository Secrets `EC2_HOST`·`EC2_USER`·`EC2_SSH_KEY`·`SLACK_WEBHOOK_URL` |
| CI/CD 러너 | 테스트 Job·CD Job은 `ubuntu-latest`, 이미지 발행 Job은 `ubuntu-24.04-arm`(공개 저장소 무료) |
| 컨테이너 이미지 | `ghcr.io/<본인 GitHub 아이디>/simple-back-ghcr:latest`, `nginx:alpine`, `mysql:8.0`, `containrrr/watchtower` |
| 메신저 | Slack 워크스페이스와 Incoming Webhook URL |

### 2.2 사전 준비 확인
1. **재시작할 인스턴스가 남아 있는지 확인합니다.** `07-3`을 끝까지 정리했다면 인스턴스가 `terminate`되어 Step 1이 실패합니다. 이 경우 `07-2`의 인스턴스 생성·Docker 설치 절차로 `$MY_INSTANCE_NAME` 인스턴스를 먼저 만듭니다(4절 Issue 1).
2. **로컬 키 파일과 AWS 키 페어가 같은 키인지 확인합니다.** 대상은 `.pem` 파일입니다. AWS 쪽 키 페어가 재발급되었다면 SSH가 `Permission denied (publickey)`로 실패합니다(4절 Issue 2).
3. **현재 공인 IP가 22번 규칙에 등록되어 있는지 확인합니다.** 학습자 규칙은 `/32` 단일 IP입니다. 러너용 전체 허용 규칙은 Step 2에서 별도로 추가하고 5절에서 회수합니다.
4. **배포 대상 저장소 클론을 확인합니다.** `~/workspace/simple-back`이 본인의 `simple-back-ghcr` 원격을 가리키고, `.github/workflows/docker-publish.yml`과 멀티 스테이지 `Dockerfile`이 있어야 합니다(`git remote -v`, `ls .github/workflows`).
5. **GHCR 패키지 공개 여부를 확인합니다.** 공개 패키지는 EC2와 Watchtower가 인증 없이 pull 합니다. 비공개라면 `read:packages` 권한의 PAT(Personal Access Token, 개인용 액세스 토큰)가 필요합니다.
6. **Slack 워크스페이스에 앱을 추가할 권한이 있는지 확인합니다.** Webhook URL은 비밀 값이므로 파일·대화에 남기지 않고 GitHub Secrets에만 등록합니다.

---

## 3. 핵심 실습 절차 (Step-by-Step)

### Step 1. EC2 인스턴스 재시작 및 가동 상태 확인
선행 차시에서 중지해 둔 인스턴스를 재시작하고 새로 할당된 공인 IP를 조회합니다. 모든 자원 이름은 배부받은 `STUDENT_ID` 하나에서 파생합니다.

```bash
# 1. 실습 식별자와 공통 환경변수 (본인 값으로 바꾸는 곳은 첫 줄뿐이다)
export STUDENT_ID="student01"
export AWS_PROFILE="$STUDENT_ID"
export AWS_REGION="ap-northeast-2"
export AWS_PAGER=""

export MY_KEY_NAME="${STUDENT_ID}-key"
export MY_SG_NAME="${STUDENT_ID}-web-sg"
# 07-3에서 생성·중지해 둔 managed-ec2 인스턴스 (또는 07-2 compose-ec2)
export MY_INSTANCE_NAME="${STUDENT_ID}-managed-ec2"
# export MY_INSTANCE_NAME="${STUDENT_ID}-compose-ec2"  # 07-2 인스턴스를 유지한 경우

cd ~/workspace/aws-lab          # 키 페어(.pem)를 발급해 둔 실습 작업 디렉터리

# 2. 이름 태그와 중지 상태를 함께 필터해 대상 인스턴스 ID 조회
export INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=$MY_INSTANCE_NAME" "Name=instance-state-name,Values=stopped" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
echo "실습에 사용할 인스턴스 ID: $INSTANCE_ID"

# 3. 인스턴스 재시작 및 2/2 상태 검사 통과 대기
aws ec2 start-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"

# 4. 재시작으로 새로 할당된 공인 IP 조회
export PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "새로 할당된 공인 IP: $PUBLIC_IP"
```

- **검증 포인트**: `INSTANCE_ID`가 `i-`로 시작하고, 재시작 전과 다른 공인 IP가 출력됩니다. 두 대기 명령은 합쳐서 약 2~3분이 걸립니다.
- **조회 결과가 비었을 때**: 상태 필터를 빼고 현재 상태부터 확인합니다. `running`이면 재시작을 건너뛰고 공인 IP 조회부터 진행하고, 결과 자체가 비어 있으면 4절 Issue 1을 따릅니다.
  `aws ec2 describe-instances --filters "Name=tag:Name,Values=$MY_INSTANCE_NAME" --query "Reservations[].Instances[].[InstanceId,State.Name]" --output table`

### Step 2. 원격 SSH 접속 점검 및 GitHub Secrets 자격 증명 등록
SSH 접속을 점검하고, 러너가 EC2에 접속할 수 있도록 보안 그룹 규칙과 GitHub Secrets를 준비합니다.

```bash
# 1. EC2 SSH 접속 정상 여부 사전 점검
ssh -i ./"$MY_KEY_NAME".pem -o StrictHostKeyChecking=accept-new ubuntu@"$PUBLIC_IP" \
  "sudo docker compose version"

# 2. macOS/Linux 클립보드에 프라이빗 키 전문 복사
pbcopy < ./"$MY_KEY_NAME".pem 2>/dev/null || xclip -selection clipboard < ./"$MY_KEY_NAME".pem 2>/dev/null || cat ./"$MY_KEY_NAME".pem
echo "SSH 키가 복사되었습니다. GitHub Secrets의 EC2_SSH_KEY 값에 붙여넣으세요."

# 3. GitHub Actions 러너의 SSH 접속 허용 (실습 중 임시 개방, Step 18에서 회수)
export MY_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=$MY_SG_NAME" \
  --query "SecurityGroups[0].GroupId" --output text)
export RUNNER_SSH_RULE_ID=$(aws ec2 authorize-security-group-ingress \
  --group-id "$MY_SG_ID" --protocol tcp --port 22 --cidr 0.0.0.0/0 \
  --query "SecurityGroupRules[0].SecurityGroupRuleId" --output text)
echo "러너 SSH 임시 허용 규칙 ID: $RUNNER_SSH_RULE_ID"
```

- **GitHub 웹 저장소 설정**: 본인의 `simple-back-ghcr` 저장소에서 `Settings` → `Secrets and variables` → `Actions` → `New repository secret`으로 3건을 등록합니다.
  - `EC2_HOST`: 위에서 출력된 `$PUBLIC_IP` 값
  - `EC2_USER`: `ubuntu`
  - `EC2_SSH_KEY`: 복사한 프라이빗 키 전문(`-----BEGIN ...-----`부터 `-----END ...-----`까지 개행 포함)
- **검증 포인트**: SSH 점검에서 `Docker Compose version v...`가 출력되고, 규칙 ID가 `sgr-`로 시작합니다.
- **러너 SSH 허용이 필요한 이유**: GitHub 호스팅 러너는 실행마다 다른 공인 IP를 쓰므로, 학습자 IP(`/32`)만 허용된 22번 포트로는 접속하지 못해 CD가 실패합니다(4절 Issue 6). 전체 허용은 키 인증만 허용되는 전제의 실습용 임시 조치이며, 실무에서는 22번 포트를 열지 않는 SSM Session Manager나 Self-hosted Runner를 씁니다.
- **주의**: `MY_SG_ID`와 `RUNNER_SSH_RULE_ID`는 5절 정리에서 다시 씁니다. 새 터미널을 열었다면 정리 전에 두 값을 다시 설정해야 합니다. 인스턴스를 재시작하면 공인 IP가 바뀌므로 `EC2_HOST` 시크릿도 함께 갱신합니다.

### Step 3. Nginx 동적 리버스 프록시 및 Blue/Green Compose 스택 정의
EC2에 접속해 프록시 설정 조각, Nginx 메인 설정, Compose 보간용 `~/.env`, Blue/Green 스택 정의를 작성합니다.

```bash
# 1. EC2 원격 서버 접속
ssh -i ./"$MY_KEY_NAME".pem ubuntu@"$PUBLIC_IP"

# ─── (원격 EC2 터미널 내부 실행) ───
cd ~/
export GITHUB_USERNAME="본인_깃허브_아이디"

# 2. 초기 Nginx 동적 프록시 include 설정 파일 생성 (초기값: app-blue:8080)
cat <<'EOF' > ~/service-url.inc
proxy_pass http://app-blue:8080;
EOF

# 3. Nginx 메인 설정 파일 작성 (service-url.inc 포함)
cat <<'EOF' > ~/nginx.conf
events {
    worker_connections 1024;
}

http {
    server {
        listen 80;

        location / {
            include /etc/nginx/conf.d/service-url.inc;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
EOF

# 4. Compose 보간·컨테이너 주입용 ~/.env 작성 (실습 전용 값, 저장소에 커밋하지 않음)
cat <<'EOF' > ~/.env
DB_NAME=mydb
DB_USER=myuser
DB_PASSWORD=mypassword
DB_ROOT_PASSWORD=rootpassword
MYSQL_USER=myuser
MYSQL_PASSWORD=mypassword
SPRING_DATASOURCE_HIKARI_INITIALIZATIONFAILTIMEOUT=-1
EOF

# 5. Blue/Green 지원 compose.blue-green.yaml 작성
cat <<EOF > ~/compose.blue-green.yaml
services:
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./service-url.inc:/etc/nginx/conf.d/service-url.inc:ro
    depends_on:
      - app-blue
    deploy:
      resources:
        limits:
          memory: 64M
    networks:
      - frontend-net

  app-blue:
    image: ghcr.io/${GITHUB_USERNAME}/simple-back-ghcr:latest
    container_name: app-blue
    restart: on-failure
    ports:
      - "8081:8080"
    env_file:
      - .env
    environment:
      PORT: 8080
      JAVA_TOOL_OPTIONS: "-XX:MaxRAMPercentage=75.0"
      SPRING_DATASOURCE_URL: "jdbc:mysql://db:3306/\${DB_NAME}?useSSL=false&allowPublicKeyRetrieval=true"
      SPRING_DATASOURCE_USERNAME: \${DB_USER}
      SPRING_DATASOURCE_PASSWORD: \${DB_PASSWORD}
    depends_on:
      - db
    deploy:
      resources:
        limits:
          memory: 896M
    networks:
      - frontend-net
      - backend-net

  app-green:
    image: ghcr.io/${GITHUB_USERNAME}/simple-back-ghcr:latest
    container_name: app-green
    restart: on-failure
    ports:
      - "8082:8080"
    env_file:
      - .env
    environment:
      PORT: 8080
      JAVA_TOOL_OPTIONS: "-XX:MaxRAMPercentage=75.0"
      SPRING_DATASOURCE_URL: "jdbc:mysql://db:3306/\${DB_NAME}?useSSL=false&allowPublicKeyRetrieval=true"
      SPRING_DATASOURCE_USERNAME: \${DB_USER}
      SPRING_DATASOURCE_PASSWORD: \${DB_PASSWORD}
    depends_on:
      - db
    deploy:
      resources:
        limits:
          memory: 896M
    networks:
      - frontend-net
      - backend-net

  db:
    image: mysql:8.0
    container_name: mysql-db
    restart: always
    env_file:
      - .env
    environment:
      MYSQL_ROOT_PASSWORD: \${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: \${DB_NAME}
    deploy:
      resources:
        limits:
          memory: 512M
    networks:
      - backend-net

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
EOF
```

- **검증 포인트**: `cat ~/service-url.inc`가 `proxy_pass http://app-blue:8080;`를 출력하고, `grep -n image ~/compose.blue-green.yaml`에서 `${GITHUB_USERNAME}` 자리가 본인 아이디로 치환되어 있어야 합니다.
- **heredoc 따옴표 차이**: 따옴표 없는 `<<EOF`는 작성 시점에 `${GITHUB_USERNAME}`을 치환하고, `\${DB_USER}`처럼 역슬래시를 붙인 값은 그대로 남겨 Compose가 `~/.env`로 보간하게 합니다.
- **환경 파일이 필요한 이유**: Compose 파일과 같은 디렉터리의 `~/.env`가 `DB_*` 값을 채우고, `env_file`로 `db` 컨테이너에 `MYSQL_USER`·`MYSQL_PASSWORD`를 주입해 애플리케이션 계정을 만듭니다. 이 파일이 없으면 기동 자체가 거부됩니다(4절 Issue 3). 값을 확인할 때는 `cut -d= -f1 ~/.env`로 키만 봅니다.

### Step 4. 초기 Compose 스택 기동 및 컨테이너 헬스 점검
이전 차시 스택을 정리하고 `db`, `nginx`, `app-blue` 3개 서비스로 초기 Blue 상태를 만듭니다. 본 차시는 EC2 내부 `db` 컨테이너를 함께 기동해 독립적으로 진행합니다.

```bash
# ─── (원격 EC2 터미널 내부 실행) ───

# 1. 기존 07-2/07-3 단일 스택 컨테이너 안전 정리
sudo docker compose down 2>/dev/null || true

# 2. Blue 초기 스택 기동 (db, nginx, app-blue)
sudo docker compose -f ~/compose.blue-green.yaml up -d db nginx app-blue

# 3. DB 초기화 및 스프링부트 컨테이너 기동 대기 (10초)
sleep 10

# 4. 포트별 서비스 응답 점검
echo "Nginx 80번 진입점 응답:"
curl -s http://localhost/
echo ""
echo "app-blue 8081번 직접 호출 응답:"
curl -s http://localhost:8081/
echo ""

# 5. 가동 중인 컨테이너 상태 점검
sudo docker compose -f ~/compose.blue-green.yaml ps
```

- **기대 결과**: 컨테이너 3개(`mysql-db`, `app-blue`, `nginx-proxy`)가 `Started`로 기동하고, 응답이 준비되면 두 호출 모두 아래 형식의 JSON을 반환합니다.
  ```text
  {"status":"UP","message":"Hello Spring Boot 4 Infra Practice!","timestamp":"..."}
  ```
- **주의**: `sleep 10` 시점에는 MySQL 초기화가 끝나지 않아 `502 Bad Gateway`나 빈 응답이 나올 수 있습니다. `restart: on-failure`로 애플리케이션이 자동 재기동되므로 20~30초 뒤 다시 호출합니다(4절 Issue 4). 기동 로그는 `sudo docker logs -f --tail 30 app-blue`로 확인합니다.

### Step 5. Blue/Green 무중단 배포 스크립트(deploy.sh) 작성 및 수동 전환 검증
활성 대상을 감지해 반대편 컨테이너를 기동하고, 헬스체크 통과 후 프록시를 전환하는 `deploy.sh`를 작성해 수동으로 실행합니다.

```bash
# ─── (원격 EC2 터미널 내부 실행) ───

# 1. deploy.sh 배포 자동화 스크립트 작성
cat <<'EOF' > ~/deploy.sh
#!/usr/bin/env bash
set -euo pipefail

echo "=================================================="
echo "🚀 [CD] Blue/Green 무중단 배포 시작: $(date '+%Y-%m-%d %H:%M:%S')"
echo "=================================================="

cd ~/

# 1. service-url.inc 점검 및 현재 활성 포트 감지
if [ ! -f ~/service-url.inc ]; then
  echo "proxy_pass http://app-blue:8080;" > ~/service-url.inc
fi

CURRENT_URL=$(cat ~/service-url.inc)

if echo "$CURRENT_URL" | grep -q "app-blue"; then
  CURRENT_SERVICE="app-blue"
  CURRENT_PORT=8081
  TARGET_SERVICE="app-green"
  TARGET_PORT=8082
else
  CURRENT_SERVICE="app-green"
  CURRENT_PORT=8082
  TARGET_SERVICE="app-blue"
  TARGET_PORT=8081
fi

echo "현재 활성 서비스: $CURRENT_SERVICE (포트: $CURRENT_PORT)"
echo "신규 배포 대상: $TARGET_SERVICE (포트: $TARGET_PORT)"

# 2. 신규 배포 대상 최신 이미지 Pull
echo "📥 1. GHCR 최신 이미지 다운로드 중..."
sudo docker compose -f compose.blue-green.yaml pull "$TARGET_SERVICE"

# 3. 비활성 컨테이너 기동
echo "🔄 2. 신규 컨테이너($TARGET_SERVICE) 백그라운드 기동 중..."
sudo docker compose -f compose.blue-green.yaml up -d "$TARGET_SERVICE"

# 4. 신규 컨테이너 헬스체크 폴링 (최대 10회, 3초 간격)
echo "🔍 3. 신규 컨테이너 헬스체크 시작 (http://localhost:$TARGET_PORT/)..."
HEALTH_CHECK_SUCCESS=false

for i in $(seq 1 10); do
  echo "헬스체크 시도 ($i/10)..."
  STATUS_CODE=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:$TARGET_PORT/" || true)
  if [ "$STATUS_CODE" -eq 200 ]; then
    echo "✅ 헬스체크 통과! (HTTP status: $STATUS_CODE)"
    HEALTH_CHECK_SUCCESS=true
    break
  fi
  sleep 3
done

if [ "$HEALTH_CHECK_SUCCESS" = false ]; then
  echo "🚨 헬스체크 실패! 신규 컨테이너($TARGET_SERVICE)를 즉시 중단하고 롤백합니다."
  sudo docker compose -f compose.blue-green.yaml stop "$TARGET_SERVICE"
  exit 1
fi

# 5. Nginx 프록시 대상 스위칭 및 무중단 리로드
echo "🔀 4. Nginx 프록시 대상 전환 -> $TARGET_SERVICE"
echo "proxy_pass http://$TARGET_SERVICE:8080;" > ~/service-url.inc
sudo docker exec nginx-proxy nginx -s reload

# 6. 이전 컨테이너 중지
echo "🛑 5. 이전 컨테이너($CURRENT_SERVICE) 안전 중단..."
sudo docker compose -f compose.blue-green.yaml stop "$CURRENT_SERVICE"

# 7. 불필요한 구버전 미태그(Dangling) 이미지 정리
echo "🧹 6. 불필요한 구버전 이미지 정리 중..."
sudo docker image prune -f

echo "=================================================="
echo "✅ [CD] Blue/Green 배포 완료: $TARGET_SERVICE로 전환 완료"
echo "=================================================="
EOF

# 2. 실행 권한 부여 및 수동 전환 테스트 실행
chmod +x ~/deploy.sh
~/deploy.sh

# 3. 전환 결과 점검 (service-url.inc 내용 및 활성 컨테이너 확인)
echo "현재 service-url.inc 라우팅 대상:"
cat ~/service-url.inc
sudo docker compose -f ~/compose.blue-green.yaml ps -a

# 4. 원격 EC2 세션 종료
exit
```

- **기대 결과**: `헬스체크 통과! (HTTP status: 200)` 후 `Nginx 프록시 대상 전환 -> app-green`이 출력되고, `service-url.inc`가 `proxy_pass http://app-green:8080;`로 바뀝니다. `ps -a`에서 `app-green`은 `Up`, `app-blue`는 `Exited (143)`로 보입니다.
- **주의**: 기본 `ps`는 실행 중인 컨테이너만 보여 주므로, 중지된 이전 컨테이너까지 확인하려면 반드시 `ps -a`를 씁니다.
- **롤백 동작**: 헬스체크가 10회 모두 실패하면 트래픽을 전환하기 전에 신규 컨테이너만 멈추고 `exit 1`로 끝나므로, 기존 컨테이너가 계속 서비스합니다.

### Step 6. 테스트 게이트 CI 도입과 SSH CD 워크플로우(cd.yml) 작성
배포 대상 저장소의 테스트 없는 이미지 발행 워크플로우를 테스트 게이트 CI로 교체하고, CI 성공 시에만 실행되는 CD 워크플로우를 작성합니다.

```bash
cd ~/workspace/simple-back

# 1. 테스트 게이트가 없는 기존 이미지 발행 워크플로우 제거
git rm .github/workflows/docker-publish.yml
# 유일한 파일이 지워지면 Git이 빈 디렉터리도 제거하므로 곧바로 다시 만든다
mkdir -p .github/workflows

# 2. 테스트 통과 시에만 이미지를 발행하는 CI 워크플로우로 교체
cat <<'EOF' > .github/workflows/ci.yml
name: Continuous Integration & Package

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Run Unit & Slice Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Set up JDK 17
        uses: actions/setup-java@v6
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Run tests with Gradle Wrapper
        run: ./gradlew test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v7
        with:
          name: gradle-test-reports
          path: build/reports/tests/test/

  docker-build-push:
    name: Build & Push Docker Image to GHCR
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-24.04-arm
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v4
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Lowercase image name
        run: echo "IMAGE_NAME=${IMAGE_NAME,,}" >> $GITHUB_ENV

      - name: Build and push Docker image
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          platforms: linux/arm64
          cache-from: type=gha
          cache-to: type=gha,mode=max
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
EOF

git add .github/workflows/ci.yml
git commit -m "ci: replace build-only publish workflow with test-gated ci pipeline"
git push origin main

# 3. CD 워크플로우(cd.yml) 생성
cat <<'EOF' > .github/workflows/cd.yml
name: CD Pipeline (Push via SSH)

on:
  workflow_run:
    workflows: ["Continuous Integration & Package"]
    types: [completed]
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    name: Blue-Green Deploy to AWS EC2
    runs-on: ubuntu-latest
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: SSH 원격 접속 및 Blue/Green 무중단 배포 스크립트 실행
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          port: 22
          script: |
            echo "Connected to EC2 via SSH."
            ~/deploy.sh
EOF
```

| 항목 | 설명 |
| --- | --- |
| `mkdir -p .github/workflows` | `git rm`으로 유일한 워크플로우를 지우면 디렉터리도 사라지므로 바로 다시 만든다. 빠뜨리면 `ci.yml` 작성이 실패한 채 삭제만 커밋된다(4절 Issue 5) |
| `needs: test` | `test` Job이 종료 코드 0으로 성공해야만 이미지 발행 Job이 실행된다 |
| `runs-on: ubuntu-24.04-arm` | ARM 네이티브 러너에서 `linux/arm64`를 빌드해 QEMU 에뮬레이션을 없앤다(4절 Issue 9) |
| `cache-from`/`cache-to: type=gha` | Docker 레이어를 GitHub Actions 캐시에 저장해 다음 실행부터 의존성 레이어를 재사용한다 |
| `workflow_run.workflows` | CI 워크플로우의 `name:`과 한 글자라도 다르면 오류 없이 CD가 영구히 트리거되지 않는다 |

- **검증 포인트**: `ci.yml` 푸시 직후 `Actions` 탭에서 `Continuous Integration & Package`가 실행되고, `Run Unit & Slice Tests`가 끝난 뒤에야 `Build & Push Docker Image to GHCR`가 시작됩니다. `cd.yml`은 아직 커밋하지 않은 상태(`git status`에서 `??`)로 둡니다.

### Step 7. CD 워크플로우 원격 저장소 푸시 및 GitHub Actions 트리거
`cd.yml`을 푸시해 CI 완료 후 CD가 자동으로 이어지는지 확인합니다.

```bash
# 1. Git 변경 사항 추가 및 커밋/푸시
git add .github/workflows/cd.yml
git commit -m "ci: add GitHub Actions SSH Blue/Green CD pipeline"
git push origin main
```

- **확인 절차**: `Actions` 탭에서 `Continuous Integration & Package`가 먼저 실행되고, 성공(녹색 체크) 직후 `CD Pipeline (Push via SSH)`가 `workflow_run` 이벤트로 자동 실행되는지 확인합니다.
- **기대 결과**: CD의 `SSH 원격 접속 및 Blue/Green 무중단 배포 스크립트 실행` 스텝 로그에 `Connected to EC2 via SSH.`와 `deploy.sh` 출력이 이어지고, 마지막에 `Successfully executed commands to all hosts.`가 표시됩니다. CD Job 자체는 30초 안팎에 끝납니다.
- **주의**: 로그에 `dial tcp ***:22: i/o timeout`이 보이면 Step 2의 러너 SSH 허용 규칙이 없는 상태입니다(4절 Issue 6). 규칙을 추가한 뒤 실행 화면의 `Re-run jobs`로 다시 실행합니다.

### Step 8. EC2 원격 서버 무중단 전환 상태 및 로그 검증
러너가 실행한 배포 결과를 로컬에서 일회성 SSH 명령과 외부 HTTP 호출로 확인합니다.

```bash
# 1. 로컬 터미널에서 EC2 컨테이너 전환 상태 및 라우팅 점검
ssh -i ./"$MY_KEY_NAME".pem ubuntu@"$PUBLIC_IP" \
  "cat ~/service-url.inc && sudo docker compose -f ~/compose.blue-green.yaml ps"

# 2. Nginx를 통한 서비스 정상 응답 확인
curl -s http://"$PUBLIC_IP"/
echo ""
```

- **기대 결과**: Step 5에서 Green으로 전환했으므로 CD 실행 후에는 `proxy_pass http://app-blue:8080;`로 되돌아가 있고, 외부 호출이 `{"status":"UP",...}` JSON을 반환합니다.
- **주의**: 이 명령의 `ps`는 실행 중인 컨테이너만 보여 줍니다. 중지된 `app-green`까지 보려면 `ps -a`로 바꿔 실행합니다. 배포 과정의 헬스체크 출력은 Actions 실행 화면의 SSH 스텝 로그에서 확인할 수 있습니다.

### Step 9. Watchtower 주기적 폴링(Polling) 모드 구동 및 다이제스트 감시
EC2 안에 상주하며 레지스트리의 새 이미지를 감시하는 Pull 방식 데몬 컨테이너 Watchtower를 30초 폴링 모드로 구동합니다. GHCR 패키지가 비공개라면 `read:packages` 권한만 가진 PAT를 `CR_PAT`에 넣고, 공개 패키지라면 `REPO_USER`·`REPO_PASS` 두 줄 없이도 동작합니다.

```bash
ssh -i ./"$MY_KEY_NAME".pem ubuntu@"$PUBLIC_IP"

# ─── (원격 EC2 터미널 내부 실행) ───
export GITHUB_USERNAME="본인_깃허브_아이디"
export CR_PAT="ghp_실제발급받은토큰문자열"

# 1. Watchtower 폴링 모드 컨테이너 구동 (30초 주기 감시)
sudo docker run -d \
  --name watchtower-poll \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e DOCKER_API_VERSION=1.44 \
  -e REPO_USER="$GITHUB_USERNAME" \
  -e REPO_PASS="$CR_PAT" \
  containrrr/watchtower \
  --interval 30 \
  --cleanup \
  app-blue

# 2. 폴링 동작 로그 확인 (30초마다 GHCR 다이제스트를 조회하는 동작 관찰)
sudo docker logs -f --tail 20 watchtower-poll
# (확인 후 터미널에서 Ctrl+C를 눌러 탈출)
```

| 옵션 / 환경변수 | 설명 |
| --- | --- |
| `-v /var/run/docker.sock:...` | 호스트 Docker Daemon 제어권을 컨테이너에 위임해 형제 컨테이너를 교체할 수 있게 한다 |
| `DOCKER_API_VERSION=1.44` | Watchtower 1.7.1의 기본 API(1.25)를 Docker Engine 29가 거부하므로 지원 버전을 명시한다 |
| `--interval 30` / `--cleanup` | 30초마다 이미지 다이제스트 변경 여부를 질의하고, 교체 후 구버전 이미지를 정리한다 |
| `app-blue` | 호스트 전체가 아닌 지정한 컨테이너만 감시 대상으로 한정한다 |

- **기대 결과**: `sudo docker ps -a --filter name=watchtower`에서 `watchtower-poll`이 `Up`이고, 로그에 `Only checking containers which name matches "app-blue"`와 30초 간격의 `Session done Failed=0 Scanned=1 Updated=0`이 기록됩니다. 레지스트리의 `latest`가 실행 중인 이미지와 같으면 `Updated=0`이 정상입니다.
- **주의**: `docker run`이 종료 코드 0으로 끝나도 컨테이너가 `Restarting` 상태라면 API 버전 문제입니다(4절 Issue 7).

### Step 10. Watchtower HTTP Webhook 모드 구동 및 즉시 갱신 테스트
폴링을 멈추고, 인증 토큰이 있는 HTTP 요청을 받을 때만 갱신하는 웹훅 모드로 전환합니다.

```bash
# ─── (원격 EC2 터미널 내부 실행) ───

# 1. 기존 폴링 모드 컨테이너 정지 및 제거
sudo docker rm -f watchtower-poll

# 2. 8000번 포트로 수신 대기하는 웹훅 모드 Watchtower 구동
sudo docker run -d \
  --name watchtower-webhook \
  --restart unless-stopped \
  -p 8000:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e DOCKER_API_VERSION=1.44 \
  -e REPO_USER="$GITHUB_USERNAME" \
  -e REPO_PASS="$CR_PAT" \
  containrrr/watchtower \
  --http-api-update \
  --http-api-token "watchtower-secret-token" \
  --cleanup \
  app-blue

# 3. EC2 로컬에서 HTTP Webhook 수동 호출 검증
curl -s -X GET -H "Authorization: Bearer watchtower-secret-token" http://localhost:8000/v1/update

# 4. Watchtower 컨테이너 로그에서 즉시 트리거된 이미지 갱신 작업 확인
sudo docker logs --tail 30 watchtower-webhook
```

| 옵션 / 파라미터 | 설명 |
| --- | --- |
| `-p 8000:8080` | Watchtower 내부 웹훅 리스너(8080)를 호스트 8000번 포트에 매핑한다 |
| `--http-api-update` | 주기 실행을 끄고 `/v1/update` 요청을 받을 때만 갱신한다 |
| `--http-api-token` | Bearer 토큰이 없는 호출을 거부한다 |

- **기대 결과**: 로그에 `Periodic runs are not enabled.`, `The HTTP API is enabled at :8080.`, `Updates triggered by HTTP API request.`, `Session done ... Scanned=1`이 순서대로 남습니다.
- **대조 확인**: 토큰 헤더 없이 호출하면 거부됩니다. `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/v1/update` → `401`

### Step 11. Watchtower 컨테이너 정리 및 Push 파이프라인 복귀
Watchtower가 남아 있으면 이후 `deploy.sh`와 이미지 교체 주체가 충돌하므로 반드시 제거합니다.

```bash
# ─── (원격 EC2 터미널 내부 실행) ───

# 1. Watchtower 컨테이너 완전 제거
sudo docker rm -f watchtower-webhook

# 2. EC2 컨테이너 가동 상태 확인 (app, db, nginx만 남은 상태 확인)
sudo docker compose -f ~/compose.blue-green.yaml ps

# 3. 원격 EC2 세션 종료
exit
```

- **기대 결과**: `ps`에 활성 애플리케이션 컨테이너 1개, `mysql-db`, `nginx-proxy`만 남습니다. 종료 전에 `sudo docker ps -a --filter name=watchtower`의 결과가 비어 있는지도 확인합니다.

### Step 12. Slack Incoming Webhook 앱 생성 및 Webhook URL 발급
배포 결과를 받을 Slack 앱 `Deploy-Bot`을 만들고 Webhook URL을 발급합니다.

1. 본인 Slack 워크스페이스에 로그인한 상태에서 `https://api.slack.com/apps`로 이동해 `Create New App` → `From scratch`를 선택합니다.
   - App Name: `Deploy-Bot`
   - Pick a workspace: 본인 실습 워크스페이스 선택 후 `Create App`
2. 좌측 메뉴 `Features` → `Incoming Webhooks`에서 `Activate Incoming Webhooks`를 `On`으로 켭니다.
3. 하단의 `Add New Webhook to Workspace`를 누르고 알림 채널(예: `#general`, `#deploy-notifications`)을 지정해 `Allow`를 누릅니다.
4. 생성된 `Webhook URL`(형식: `https://hooks.slack.com/services/T.../B.../...`)을 `Copy`로 복사합니다.

- **주의**: Webhook URL을 아는 사람은 누구나 해당 채널에 메시지를 보낼 수 있습니다. 파일·스크립트·채팅에 붙여 넣지 말고 Step 13의 GitHub Secrets에만 등록합니다.

### Step 13. GitHub Secrets 등록 및 cd.yml 슬랙 알림 연동
Webhook URL을 시크릿으로 등록하고, 배포 결과에 따라 색이 다른 카드를 보내는 알림 스텝을 `cd.yml`에 추가합니다.

- **시크릿 등록**: 저장소 `Settings` → `Secrets and variables` → `Actions` → `New repository secret`에서 Name `SLACK_WEBHOOK_URL`, Secret에 복사한 URL 전체를 넣고 `Add secret`을 누릅니다.

```bash
cd ~/workspace/simple-back

cat <<'EOF' > .github/workflows/cd.yml
name: CD Pipeline (Push via SSH with Slack Notification)

on:
  workflow_run:
    workflows: ["Continuous Integration & Package"]
    types: [completed]
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    name: Blue-Green Deploy to AWS EC2
    runs-on: ubuntu-latest
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: SSH 원격 접속 및 Blue/Green 무중단 배포 스크립트 실행
        id: ssh_deploy
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          port: 22
          script: |
            ~/deploy.sh

      - name: Slack 배포 결과 알림 전송
        if: always()
        run: |
          STATUS="${{ steps.ssh_deploy.outcome }}"
          if [ "$STATUS" = "success" ]; then
            COLOR="#36a64f"
            TITLE="✅ EC2 Blue/Green 배포 성공 (Deployment Succeeded)"
            TEXT="EC2 단일 인스턴스 Blue/Green 무중단 전환이 성공적으로 완료되었습니다."
          else
            COLOR="#dc3545"
            TITLE="🚨 EC2 Blue/Green 배포 실패 (Deployment Failed)"
            TEXT="EC2 배포 스크립트 실행 또는 헬스체크 검증 중 오류가 발생하여 롤백되었습니다."
          fi

          PAYLOAD=$(cat <<JSON
          {
            "attachments": [
              {
                "color": "$COLOR",
                "title": "$TITLE",
                "text": "$TEXT",
                "fields": [
                  { "title": "Repository", "value": "${{ github.repository }}", "short": true },
                  { "title": "Branch", "value": "${{ github.ref_name }}", "short": true },
                  { "title": "Triggered By", "value": "${{ github.actor }}", "short": true },
                  { "title": "Commit SHA", "value": "\`${{ github.sha }}\`", "short": true }
                ],
                "footer": "AWS EC2 Blue/Green CD Pipeline",
                "ts": $(date +%s)
              }
            ]
          }
          JSON
          )

          curl -H "Content-Type: application/json" \
               -X POST \
               -d "$PAYLOAD" \
               "${{ secrets.SLACK_WEBHOOK_URL }}"
EOF
```

| 설정 키 | 설명 |
| --- | --- |
| `id: ssh_deploy` | 알림 스텝이 배포 스텝의 결과를 참조하기 위한 식별자 |
| `if: always()` | 배포 스텝이 실패해도 알림 스텝을 반드시 실행한다 |
| `steps.ssh_deploy.outcome` | 배포 결과(`success`/`failure`)에 따라 카드 색과 문구를 고른다 |
| `attachments.color` | 성공 초록(`#36a64f`), 실패 빨강(`#dc3545`) 카드 사이드바 |

- **검증 포인트**: `git diff --stat`에서 `cd.yml`만 변경되고, 첫 줄이 `name: CD Pipeline (Push via SSH with Slack Notification)`입니다. 워크플로우 이름을 바꾸면 Actions 목록의 과거 실행 이름도 새 이름으로 표시됩니다.

### Step 14. 슬랙 연동 CD 워크플로우 푸시 및 알림 발송 검증
변경한 `cd.yml`을 푸시해 CI → CD → Slack 알림이 이어지는지 확인합니다.

```bash
# 1. 변경된 cd.yml 파일 커밋 및 원격 저장소 푸시
git add .github/workflows/cd.yml
git commit -m "ci: add Slack deployment result webhook notification"
git push origin main
```

- **확인 절차**:
  1. `Actions`에서 `Continuous Integration & Package` → `CD Pipeline (Push via SSH with Slack Notification)` 순서로 실행되는지 확인합니다.
  2. `Slack 배포 결과 알림 전송` 스텝 로그 마지막 줄에 Slack 응답 `ok`가 찍히는지 확인합니다.
  3. Slack 채널에 Repository, Branch, Triggered By, Commit SHA가 담긴 초록색 `✅ EC2 Blue/Green 배포 성공` 카드가 도착했는지 확인합니다.
- **주의**: 알림 스텝이 `ok` 대신 오류를 출력하면 `SLACK_WEBHOOK_URL` 값이 비었거나 잘못 붙여 넣어진 것입니다. 시크릿을 다시 등록한 뒤 `Re-run jobs`로 재실행합니다.

### Step 15. 로컬 애플리케이션 코드 수정(v2.0) 및 무중단 트래픽 모니터링 가동
응답 메시지 기본값을 `v2.0`으로 바꾸고 로컬 테스트를 통과시킨 뒤, 별도 터미널에서 0.5초 간격 트래픽 감시 루프를 시작합니다.

```bash
cd ~/workspace/simple-back

# 1. 로컬 애플리케이션 응답 메시지 기본값 수정 (v2.0 릴리스)
sed -i.bak 's|message: .*|message: "${APP_MESSAGE:Hello, Zero-Downtime Blue-Green CD v2.0!}"|' \
  src/main/resources/application.yml
rm -f src/main/resources/application.yml.bak

# 2. 변경 내용 확인
grep -n "message:" src/main/resources/application.yml

# 3. 로컬 단위 테스트 사전 검증 (Fail-Fast 원칙)
./gradlew test
```

- **기대 결과**: `grep` 결과가 `message: "${APP_MESSAGE:Hello, Zero-Downtime Blue-Green CD v2.0!}"` 한 줄이고, `./gradlew test`가 `BUILD SUCCESSFUL`로 끝납니다. `-i.bak` 형식이라 macOS와 Linux의 `sed`에서 모두 동작합니다.

이제 **새 터미널 창(창 2)**을 열어 감시 루프를 실행합니다.

```bash
# ─── (새 터미널 창 2: 무중단 트래픽 모니터링 루프) ───
export PUBLIC_IP="터미널1에서_확인한_EC2_공인_IP"

# 0.5초 주기로 Nginx 80번 포트에 지속 요청 전송
while true; do
  curl -s -w " | HTTP %{http_code}\n" http://"$PUBLIC_IP"/
  sleep 0.5
done
```

### Step 16. 전체 E2E CI/CD 파이프라인 트리거 및 무중단 전환 관찰
작업 터미널(창 1)에서 변경을 푸시하고, 창 2에서 전환 순간을 관찰합니다.

```bash
# ─── (기존 작업 터미널 창 1) ───
git add src/main/resources/application.yml
git commit -m "feat: upgrade message to v2.0 for Blue/Green Zero-Downtime verification"
git push origin main
```

- **관찰 요령**: 이미지 빌드와 신규 컨테이너 기동 중에도 창 2에는 구버전 응답이 `HTTP 200`으로 계속 찍히고, `nginx -s reload` 직후 요청부터 `v2.0` 메시지로 바뀝니다.

```text
{"status":"UP","message":"Hello Spring Boot 4 Infra Practice!","timestamp":"..."} | HTTP 200
{"status":"UP","message":"Hello Spring Boot 4 Infra Practice!","timestamp":"..."} | HTTP 200
{"status":"UP","message":"Hello, Zero-Downtime Blue-Green CD v2.0!","timestamp":"..."} | HTTP 200   <-- 무중단 전환 시점
{"status":"UP","message":"Hello, Zero-Downtime Blue-Green CD v2.0!","timestamp":"..."} | HTTP 200
```

- **판정 기준**: 전환 전후 모든 줄이 `HTTP 200`이고, `502 Bad Gateway`나 연결 실패(빈 응답, `HTTP 000`)가 한 건도 없어야 합니다. 전환 이후에는 구버전 메시지가 다시 나타나지 않아야 합니다.
- **소요 시간 참고**: ARM 네이티브 러너 기준 이미지 발행 Job이 약 2분, 푸시부터 전환 완료까지 약 3분 안팎입니다.

### Step 17. 배포 완료 후 EC2 상태 및 슬랙 카드 최종 점검
창 2의 루프를 `Ctrl+C`로 멈추고, 라우팅 상태와 Slack 카드를 대조합니다.

```bash
# 1. 모니터링 터미널(창 2)에서 Ctrl+C를 눌러 루프 종료

# 2. 작업 터미널(창 1)에서 EC2 호스트 최종 상태 조회
ssh -i ./"$MY_KEY_NAME".pem ubuntu@"$PUBLIC_IP" \
  "cat ~/service-url.inc && sudo docker compose -f ~/compose.blue-green.yaml ps"
```

| 파이프라인 단계 | 점검 위치 | 정상 기준 |
| --- | --- | --- |
| CI 테스트 | Actions `Continuous Integration & Package` | 테스트 Job 통과(녹색 체크) |
| 이미지 레지스트리 | GitHub Packages(GHCR) | `latest` 다이제스트 갱신 |
| CD 원격 배포 | Actions `CD Pipeline` | SSH 접속과 `deploy.sh` 헬스체크 통과 |
| 무중단 전환 | 창 2 감시 루프 | 오류 응답 0건으로 `v2.0` 전환 |
| 리버스 프록시 | EC2 `~/service-url.inc` | 새 활성 서비스(`app-green` 또는 `app-blue`)로 갱신 |
| 팀 메신저 알림 | Slack 채널 | 초록색 성공 카드와 Commit SHA 수신 |

### Step 18. 실습 리소스 정리 및 비용 통제
러너용 SSH 임시 허용 규칙을 회수하고 인스턴스를 중지합니다. 상세 절차는 5.1절과 같습니다.

---

## 4. 실무 트러블슈팅 가이드

### Issue 1: Step 1에서 `INSTANCE_ID`가 `None`이고 `InvalidInstanceID.Malformed`로 실패함
- **원인**: `07-3`의 마지막 정리 단계가 인스턴스를 `terminate` 하므로 재시작할 중지 인스턴스가 계정에 남아 있지 않습니다.
- **해결**: 상태 필터 없이 조회해 결과가 비어 있으면 `07-2`의 인스턴스 생성(Step 2)과 Docker 설치(Step 3) 절차로 `$MY_INSTANCE_NAME` 인스턴스를 새로 만든 뒤, `INSTANCE_ID`·`PUBLIC_IP`를 설정하고 이 가이드의 Step 2로 진행합니다.
```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=$MY_INSTANCE_NAME" \
  --query "Reservations[].Instances[].[InstanceId,State.Name]" --output table

# 07-2 절차로 새로 만든 뒤 두 값을 다시 설정
export INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=$MY_INSTANCE_NAME" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
export PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
```

### Issue 2: `Permission denied (publickey)`로 SSH 인증이 거부됨
- **원인**: AWS의 같은 이름 키 페어가 다시 발급되어 로컬 `.pem`과 짝이 맞지 않습니다. 인스턴스는 생성 당시의 공개 키만 신뢰합니다.
- **해결**: 두 지문을 대조하고, 다르면 키를 임의로 삭제·재발급하지 말고 강사에게 확인합니다. 인스턴스를 새로 만드는 경우에는 로컬에 개인 키가 있는 키 페어로 생성합니다.
```bash
aws ec2 describe-key-pairs --key-names "$MY_KEY_NAME" --query "KeyPairs[0].KeyFingerprint" --output text
openssl pkcs8 -in ./"$MY_KEY_NAME".pem -nocrypt -topk8 -outform DER | openssl sha1 -c
ls -l ./"$MY_KEY_NAME".pem    # 권한이 400(-r--------)이어야 함
```

### Issue 3: `env file /home/ubuntu/.env not found`로 Compose 기동이 거부됨
- **증상**: `The "DB_NAME" variable is not set` 경고 4건이 먼저 출력되고 컨테이너가 하나도 생성되지 않습니다.
- **원인**: `compose.blue-green.yaml`이 `env_file: .env`와 `${DB_*}` 보간을 사용하는데 `~/.env`가 없습니다.
- **해결**: Step 3의 4번 블록(`cat <<'EOF' > ~/.env`)을 실행한 뒤 다시 기동합니다.
```bash
cut -d= -f1 ~/.env    # 키 7개만 확인 (값은 출력하지 않음)
sudo docker compose -f ~/compose.blue-green.yaml up -d db nginx app-blue
```

### Issue 4: 첫 기동 직후 `502 Bad Gateway`가 나오고 로그에 `Unable to determine Dialect`가 보임
- **원인**: MySQL 초기화가 끝나기 전에 애플리케이션이 JPA 초기화를 시도해 한 번 종료됩니다. `restart: on-failure` 정책이 자동으로 다시 기동합니다.
- **해결**: 20~30초 기다린 뒤 다시 호출합니다. 계속 실패하면 DB 컨테이너 상태와 애플리케이션 로그를 함께 확인합니다.
```bash
sudo docker logs --tail 30 app-blue
sudo docker compose -f ~/compose.blue-green.yaml ps -a
curl -s -w " | HTTP %{http_code}\n" http://localhost/
```

### Issue 5: `ci.yml: No such file or directory` 후에도 커밋·푸시가 진행되어 워크플로우가 사라짐
- **원인**: `git rm`이 `.github/workflows/`의 유일한 파일을 지우면 Git이 빈 디렉터리도 제거합니다. `ci.yml` 작성과 `git add`가 실패해도 `git commit`은 이미 스테이징된 삭제만으로 성공해, 워크플로우가 하나도 없는 커밋이 푸시되고 Actions도 실행되지 않습니다.
- **해결**: 되돌리지 말고 디렉터리를 다시 만든 뒤 Step 6의 2번 블록(`ci.yml` 작성·커밋·푸시)을 이어서 실행합니다.
```bash
ls .github/workflows 2>/dev/null || echo "디렉터리 없음"
mkdir -p .github/workflows
# Step 6의 2번 블록(cat <<'EOF' > .github/workflows/ci.yml ... git push origin main) 재실행
git log --oneline -3
```

### Issue 6: CD 실행이 `dial tcp ***:22: i/o timeout`으로 약 30초 뒤 실패함
- **원인**: 웹 보안 그룹의 22번 포트가 학습자 IP(`/32`)만 허용해, 실행마다 IP가 바뀌는 GitHub 호스팅 러너의 접속이 차단됩니다. `EC2_HOST`가 재시작 전의 공인 IP로 남아 있어도 같은 오류가 납니다.
- **해결**: Step 2의 러너 SSH 임시 허용 규칙을 추가하고, `EC2_HOST`가 현재 공인 IP인지 확인한 뒤 실패한 실행을 `Re-run jobs`로 다시 실행합니다.
```bash
aws ec2 describe-security-groups --group-ids "$MY_SG_ID" \
  --query "SecurityGroups[0].IpPermissions[?FromPort==\`22\`].IpRanges[].CidrIp" --output text
echo "$PUBLIC_IP"    # GitHub Secrets의 EC2_HOST 값과 같아야 함
```

### Issue 7: Watchtower가 `Restarting (1)` 상태를 반복함
- **증상**: `docker run`은 종료 코드 0으로 끝났지만 로그에 `Error response from daemon: client version 1.25 is too old. Minimum supported API version is 1.40`이 반복됩니다.
- **원인**: `containrrr/watchtower` 1.7.1은 Docker API 1.25로 접속하는데, 최신 Docker Engine(29 이상)은 1.40 미만 요청을 거부합니다.
- **해결**: 컨테이너를 지우고 `-e DOCKER_API_VERSION=1.44`를 넣어 다시 실행합니다(Step 9·10의 명령 그대로).
```bash
sudo docker rm -f watchtower-poll
sudo docker version --format "server API {{.Server.APIVersion}} min {{.Server.MinAPIVersion}}"
# Step 9의 docker run 블록 재실행 후
sudo docker ps -a --filter name=watchtower
```

### Issue 8: 인스턴스를 재시작하자 중지해 둔 이전 컨테이너가 다시 실행됨
- **원인**: `deploy.sh`가 `docker compose stop`으로 멈춘 컨테이너는 종료 코드 `143`(SIGTERM)으로 남고, `restart: on-failure` 정책이 이를 실패 종료로 보아 데몬 기동 시 다시 실행합니다. 라우팅은 `service-url.inc` 기준이라 서비스에는 영향이 없지만 JVM 두 개가 메모리를 함께 점유합니다.
- **해결**: 재시작 후 비활성 컨테이너를 다시 중지합니다.
```bash
ssh -i ./"$MY_KEY_NAME".pem ubuntu@"$PUBLIC_IP" 'ACTIVE=$(grep -o "app-[a-z]*" ~/service-url.inc); for s in app-blue app-green; do [ "$s" = "$ACTIVE" ] || sudo docker compose -f ~/compose.blue-green.yaml stop "$s"; done'
```

### Issue 9: 이미지 발행 Job이 5~10분씩 걸림
- **원인**: `ubuntu-latest`(x86) 러너에서 `linux/arm64`를 빌드하면 Dockerfile의 Gradle 빌드 전체가 QEMU 에뮬레이션으로 실행됩니다. `arm64`만 빌드해도 5분 44초~9분 23초가 걸렸습니다.
- **해결**: 발행 Job을 ARM 네이티브 러너로 바꾸고 GitHub Actions 빌드 캐시를 켭니다(Step 6의 `ci.yml` 그대로). 같은 조건에서 발행 Job이 1분 40초로 줄었습니다.
```bash
grep -n "runs-on\|cache-" .github/workflows/ci.yml
# docker-build-push Job: runs-on: ubuntu-24.04-arm
# build-push-action: cache-from: type=gha / cache-to: type=gha,mode=max
```

---

## 5. 실습 자원 정리 (Cleanup)

### 5.1 당일 세션 종료 시
러너용 SSH 임시 허용 규칙을 먼저 회수하고 인스턴스를 중지합니다. EBS 볼륨의 Compose 설정·DB 데이터와 GitHub Secrets는 보존되므로 다음 세션에서 재시작해 이어갈 수 있습니다.

```bash
# 1. Step 2에서 연 러너 SSH 임시 허용 규칙 회수 (22번 포트를 학습자 IP 전용으로 복구)
aws ec2 revoke-security-group-ingress \
  --group-id "$MY_SG_ID" --security-group-rule-ids "$RUNNER_SSH_RULE_ID"

# 2. EC2 인스턴스 중지 명령
aws ec2 stop-instances --instance-ids "$INSTANCE_ID"

# 3. 인스턴스 완전 정지 대기
aws ec2 wait instance-stopped --instance-ids "$INSTANCE_ID"
echo "인스턴스가 안전하게 중지(Stopped)되었습니다. 컴퓨팅 요금이 발생하지 않습니다."

# 4. 현재 인스턴스 상태 확인
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].State.Name" --output text
```

- **확인 포인트**: 마지막 출력이 `stopped`이고, 보안 그룹의 22번 규칙에 `0.0.0.0/0`이 남아 있지 않아야 합니다. 중지 중에도 EBS 볼륨 요금은 발생합니다.
- **새 터미널에서 정리할 때**: `MY_SG_ID`·`RUNNER_SSH_RULE_ID`가 비어 있으면 아래처럼 전체 허용 규칙의 ID를 다시 찾습니다.
```bash
export MY_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=$MY_SG_NAME" \
  --query "SecurityGroups[0].GroupId" --output text)
export RUNNER_SSH_RULE_ID=$(aws ec2 describe-security-group-rules \
  --filters "Name=group-id,Values=$MY_SG_ID" \
  --query "SecurityGroupRules[?FromPort==\`22\` && CidrIpv4=='0.0.0.0/0'].SecurityGroupRuleId | [0]" --output text)
echo "$RUNNER_SSH_RULE_ID"
```
- **다음 세션 재시작 시**: Step 1로 재시작한 뒤 새 공인 IP로 `EC2_HOST` 시크릿을 갱신하고, CD를 다시 돌릴 때만 Step 2의 3번 블록으로 러너 규칙을 다시 추가합니다. 재시작 후에는 4절 Issue 8의 명령으로 비활성 컨테이너를 멈춥니다.

### 5.2 전체 종료 후 전면 삭제
더 이상 실습을 진행하지 않을 때 계정과 GitHub·Slack의 실습 자원을 모두 정리합니다. 아래 삭제는 되돌릴 수 없으므로 대상 이름을 확인한 뒤 실행합니다.

```bash
# 1. EC2 인스턴스 영구 종료 (루트 EBS 볼륨도 함께 삭제됨)
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"

# 2. 보안 그룹 삭제 (인스턴스 종료 후에만 삭제 가능)
export MY_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=$MY_SG_NAME" \
  --query "SecurityGroups[0].GroupId" --output text)
aws ec2 delete-security-group --group-id "$MY_SG_ID"

# 3. SSH 키 페어 정리
aws ec2 delete-key-pair --key-name "$MY_KEY_NAME"
rm -f ./"$MY_KEY_NAME".pem

# 4. 남은 자원이 없는지 확인 (결과가 비어 있어야 함)
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=$MY_INSTANCE_NAME" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[].Instances[].InstanceId" --output text
```

- **GitHub 저장소 정리**: `simple-back-ghcr` 저장소의 `Settings` → `Secrets and variables` → `Actions`에서 `EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY`, `SLACK_WEBHOOK_URL` 4건을 삭제합니다. 삭제된 서버의 접속 정보와 프라이빗 키를 저장소에 남겨 두지 않기 위함입니다. CD가 더 이상 실행되지 않게 하려면 `Actions` 탭에서 `CD Pipeline (Push via SSH with Slack Notification)` 워크플로우를 `Disable workflow`로 끄거나 `cd.yml`을 삭제해 푸시합니다.
- **GHCR 패키지 정리(선택)**: 이미지가 더 필요 없다면 GitHub 프로필의 `Packages`에서 `simple-back-ghcr` 패키지를 삭제합니다. 후속 과정에서 같은 이미지를 쓴다면 유지합니다.
- **Slack 앱 정리**: `https://api.slack.com/apps`에서 `Deploy-Bot`을 선택해 `Incoming Webhooks`의 Webhook을 `Remove`하거나, `Basic Information` 하단의 `Delete App`으로 앱을 삭제합니다. 앱을 지우면 기존 Webhook URL은 즉시 무효가 됩니다.
- **로컬 정리**: `~/workspace/simple-back`의 `v2.0` 변경을 유지할지 확인하고, 필요 없으면 원격과 함께 되돌립니다.
