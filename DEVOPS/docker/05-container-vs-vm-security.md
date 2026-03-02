# 컨테이너 vs VM / Docker 보안

---

## 1. 컨테이너 vs VM

### 가상 머신 (VM) 구조

```
┌────────────────────────────────────────┐
│  App A    │  App B    │  App C         │
│───────────│───────────│───────────     │
│  Guest OS │  Guest OS │  Guest OS      │  각각 독립된 OS 커널
│  (Ubuntu) │  (CentOS) │  (Alpine)      │
│────────────────────────────────────────│
│              Hypervisor                │  VMware ESXi, KVM, VirtualBox
│────────────────────────────────────────│
│              Host OS                   │
│────────────────────────────────────────│
│              Hardware                  │
└────────────────────────────────────────┘
```

### 컨테이너 구조

```
┌────────────────────────────────────────┐
│  App A    │  App B    │  App C         │
│───────────│───────────│───────────     │
│ Container │ Container │ Container      │  라이브러리 + 앱만 포함
│────────────────────────────────────────│
│           Docker Engine                │  컨테이너 런타임
│────────────────────────────────────────│
│           Host OS (Linux Kernel)       │  커널 공유
│────────────────────────────────────────│
│           Hardware                     │
└────────────────────────────────────────┘
```

### 핵심 차이

| | VM | Container |
|---|---|---|
| **격리 단위** | 하이퍼바이저 레벨 | Namespace / cgroup 레벨 |
| **OS** | 각자 독립 커널 | 호스트 커널 공유 |
| **이미지 크기** | 수 GB | 수 MB ~ 수백 MB |
| **시작 시간** | 수 분 | 수 초 이내 |
| **리소스 오버헤드** | 큼 (OS 자체 자원 소모) | 최소 |
| **격리 강도** | 강함 | 상대적으로 약함 |
| **이식성** | OS 이미지 단위 | 이미지 레이어 단위 |
| **사용 사례** | 강한 격리, 다른 OS 필요 | 마이크로서비스, CI/CD |

### 컨테이너 격리 메커니즘

```
Linux Namespace: 프로세스가 보는 시스템 리소스를 격리
  - PID namespace: 컨테이너 내부 프로세스 ID 격리
  - Network namespace: 독립된 네트워크 스택
  - Mount namespace: 독립된 파일시스템 뷰
  - UTS namespace: 독립된 호스트명

cgroup (Control Groups): 리소스 사용량 제한/모니터링
  - CPU 사용률 제한
  - 메모리 사용량 제한
  - 디스크 I/O 제한

docker run --memory="512m" --cpus="1.0" my-app
           ↑ cgroup으로 컨테이너 리소스 제한
```

---

## 2. Docker 보안

### root로 실행하지 않기

컨테이너 내부에서 root로 실행하면, 컨테이너 탈출 취약점 발생 시 호스트에 root 권한이 생긴다.

```dockerfile
# Bad: root로 실행 (기본값)
FROM openjdk:21-jre-slim
COPY app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
# 컨테이너 내부: whoami → root

# Good: 일반 유저로 실행
FROM openjdk:21-jre-slim

# 시스템 유저 생성 (로그인 불가, 홈 없음)
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 \
            --ingroup appgroup \
            --no-create-home \
            --shell /sbin/nologin \
            appuser

# 파일 소유권 설정
COPY --chown=appuser:appgroup app.jar app.jar

# 유저 전환 (이후 CMD/ENTRYPOINT도 이 유저로 실행)
USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 민감 정보를 이미지에 포함하지 않기

```dockerfile
# Bad: 비밀번호 하드코딩 → docker history로 노출됨
ENV DB_PASSWORD=super_secret
COPY application.properties .   # DB 비밀번호 포함된 파일
RUN echo "password=secret" >> /app/config.properties
```

```dockerfile
# Good: 런타임에 환경 변수로 주입
# Dockerfile에는 키 이름만 선언
ENV DB_PASSWORD=""

# 실행 시 주입
# docker run -e DB_PASSWORD=secret my-app
# docker compose (env_file 또는 environment 섹션)
```

```bash
# Docker Secret 사용 (Swarm 모드)
echo "my_secret_password" | docker secret create db_password -

# Compose에서 Secret 참조
# services:
#   app:
#     secrets:
#       - db_password
# secrets:
#   db_password:
#     external: true
```

### .dockerignore로 민감 파일 제외

```
# .dockerignore

.env
.env.*
*.pem
*.key
*.crt
application-local.yml
application-secret.yml
secrets/
.git/
```

### 읽기 전용 파일 시스템

```bash
# 컨테이너 파일 시스템을 읽기 전용으로 실행
# 악성코드가 파일을 수정하거나 쓰지 못함
docker run --read-only \
  --tmpfs /tmp \           # /tmp는 쓰기 가능 (필요한 경우)
  --tmpfs /app/logs \      # 로그 디렉토리는 쓰기 가능
  my-app
```

### Capability 제한

```bash
# 컨테이너가 가지는 Linux Capability를 최소화
docker run \
  --cap-drop ALL \           # 모든 capability 제거
  --cap-add NET_BIND_SERVICE \ # 80/443 포트 바인딩만 허용
  my-app
```

### 이미지 취약점 스캔

```bash
# Docker Scout (Docker Desktop 내장)
docker scout cves my-app:1.0
docker scout quickview my-app:1.0

# Trivy (오픈소스, CI/CD에 주로 사용)
trivy image my-app:1.0
trivy image --severity HIGH,CRITICAL my-app:1.0
trivy image --exit-code 1 --severity CRITICAL my-app:1.0
#           ↑ CRITICAL 발견 시 exit code 1 반환 → CI 파이프라인 실패

# GitHub Actions에서 자동 스캔
# - uses: aquasecurity/trivy-action@master
#   with:
#     image-ref: 'my-app:${{ github.sha }}'
#     severity: 'HIGH,CRITICAL'
#     exit-code: '1'
```

### 신뢰할 수 있는 베이스 이미지 사용

```dockerfile
# Bad: 검증되지 않은 이미지, latest 태그 (변경 가능)
FROM some-random-image:latest
FROM openjdk:latest

# Good: 공식 이미지 + 구체적인 버전 태그 (재현 가능)
FROM eclipse-temurin:21.0.3_9-jre-jammy
FROM openjdk:21.0.3-jre-slim

# 더 좋음: 다이제스트(SHA) 고정 (이미지 내용까지 보장)
FROM eclipse-temurin:21-jre@sha256:abc123...
```

### 불필요한 패키지 설치 금지

```dockerfile
# Bad: 필요 이상으로 설치
RUN apt-get install -y wget curl vim git ssh netcat

# Good: 필요한 것만, --no-install-recommends
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl && \        # 헬스체크에 필요한 curl만
    rm -rf /var/lib/apt/lists/*
```

---

## 3. 컨테이너 리소스 제한

컨테이너에 제한을 두지 않으면 하나의 컨테이너가 호스트 전체 리소스를 소모할 수 있다.

```bash
# 메모리 제한
docker run --memory="512m" my-app         # 최대 512MB
docker run --memory="512m" --memory-swap="512m" my-app  # swap도 비활성화

# CPU 제한
docker run --cpus="1.5" my-app            # 최대 1.5 코어
docker run --cpu-shares=512 my-app        # 상대적 가중치 (기본 1024)

# 확인
docker stats my-app
```

```yaml
# docker-compose.yml에서 리소스 제한
services:
  app:
    image: my-app
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: '1.0'
        reservations:       # 최소 보장량
          memory: 256m
          cpus: '0.5'
```

### JVM과 컨테이너 메모리 설정

```dockerfile
# JVM이 컨테이너 메모리 제한을 인식하게 설정
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \      # 컨테이너 메모리 한도 인식 (Java 10+)
    "-XX:MaxRAMPercentage=75.0", \     # 컨테이너 메모리의 75%를 힙으로 사용
    "-XX:InitialRAMPercentage=50.0", \ # 초기 힙 크기
    "-jar", "app.jar"]

# 컨테이너 메모리 512m 설정 시:
# JVM 힙: 512m * 75% = 약 384m
# 나머지: 메타스페이스, 스레드 스택, 코드 캐시 등
```
