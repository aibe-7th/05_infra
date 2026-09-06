# Docker Registry GHCR과 GitHub Actions 통합 실습 가이드

## 1. 실습 개요 및 목표

본 실습은 로컬 환경에서 완성된 도커 이미지를 클라우드 원격 저장소인 **GHCR (GitHub Container Registry)**에 안전하게 배포하고, **GitHub Actions CI 파이프라인**을 통해 Git 커밋/푸시 시점에 자동으로 컨테이너 이미지를 빌드·발행하는 클라우드 네이티브 자동화 배포 파이프라인을 구축하는 실습입니다.

> 💡 **[핵심 원리]**
> - **컨테이너 레지스트리 네임스페이스 규격**: 모든 OCI/도커 원격 이미지는 `[레지스트리 도메인]/[소유자 네임스페이스]/[이미지명]:[태그]` 형태의 절대 경로 식별자를 갖습니다. GHCR의 경우 `ghcr.io/<github-username>/<repo-name>:<tag>` 형식을 따릅니다.
> - **무인 자동화와 secrets.GITHUB_TOKEN**: 로컬 터미널에서 개인용 액세스 토큰(PAT)을 수동 발급받아 환경변수에 영구 보관하는 방식은 유출 위험이 큽니다. 반면 GitHub Actions 워크플로는 실행 중에만 일회성으로 생성되는 `secrets.GITHUB_TOKEN`과 `permissions: packages: write` 권한 선언을 조합하여 외부 자격 증명 없이 안전하게 GHCR 푸시를 무인 자동화합니다.
> - **CPU 아키텍처와 크로스 컴파일**: 로컬 개발 머신(예: Apple Silicon Mac의 `arm64`)과 클라우드 배포 서버(AWS, Render 등의 `x86_64`)의 CPU 아키텍처가 다르면 `exec format error`가 발생합니다. Buildx 크로스 플랫폼 빌드(`--platform linux/amd64`)나 표준 x86_64 클라우드 러너(`ubuntu-latest`) 빌드를 통해 완벽한 바이너리 호환성을 확보합니다.
> - **태그의 가변성과 불변 다이제스트**: `:latest`는 항상 최신 빌드를 가리키는 가변 포인터(Mutable Pointer)입니다. 실제 운영 배포에서는 고유 시맨틱 버전(`1.0`)이나 불변 다이제스트(`sha256:...`)를 통해 배포의 추적성과 안전성을 확보합니다.

### 핵심 학습 목표
1. 호스트 머신의 CPU 아키텍처를 확인하고, 클라우드 환경과의 호환성을 위해 `--platform linux/amd64` 기반 크로스 플랫폼 빌드를 수행한다.
2. GHCR 원격 레지스트리 규격(`ghcr.io/<사용자>/<이미지명>`)에 맞추어 버전(`1.0`), 환경(`prod`), 최신 포인터(`latest`) 다중 태그를 로컬에서 매핑한다.
3. GitHub Actions 워크플로(`.github/workflows/docker-publish.yml`)를 작성하고 Git 커밋/푸시를 통해 클라우드 러너가 `secrets.GITHUB_TOKEN`으로 이미지를 자동 빌드·발행하는 파이프라인을 검증한다.
4. GHCR 패키지를 Public으로 전환한 뒤 인증 없이 `docker pull`로 이미지를 내려받아 발행 결과를 검증하고 로컬 태그와의 차이를 분석한다.

---

## 2. 실습 환경 및 준비

본 실습은 앞 차시에서 작성한 Spring Boot 멀티 스테이지 `Dockerfile`이 포함된 `simple-back` 저장소를 그대로 사용합니다.

- **작업 디렉터리**: `~/workspace/simple-back`
- **형상 관리 시스템**: Git 및 GitHub 원격 리포지토리
- **컨테이너 레지스트리**: GitHub Container Registry (`ghcr.io`)
- **인증 방식**: GitHub Actions 러너 내장 `secrets.GITHUB_TOKEN` (로컬 터미널에서 별도 PAT 토큰을 발급받지 않음)
- **CI/CD 플랫폼**: GitHub Actions (`ubuntu-latest` 클라우드 러너)

> ⚠️ **[주의 사항]**
> - 실습을 진행하기 전 GitHub 웹사이트에서 본인 계정에 `simple-back` 이름의 빈(Public) 리포지토리를 미리 생성해 두어야 합니다.
> - 도커 레지스트리 규격상 이미지 저장소 경로는 **반드시 소문자(Lowercase)**만 허용됩니다. 본인 GitHub 아이디에 대문자가 포함되어 있다면 워크플로 내부에서 소문자로 자동 변환하는 절차가 필수적입니다.

---

## 3. 실습 절차

### Step 1. 클론된 저장소 상태 및 원격 연결 확인

프로젝트 루트 디렉터리로 이동하여 Git 상태와 원격 연결을 확인합니다.

```bash
cd ~/workspace/simple-back
git status
git remote -v
git log --oneline -5
```

- **옵션 및 구문 설명**:
  - `git status`: 현재 브랜치 상태 및 추적되지 않은 신규 파일(`Dockerfile` 등)을 확인합니다.
  - `git remote -v`: 현재 연결된 원격 저장소 URL을 확인합니다.

> 📌 **[기대 출력 확인]**
> `origin` 주소가 출력되고, 이전 차시에서 생성한 `Dockerfile`이 변경 사항으로 표시됩니다.

---

### Step 2. 크로스 플랫폼(Cross-platform) 빌드 점검

호스트 PC의 CPU 아키텍처를 확인하고, 클라우드 표준인 `x86_64`와 호환되도록 명시적 플랫폼 옵션을 적용하여 빌드합니다.

```bash
# 1. 호스트 머신 아키텍처 확인
uname -m

# 2. Docker Buildx 빌더 지원 환경 확인
docker buildx ls

# 3. linux/amd64 플랫폼을 명시한 크로스 빌드
docker build --platform linux/amd64 -t my-spring-app:amd64 .
docker inspect --format='{{.Architecture}}' my-spring-app:amd64
```

- **옵션 및 구문 설명**:
  - `uname -m`: 호스트 하드웨어 아키텍처(`arm64` 또는 `x86_64`)를 출력합니다.
  - `--platform linux/amd64`: 호스트 CPU와 관계없이 표준 64비트 Intel/AMD 아키텍처용 이미지를 빌드하도록 Buildx 에뮬레이터에 지시합니다.
  - `docker inspect --format='{{.Architecture}}'`: 생성된 이미지의 대상 아키텍처를 조회합니다.

> 📌 **[기대 출력 확인]**
> ```text
> amd64
> ```
> 호스트가 Apple Silicon(`arm64`)이더라도 빌드된 이미지의 아키텍처 메타데이터는 `amd64`로 정확히 지정되어야 합니다.

---

### Step 3. GHCR 규격 네임스페이스 다중 태그 매핑

원격 레지스트리 규격(`ghcr.io/<사용자>/<이미지명>:<태그>`)에 맞추어 기존 로컬 이미지에 3가지 별칭 태그를 매핑합니다. (원격 업로드는 GitHub Actions 러너가 전담하므로 로컬에서는 태그 매핑까지만 수행합니다.)

```bash
export GITHUB_USERNAME="본인_깃허브_아이디"

# 1. 고유 버전 태그(:1.0) 매핑
docker tag my-spring-app:1.0 ghcr.io/$GITHUB_USERNAME/simple-back:1.0

# 2. 운영 환경 태그(:prod) 및 최신 포인터(:latest) 추가 매핑
docker tag my-spring-app:1.0 ghcr.io/$GITHUB_USERNAME/simple-back:prod
docker tag my-spring-app:1.0 ghcr.io/$GITHUB_USERNAME/simple-back:latest

# 3. 태그 매핑 확인 (동일한 IMAGE ID 공유)
docker images ghcr.io/$GITHUB_USERNAME/simple-back
```

- **옵션 및 구문 설명**:
  - `docker tag [원본] [대상]`: 물리적 이미지를 복사하지 않고 기존 IMAGE ID에 새로운 네임스페이스와 태그 포인터를 추가합니다 (디스크 0바이트 추가 소모).

> 📌 **[기대 출력 확인]**
> ```text
> IMAGE                                           ID             CONTENT SIZE
> ghcr.io/<계정>/simple-back:1.0      da91e23a7d7c        124MB
> ghcr.io/<계정>/simple-back:latest   da91e23a7d7c        124MB
> ghcr.io/<계정>/simple-back:prod     da91e23a7d7c        124MB
> ```
> 세 태그가 모두 완전히 동일한 IMAGE ID(`da91e23a7d7c`)를 가리키고 있어야 합니다.

---

### Step 4. GitHub Actions 자동 빌드/푸시 워크플로 YAML 작성

코드 변경 시 GitHub 클라우드 러너에서 도커 이미지를 자동으로 빌드하고 GHCR로 푸시하는 CI 워크플로를 작성합니다.

```bash
mkdir -p .github/workflows

cat <<'WORKFLOW_EOF' > .github/workflows/docker-publish.yml
name: Build and Push Docker Image to GHCR

on:
  push:
    branches: [ "main" ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
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

      - name: Lowercase image name for Docker tag
        run: echo "IMAGE_NAME=${IMAGE_NAME,,}" >> $GITHUB_ENV

      - name: Build and push Docker image
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
WORKFLOW_EOF
```

- **옵션 및 구문 설명**:
  - `permissions: packages: write`: 워크플로 런타임에 GHCR 패키지를 생성하고 푸시할 수 있는 최소 권한을 부여합니다.
  - `${{ secrets.GITHUB_TOKEN }}`: 워크플로 실행 시점에만 임시 발급되는 보안 토큰으로, 수동 PAT 발급 없이 안전하게 레지스트리에 로그인합니다.
  - `${IMAGE_NAME,,}`: Bash 소문자 변환 문법으로, 계정명이나 저장소명에 포함된 대문자를 소문자로 강제 변환하여 도커 태그 명명 규칙을 준수합니다.

> 📌 **[기대 출력 확인]**
> `.github/workflows/docker-publish.yml` 파일이 정상 생성되어야 합니다.

---

### Step 5. 본인 GitHub 저장소 생성 및 커밋·푸시

작성한 워크플로와 Dockerfile을 본인의 GitHub 원격 저장소로 푸시하여 Actions 파이프라인을 트리거합니다.

```bash
git add .
git commit -m "feat: add Dockerfile and GitHub Actions workflow for GHCR"
git branch -M main

# 원격 대상을 본인 GitHub 저장소로 변경
git remote set-url origin https://github.com/$GITHUB_USERNAME/simple-back.git
git remote -v

# main 브랜치로 푸시
git push -u origin main
```

- **옵션 및 구문 설명**:
  - `git remote set-url origin`: 클론받았던 원본 주소 대신 본인이 생성한 개인 저장소 주소로 원격 타깃을 변경합니다.
  - `git push -u origin main`: 원격 저장소로 소스코드를 전송하고 트래킹 브랜치를 구성합니다.

> 📌 **[기대 출력 확인]**
> 푸시가 정상 완료되면 GitHub 웹 저장소 페이지에 커밋 내역이 반영됩니다.

---

### Step 6. Actions 파이프라인 실행 결과 및 GHCR 패키지 공개 전환

1. GitHub 웹 저장소의 **Actions** 탭으로 이동합니다.
2. `Build and Push Docker Image to GHCR` 워크플로가 실행 중인지 확인하고, 약 1~2분 후 **녹색 체크 표시(Success)**로 완료되는 것을 확인합니다.
3. 저장소 우측 또는 계정 프로필의 **Packages** 탭으로 이동하여 `simple-back` 패키지가 새로 등록되었는지 확인합니다.
4. **패키지 공개 전환 (Public)**:
   - `https://github.com/<GITHUB_USERNAME>?tab=packages` 접속
   - `simple-back` 패키지 클릭 후 우측 하단 **Package Settings** 진입
   - 페이지 최하단 **Danger Zone**의 **Change package visibility** 클릭
   - **Public**을 선택하고 확인 문구를 입력하여 공개로 전환합니다.

> 📌 **[기대 동작 확인]**
> 패키지가 Public으로 전환되면, 로컬 머신에서 별도의 `docker login` 인증을 거치지 않고도 전 세계 누구나 이미지를 자유롭게 다운로드(pull)할 수 있게 됩니다.

---

### Step 7. 공개 이미지 pull 및 로컬 실행 검증

공개 전환된 GHCR 이미지를 인증 없이 pull하여 러너가 발행한 이미지를 확인하고, 로컬 태그와의 차이점을 분석합니다.

```bash
# 1. 플랫폼 명시 pull (Apple Silicon 호스트 필수, x86_64 호스트는 생략 가능)
docker pull --platform linux/amd64 ghcr.io/$GITHUB_USERNAME/simple-back:latest

# 2. 이미지 매니페스트 확인 (인증 없이 조회 가능)
docker buildx imagetools inspect ghcr.io/$GITHUB_USERNAME/simple-back:latest
docker image inspect ghcr.io/$GITHUB_USERNAME/simple-back:latest --format '{{.Architecture}} {{.Os}}'

# 3. 로컬 태그와 원격 pull 이미지의 IMAGE ID 비교
docker images ghcr.io/$GITHUB_USERNAME/simple-back

# 4. 러너가 빌드한 amd64 이미지의 에뮬레이션 실행 확인
docker run --rm --platform linux/amd64 --entrypoint java ghcr.io/$GITHUB_USERNAME/simple-back:latest -version
```

- **옵션 및 구문 설명**:
  - `docker pull --platform linux/amd64`: GitHub Actions 기본 러너가 `linux/amd64` 이미지로 발행했으므로, Apple Silicon Mac에서 플랫폼 미지정으로 인한 매니페스트 불일치 오류를 방지합니다.
  - `docker images` 비교: 원격에서 pull받아 온 `:latest` 태그와 로컬에서 Step 3에 매핑해 둔 `:1.0`, `:prod`의 IMAGE ID를 대조합니다.

> 📌 **[기대 출력 확인]**
> ```text
> IMAGE                                           ID             CONTENT SIZE
> ghcr.io/<계정>/simple-back:1.0      da91e23a7d7c        124MB
> ghcr.io/<계정>/simple-back:latest   dc3a672edf97        124MB
> ghcr.io/<계정>/simple-back:prod     da91e23a7d7c        124MB
> 
> # Java 런타임 실행 확인
> openjdk version "17.0.20.1" 2026-08-18 LTS
> OpenJDK Runtime Environment Zulu17.68+203-CA
> ```
> - `:1.0`과 `:prod`는 로컬에서 빌드했던 ID(`da91e23a7d7c`)를 그대로 유지하는 반면, `:latest`는 GitHub Actions 러너가 클라우드에서 새로 빌드하여 푸시한 새 IMAGE ID(`dc3a672edf97`)로 교체되었음을 실증할 수 있습니다.
> - 이는 `:latest`가 고정된 불변 버전이 아니라 언제든 덮어씌워질 수 있는 '가변 포인터(Mutable Pointer)'임을 명확히 증명합니다.

---

## 4. 실무 트러블슈팅 가이드

실제 레지스트리 배포 및 CI 검증(`04_proven/03-2_Docker_Registry_GHCR과_GitHub_Actions.md`) 과정에서 확인된 핵심 오류와 해결책입니다.

### 1) Apple Silicon 호스트에서 GHCR 이미지 pull 시 매니페스트 불일치 오류
- **증상**: `docker pull ghcr.io/...:latest` 실행 시 `no matching manifest for linux/arm64/v8 in the manifest list entries: no match for platform in manifest: not found` 오류 발생.
- **원인**: 기본형 GitHub Actions 워크플로는 `platforms`를 별도 지정하지 않아 러너의 네이티브 환경인 `linux/amd64` 매니페스트만 레지스트리에 발행합니다. ARM64 호스트가 플랫폼 플래그 없이 pull을 시도하면 호환 매니페스트가 없어 다운로드가 거부됩니다.
- **해결책**: 터미널에서 `docker pull --platform linux/amd64 ghcr.io/...:latest`와 같이 아키텍처를 명시하여 pull을 실행합니다.

### 2) 로컬 CLI에서 GHCR push 시 권한 거부 및 키체인 오류
- **증상**: 터미널에서 `docker push ghcr.io/...` 실행 시 `permission_denied: The token provided does not match expected scopes.` 또는 macOS 키체인 오류(`-25308`) 발생.
- **원인**: 로컬 `gh` CLI 로그인 토큰에 `write:packages` 스코프가 누락되어 있거나 OS 키체인 잠금으로 자격 증명 저장이 차단되는 현상입니다.
- **해결책**: 정규 실습에서는 개인용 PAT를 로컬에 발급받아 환경변수에 저장하는 보안 위험을 배제하고, 워크플로 내부의 **`secrets.GITHUB_TOKEN`과 `permissions: packages: write` 선언**을 통해 클라우드 러너가 발행을 전담하도록 아키텍처를 일원화했습니다.

### 3) 리포지토리명 대문자 포함 시 태그 빌드 오류
- **증상**: GitHub Actions 로그에서 `invalid reference format: repository name must be lowercase` 에러 발생.
- **원인**: 도커 이미지 태그 규격은 오직 영문 소문자, 숫자, 마침표, 밑줄, 하이픈만 허용하며 대문자를 엄격히 금지합니다.
- **해결책**: 워크플로 스텝에 `run: echo "IMAGE_NAME=${IMAGE_NAME,,}" >> $GITHUB_ENV`를 선언하여 저장소명에 포함된 대문자를 소문자로 강제 치환합니다.

### 4) `gh api` 패키지 메타데이터 조회 시 403 에러
- **증상**: `gh api /user/packages/container/...` 실행 시 403 Forbidden 반환.
- **원인**: 로컬 `gh` 토큰에 `read:packages` 권한이 없기 때문입니다.
- **해결책**: 인증 상태와 무관하게 패키지가 Public 상태라면 `docker buildx imagetools inspect ghcr.io/...` 명령을 통해 누구나 매니페스트 메타데이터를 안정적으로 조회할 수 있습니다.

---


## 5. 실습 자원 정리 (Cleanup)

실습이 끝난 후 로컬 도커 보관소에 생성된 태그 이미지들을 정리하고 환경변수를 해제합니다.

```bash
# 1. 로컬 태그 매핑 이미지 및 pull로 교체된 이미지 일괄 정리
docker rmi ghcr.io/$GITHUB_USERNAME/simple-back:1.0
docker rmi ghcr.io/$GITHUB_USERNAME/simple-back:prod
docker rmi ghcr.io/$GITHUB_USERNAME/simple-back:latest

# 2. 크로스 빌드 테스트 이미지 정리
docker rmi my-spring-app:amd64

# 3. 임시 환경변수 해제
unset GITHUB_USERNAME

# 4. 잔여 이미지 목록 점검
docker images ghcr.io/$GITHUB_USERNAME/simple-back
```

> 💡 **[원격 GHCR 패키지 관리 안내]**
> GitHub 웹상에 등록된 `simple-back` 패키지는 실습 이후에도 보관하거나, 필요 시 GitHub 웹의 **Package Settings > Danger Zone > Delete this package**를 통해 언제든 삭제할 수 있습니다.
