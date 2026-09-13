# AWS EC2 배포 Docker 설치와 외부 Aiven DB 연동 실습 가이드

> 💡 **[핵심 원리]** 로컬에서 돌리던 컨테이너를 클라우드 서버로 옮깁니다. EC2는 애플리케이션 실행만 담당하고 데이터는 외부 관리형 DB(Aiven MySQL)가 보관하는 무상태 구조이며, 접속 정보는 이미지에 넣지 않고 실행 시점에 `--env-file`로 주입합니다(12-Factor App). Apple Silicon·Windows 개발 환경과 ARM Graviton(`t4g`) 서버의 CPU 아키텍처 차이는 GitHub Actions의 멀티 플랫폼 빌드(`linux/amd64,linux/arm64`)로 흡수합니다.

---

## 1. 실습 개요 및 목표

### 1.1 실습 개요
이전 차시에서 만들어 둔 `t4g.nano` 인스턴스를 스프링부트 컨테이너 구동에 적합한 `t4g.micro`(1GiB)로 사양 변경하고 재기동합니다. 재기동 시 공인 IP가 새로 할당되는 것을 확인한 뒤 SSH로 접속해 Docker 엔진을 설치하고, 데몬이 ARM64 아키텍처를 정상 인식하는지 검증합니다.

이어서 이미지 발행 워크플로에 `platforms: linux/amd64,linux/arm64`를 추가해 GHCR에 크로스 플랫폼 이미지를 발행하고, EC2에서 그 이미지를 내려받습니다. 마지막으로 Aiven MySQL 접속 정보를 `aiven.env` 파일로 분리해 컨테이너에 주입하고, 로컬 터미널에서 EC2 공인 IP로 직접 REST API를 호출해 보안 그룹 → 포트 매핑 → 애플리케이션 → 외부 DB로 이어지는 경로 전체를 검증합니다.

```mermaid
flowchart TB
    subgraph Local["로컬 개발 환경"]
        Repo["simple-back 저장소\n워크플로에 arm64 타깃 추가"]
        Term["로컬 터미널\nAWS CLI · SSH · curl"]
    end

    subgraph GitHub["GitHub"]
        Actions["GitHub Actions\nBuildx 멀티 플랫폼 빌드"]
        GHCR["GHCR 패키지\nlinux/amd64 + linux/arm64"]
        Actions --> GHCR
    end

    subgraph AWS["AWS 서울 리전"]
        EC2["EC2 t4g.micro (ARM)\nDocker 엔진 + simple-back 컨테이너"]
    end

    Aiven["Aiven MySQL (외부 DBaaS)\n실습 전용 데이터베이스"]

    Repo -->|git push| Actions
    GHCR -->|docker pull (arm64 자동 선택)| EC2
    Term -->|SSH 22| EC2
    Term -->|HTTP 8080| EC2
    EC2 -->|TLS 3306 계열 포트| Aiven
```

### 1.2 실습 목표
- **인스턴스 사양 변경**: 중지된 `t4g.nano`를 `t4g.micro`로 리사이징하고 재기동하여 메모리 확장과 공인 IP 재할당을 확인한다.
- **원격 서버 Docker 구성**: 공식 설치 스크립트로 Docker 엔진을 설치하고 데몬 상태와 `aarch64` 아키텍처를 검증한다.
- **크로스 플랫폼 이미지 발행**: 워크플로에 멀티 플랫폼 타깃을 추가해 GHCR에 `linux/amd64`·`linux/arm64` 매니페스트를 함께 발행한다.
- **외부 DB 연동 배포**: GHCR 이미지를 EC2에서 내려받고 Aiven 접속 정보를 `aiven.env`로 주입해 컨테이너를 구동한다.
- **외부 통신 검증과 비용 통제**: 로컬에서 공인 IP로 REST API 응답을 확인하고, 실습 종료 후 컨테이너와 인스턴스를 정지한다.

---

## 2. 실습 환경 및 준비

### 2.1 실습 환경 요약
- **AWS 자원**: 이전 차시에서 만든 EC2 인스턴스(`student01-test-ec2`), 보안 그룹(`student01-web-sg`), 키 페어(`~/.ssh/student01-key.pem`)
- **CLI 환경변수**: `AWS_PROFILE="student01"`, `AWS_REGION="ap-northeast-2"`, `AWS_PAGER=""`
- **터미널**: macOS/Linux는 `zsh`/`bash`, Windows는 **Git Bash**
- **외부 DB**: Aiven MySQL 접속 정보 5종 (호스트, 포트, 데이터베이스명, 계정, 암호)
- **로컬 프로젝트**: `~/workspace/simple-back` 클론과 이미지 발행 워크플로(`docker-publish.yml`)
- **이미지 경로**: `ghcr.io/<본인 GitHub 아이디>/simple-back:latest` — 워크플로가 `${{ github.repository }}`를 이미지 이름으로 쓰므로 **저장소 이름과 같습니다**

### 2.2 사전 준비 확인
1. **인스턴스가 `stopped` 상태인지** 확인합니다. 종료(`terminated`)했다면 이전 차시 절차로 다시 만들어야 합니다.
2. **접속 출발지 IP가 바뀌지 않았는지** 확인합니다. 네트워크가 바뀌었다면 보안 그룹의 22번 규칙을 현재 공인 IP로 다시 등록합니다.
3. **Aiven 데이터베이스는 실습 전용 빈 데이터베이스**를 사용합니다. 기본 `defaultdb`에 다른 프로젝트의 `users` 테이블이 남아 있으면 스키마 자동 갱신이 실패해 조회 API가 500을 반환합니다. 필요하면 Aiven 콘솔이나 클라이언트에서 `CREATE DATABASE simpleback;`으로 빈 데이터베이스를 만들어 두세요.

---

## 3. 핵심 실습 절차 (Step-by-Step)

### Step 1. 인스턴스 사양 변경과 재기동

```bash
export MY_KEY_NAME="student01-key"
export AWS_PROFILE="student01"
export AWS_REGION="ap-northeast-2"
export AWS_PAGER=""

# 1. 중지된 인스턴스 ID 조회
export INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=student01-test-ec2" "Name=instance-state-name,Values=stopped" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
echo "대상 인스턴스 ID: $INSTANCE_ID"

# 2. 사양 변경 (t4g.nano -> t4g.micro)
aws ec2 modify-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --instance-type "Value=t4g.micro"

# 3. 시작 및 상태 검사 통과 대기
aws ec2 start-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-status-ok --instance-ids "$INSTANCE_ID"

# 4. 새로 할당된 공인 IP 조회
export PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "새로 할당된 공인 IP: $PUBLIC_IP"
```

| 콘솔에서 볼 위치 | 확인 항목 |
|---|---|
| EC2 → `인스턴스` 목록 | `인스턴스 유형` 열이 `t4g.micro`로 변경 |
| 같은 목록의 `상태 검사` 열 | `2/2개 검사 통과` (실측 약 2분 30초) |
| 인스턴스 선택 후 `세부 정보` 탭 | `퍼블릭 IPv4 주소`가 이전 값과 다름 |

- 사양 변경은 **중지 상태에서만** 가능합니다. 실행 중이면 `IncorrectInstanceState` 오류가 반환됩니다.

### Step 2. SSH 접속과 시스템 자원 점검

```bash
ssh -i ~/.ssh/"$MY_KEY_NAME".pem -o StrictHostKeyChecking=accept-new ubuntu@"$PUBLIC_IP"
```

```bash
# ─── 원격 EC2 터미널 내부 ───
whoami          # ubuntu
uname -m        # aarch64
df -h /
free -h
```

- **검증 기준**: `free -h`의 `Mem:` 합계가 `903Mi` 내외로 표시되어야 합니다(`t4g.nano`는 `405Mi`). 리사이징이 반영되지 않았다면 인스턴스 유형을 다시 확인합니다.

### Step 3. 네트워크 기준선 확인

```bash
# ─── 원격 EC2 터미널 내부 ───
ip addr show | grep "inet "   # 사설 IP: 172.31.x.x
curl -s ifconfig.me; echo     # 공인 IP: Step 1에서 조회한 값과 동일
sudo ss -tulnp                # 컨테이너 기동 전: 22번과 로컬 DNS만 리스닝
exit
```

- 사설 IP(`172.31.x.x`)는 VPC 내부 주소이고 공인 IP는 외부 접속용 주소입니다. 인스턴스 안에서는 공인 IP가 인터페이스에 직접 보이지 않습니다.

### Step 4. Docker 엔진 설치

```bash
ssh -i ~/.ssh/"$MY_KEY_NAME".pem ubuntu@"$PUBLIC_IP"
```

```bash
# ─── 원격 EC2 터미널 내부 ───
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo systemctl status docker --no-pager
```

- **검증 기준**: `Active: active (running)` 표시. 설치는 1~2분 걸립니다.

### Step 5. 데몬 상태와 아키텍처 검증

```bash
# ─── 원격 EC2 터미널 내부 ───
sudo docker --version
sudo docker info | grep -E "Architecture|Operating System|Server Version|Storage Driver"
sudo docker ps -a
exit
```

- **검증 기준**: `Architecture: aarch64`. 이미지 메타데이터 규격(`arm64`)과 커널 표기(`aarch64`)는 같은 아키텍처의 다른 표기입니다.
- 최신 환경에서는 `Storage Driver`가 `overlay2` 대신 `overlayfs`로 표시될 수 있습니다.

### Step 6. 워크플로에 멀티 플랫폼 타깃 추가

로컬 프로젝트에서 이미지 발행 워크플로의 빌드 스텝 한 곳만 수정합니다.

```bash
cd ~/workspace/simple-back
```

```yaml
      - name: Build and push multi-platform Docker image
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

- `platforms`를 지정하지 않으면 러너 아키텍처인 `linux/amd64` 매니페스트만 발행되어, ARM EC2에서 `no matching manifest for linux/arm64/v8` 오류로 pull이 실패합니다.

### Step 7. 푸시와 발행 결과 확인

```bash
git add .github/workflows/docker-publish.yml
git commit -m "ci: add linux/arm64 target for EC2 t4g deployment"
git push origin main
```

| 화면 | 볼 위치 | 확인 항목 |
|---|---|---|
| 저장소 **Actions** 탭 | 최신 실행 행 | 초록 체크(성공). 멀티 플랫폼 빌드는 단일 아키텍처보다 오래 걸립니다 |
| 저장소 우측 **Packages** | 패키지 → 태그 상세 | `OS/Arch` 목록에 `linux/amd64`와 `linux/arm64`가 모두 표시 |

- 목록에 `unknown/unknown` 항목이 함께 보이는 것은 빌드 증명(attestation) 매니페스트로 정상입니다.

### Step 8. EC2에서 이미지 pull과 접속 정보 파일 작성

```bash
ssh -i ~/.ssh/"$MY_KEY_NAME".pem ubuntu@"$PUBLIC_IP"
```

```bash
# ─── 원격 EC2 터미널 내부 ───
export GITHUB_USERNAME="본인_깃허브_아이디"

# 1. GHCR 공개 이미지 pull (호스트 아키텍처에 맞는 arm64가 자동 선택)
sudo docker pull ghcr.io/"$GITHUB_USERNAME"/simple-back:latest
sudo docker image inspect ghcr.io/"$GITHUB_USERNAME"/simple-back:latest --format "{{.Os}}/{{.Architecture}}"
# linux/arm64

# 2. Aiven 접속 정보 파일 생성 (실습 전용 데이터베이스명 사용)
cat <<'EOF' > aiven.env
SPRING_DATASOURCE_URL=jdbc:mysql://<AIVEN_MYSQL_HOST>:<PORT>/<실습_전용_DB명>?sslMode=REQUIRED
SPRING_DATASOURCE_USERNAME=avnadmin
SPRING_DATASOURCE_PASSWORD=실제_Aiven_암호
APP_MESSAGE=Live on AWS EC2 t4g.micro via GHCR!
EOF
chmod 600 aiven.env
```

- 이미지 이름은 **저장소 이름**과 같습니다(`${{ github.repository }}` 기준). 로컬 빌드에서 쓰던 `my-spring-app`은 GHCR에 발행되지 않습니다.
- `sslMode`는 대소문자를 구분하는 MySQL Connector/J 속성입니다. 접속 정보 파일은 `chmod 600`으로 보호합니다.

### Step 9. 컨테이너 구동과 부팅 로그 확인

```bash
# ─── 원격 EC2 터미널 내부 ───
sudo docker run -d \
  --name simple-back \
  --restart unless-stopped \
  -p 8080:8080 \
  --env-file aiven.env \
  ghcr.io/"$GITHUB_USERNAME"/simple-back:latest

sudo docker ps
sudo docker logs --tail 30 simple-back
sudo ss -tulnp | grep 8080
exit
```

- **검증 기준**: 로그 마지막에 `Started SimpleBackApplication in N seconds`(실측 약 12초)가 출력되고, 8080 포트를 `docker-proxy`가 리스닝합니다.
- 로그에 `Incorrect integer value ... for column 'id'` 같은 스키마 예외가 보이면 대상 데이터베이스에 다른 스키마의 `users` 테이블이 있는 것입니다. 4절 Issue 3을 참고하세요.

### Step 10. 로컬에서 외부 REST API 검증

```bash
# 로컬 호스트 터미널에서 실행
curl -i http://"$PUBLIC_IP":8080/
curl -i http://"$PUBLIC_IP":8080/users
```

```
HTTP/1.1 200
{"status":"UP","message":"Live on AWS EC2 t4g.micro via GHCR!","timestamp":"..."}

HTTP/1.1 200
[{"id":1,"name":"infra-admin","email":"admin@example.com"}]
```

- `/`는 DB를 거치지 않고, `/users`는 Aiven MySQL을 조회합니다. 둘 다 200이면 보안 그룹 → 포트 매핑 → 애플리케이션 → 외부 DB 경로가 전 구간 정상입니다.

### Step 11. 컨테이너 정지와 인스턴스 중지

```bash
# 1. 원격 컨테이너 정지
ssh -i ~/.ssh/"$MY_KEY_NAME".pem ubuntu@"$PUBLIC_IP" "sudo docker stop simple-back"

# 2. 인스턴스 중지 (컴퓨팅 요금 동결)
aws ec2 stop-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-stopped --instance-ids "$INSTANCE_ID"
```

- 컨테이너 종료 코드 `143`은 `128 + 15`(SIGTERM)로 `docker stop`에 의한 정상 종료입니다.

---

## 4. 실무 트러블슈팅 가이드

### Issue 1: `no matching manifest for linux/arm64/v8`
이미지가 `linux/amd64`로만 발행된 경우입니다. 워크플로의 `platforms`를 확인하고, 급하면 `docker pull --platform linux/amd64`로 강제 수신할 수 있지만 ARM 인스턴스에서는 실행되지 않거나 성능이 크게 떨어집니다.

### Issue 2: `docker pull` 시 `denied` 또는 `unauthorized`
GHCR 패키지가 비공개인 경우입니다. 패키지 설정에서 공개로 바꾸거나, EC2에서 `read:packages` 권한의 토큰으로 `docker login ghcr.io`를 먼저 수행합니다.

### Issue 3: 부팅 로그에 스키마 예외, `/users`만 500
대상 데이터베이스에 다른 프로젝트의 `users` 테이블이 남아 있어 JPA 스키마 갱신이 기존 컬럼 타입 변환에 실패한 경우입니다(예: `Incorrect integer value: 'admin' for column 'id'`). 기존 테이블을 지우지 말고 `CREATE DATABASE simpleback;`으로 실습 전용 데이터베이스를 만든 뒤 `aiven.env`의 URL에서 데이터베이스명만 교체하고 컨테이너를 다시 만듭니다.

```bash
sudo docker rm -f simple-back
# aiven.env 수정 후 Step 9의 docker run 재실행
```

### Issue 4: SSH 접속 타임아웃
인스턴스를 재기동하면 공인 IP가 바뀌고, 접속 출발지 IP가 바뀌면 보안 그룹 22번 규칙이 더 이상 본인을 가리키지 않습니다. `describe-instances`로 IP를 다시 조회하고, `curl -fsS https://checkip.amazonaws.com`으로 현재 IP를 확인해 규칙을 갱신합니다.

### Issue 5: 사양 변경이 거부됨
`modify-instance-attribute`는 인스턴스가 `stopped` 상태일 때만 동작합니다. 또한 교육 계정 가드레일 때문에 `t4g.nano`·`t4g.micro`·`t4g.small` 외의 유형으로는 변경·생성할 수 없습니다.

---

## 5. 실습 자원 정리 (Cleanup)

### 5.1 당일 세션 종료 시 (다음 차시 재사용)

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].{State:State.Name,Type:InstanceType,Ip:PublicIpAddress}" --output json
# {"State": "stopped", "Type": "t4g.micro", "Ip": null}
```

- 중지하면 컴퓨팅 요금은 0이 되고 EBS 보관료만 남습니다. 공인 IP는 회수되므로 다음 기동 시 다시 조회해야 합니다.

### 5.2 전체 실습 종료 후 (자원 전면 삭제)

```bash
# 1. 인스턴스 영구 종료
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"

# 2. 보안 그룹 삭제
aws ec2 delete-security-group --group-id "$MY_SG_ID"

# 3. 키 페어 삭제 및 로컬 파일 제거
aws ec2 delete-key-pair --key-name "$MY_KEY_NAME"
rm -f ~/.ssh/"$MY_KEY_NAME".pem

# 4. 잔여 자원 확인 (모두 빈 결과여야 정상)
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running,stopped" \
  --query "Reservations[].Instances[].InstanceId" --output text
aws ec2 describe-volumes --query "Volumes[].VolumeId" --output text
```

- 인스턴스를 종료하면 루트 EBS 볼륨도 함께 삭제되어 스토리지 요금까지 멈춥니다.
- Aiven에 실습 전용 데이터베이스를 만들었다면 더 이상 쓰지 않을 때 `DROP DATABASE`로 정리합니다. 기본 데이터베이스와 기존 테이블은 건드리지 않습니다.
