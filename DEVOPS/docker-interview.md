# 백엔드 면접 - Docker 핵심 개념 정리

---

## 목차

1. [Docker란?](#1-docker란)
2. [이미지 vs 컨테이너](#2-이미지-vs-컨테이너)
3. [Dockerfile](#3-dockerfile)
4. [레이어 캐시 (Layer Cache)](#4-레이어-캐시-layer-cache)
5. [볼륨 (Volume)](#5-볼륨-volume)
6. [네트워크 (Network)](#6-네트워크-network)
7. [Docker Compose](#7-docker-compose)
8. [멀티 스테이지 빌드](#8-멀티-스테이지-빌드)
9. [컨테이너 vs VM](#9-컨테이너-vs-vm)
10. [Docker 보안](#10-docker-보안)

---

## 1. Docker란?

### 개념

Docker는 애플리케이션을 **컨테이너(Container)**라는 격리된 환경에서 실행할 수 있게 해주는 플랫폼이다.

"내 컴퓨터에서는 잘 되는데 서버에서는 안 돼요" 문제를 해결한다.
실행 환경(OS, 라이브러리, 설정)을 컨테이너 안에 함께 패키징하기 때문이다.

### 왜 Docker를 쓰나?

```
Before Docker:
  개발 환경: Java 11, MySQL 5.7, Redis 6
  운영 환경: Java 17, MySQL 8.0, Redis 7
  → 환경 차이로 예상치 못한 버그 발생

After Docker:
  개발 환경 = 운영 환경 = 테스트 환경
  → 컨테이너 이미지 하나로 어디서나 동일하게 실행

추가 장점:
  - 빠른 배포 (이미지 pull & run)
  - 격리 (컨테이너끼리 독립)
  - 자원 효율 (VM보다 가벼움)
  - 스케일링 용이 (컨테이너 늘리기 쉬움)
```

---

## 2. 이미지 vs 컨테이너

### 이미지 (Image)

컨테이너를 만들기 위한 **읽기 전용 템플릿**. 소스코드, 라이브러리, 설정, 실행 명령이 포함됨.
클래스와 비슷한 개념.

```bash
# 이미지 목록 확인
docker images

# 이미지 다운로드
docker pull openjdk:21-jdk-slim

# 이미지 삭제
docker rmi openjdk:21-jdk-slim
```

### 컨테이너 (Container)

이미지를 실행한 **인스턴스**. 이미지 위에 쓰기 가능한 레이어가 추가됨.
클래스로 생성한 객체(인스턴스)와 비슷한 개념.

```bash
# 컨테이너 실행
docker run -d \
  --name my-app \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  my-spring-app:1.0

# 실행 중인 컨테이너 목록
docker ps

# 모든 컨테이너 (중지된 것 포함)
docker ps -a

# 컨테이너 로그 확인
docker logs -f my-app   # -f: follow (실시간)

# 컨테이너 내부 접속
docker exec -it my-app /bin/bash

# 컨테이너 중지 / 삭제
docker stop my-app
docker rm my-app
```

### 이미지 레이어 구조

```
이미지는 여러 레이어가 쌓인 구조
각 레이어는 읽기 전용, 컨테이너 실행 시 맨 위에 쓰기 레이어 추가

Layer 4: COPY app.jar /app.jar  (내 앱)
Layer 3: RUN apt-get install curl  (라이브러리)
Layer 2: FROM openjdk:21-slim  (JDK)
Layer 1: Base OS (Ubuntu/Alpine)
────────────────────────────────
Container 쓰기 레이어 (컨테이너 종료 시 사라짐)
```

---

## 3. Dockerfile

### 기본 구조

```dockerfile
# 베이스 이미지 지정
FROM openjdk:21-jdk-slim

# 메타데이터 (선택적)
LABEL maintainer="dev@example.com"

# 환경 변수 설정
ENV APP_HOME=/app \
    TZ=Asia/Seoul

# 작업 디렉토리 설정
WORKDIR $APP_HOME

# 파일 복사 (호스트 → 컨테이너)
COPY build/libs/app.jar app.jar

# 포트 노출 선언 (실제 포트 열리는 건 -p 옵션)
EXPOSE 8080

# 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# 컨테이너 시작 시 실행할 명령
ENTRYPOINT ["java", "-jar", "app.jar"]

# ENTRYPOINT에 전달할 기본 인자 (docker run 시 덮어쓸 수 있음)
CMD ["--spring.profiles.active=prod"]
```

### Spring Boot 앱 Dockerfile

```dockerfile
FROM openjdk:21-jdk-slim

# 타임존 설정
RUN apt-get update && \
    apt-get install -y --no-install-recommends tzdata curl && \
    ln -snf /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    echo "Asia/Seoul" > /etc/timezone && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# JAR 파일 복사
COPY build/libs/*.jar app.jar

# 보안: root 대신 일반 유저로 실행
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup appuser
USER appuser

EXPOSE 8080

# JVM 옵션 (컨테이너 환경에 맞게)
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", "app.jar"]
```

### ENTRYPOINT vs CMD

```dockerfile
# ENTRYPOINT: 반드시 실행할 명령 (docker run으로 덮어쓰기 어려움)
# CMD: 기본 인자 또는 명령 (docker run으로 쉽게 덮어쓸 수 있음)

# 조합해서 사용
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=local"]

# docker run my-app                       → java -jar app.jar --spring.profiles.active=local
# docker run my-app --spring.profiles.active=prod → java -jar app.jar --spring.profiles.active=prod
```

### 이미지 빌드 및 푸시

```bash
# 이미지 빌드
docker build -t my-app:1.0 .
docker build -t my-app:1.0 -f Dockerfile.prod .  # 다른 Dockerfile 지정

# 태그 추가
docker tag my-app:1.0 registry.example.com/my-app:1.0

# 레지스트리에 푸시
docker push registry.example.com/my-app:1.0
```

---

## 4. 레이어 캐시 (Layer Cache)

### 개념

Dockerfile의 각 명령어는 별도 레이어를 생성한다.
**변경되지 않은 레이어는 캐시를 재사용**하므로 순서가 빌드 속도에 중요하다.

### 캐시 최적화 - 변경 빈도 낮은 것을 먼저

```dockerfile
# Bad: 소스코드가 바뀔 때마다 gradle 의존성도 다시 다운로드
FROM openjdk:21-jdk-slim
COPY . /app           # 소스 전체 복사 (코드 변경 시 캐시 무효화)
WORKDIR /app
RUN ./gradlew build   # 의존성 다시 다운로드 (느림)

# Good: 의존성 파일과 소스코드를 분리 복사
FROM openjdk:21-jdk-slim
WORKDIR /app

# Step 1: 의존성 관련 파일만 먼저 복사 (변경 드묾)
COPY build.gradle settings.gradle gradlew ./
COPY gradle ./gradle

# Step 2: 의존성 다운로드 (의존성 파일 변경 없으면 캐시 사용)
RUN ./gradlew dependencies --no-daemon

# Step 3: 소스코드 복사 (자주 변경됨)
COPY src ./src

# Step 4: 빌드 (소스 변경 시에만 여기부터 재실행)
RUN ./gradlew build --no-daemon -x test
```

### 캐시 무효화 규칙

```
레이어 N이 변경되면 레이어 N 이후의 모든 레이어 캐시가 무효화됨

COPY 명령: 복사할 파일의 내용이 변경되면 캐시 무효화
RUN 명령: 명령어 텍스트가 같으면 캐시 사용 (실제 결과와 무관)

→ 패키지 설치는 버전을 명시해서 캐시 일관성 유지
RUN apt-get install -y curl=7.68.0   # 버전 명시 권장
```

---

## 5. 볼륨 (Volume)

### 개념

컨테이너 내부 파일은 **컨테이너 종료 시 사라진다**. 데이터를 영구적으로 보존하려면 볼륨을 사용한다.

### 볼륨 종류

```
Named Volume (관리 볼륨):
  Docker가 관리하는 영구 저장소
  경로: /var/lib/docker/volumes/볼륨명/
  → DB 데이터, 애플리케이션 데이터 저장에 권장

Bind Mount (호스트 마운트):
  호스트의 특정 디렉토리를 컨테이너에 마운트
  → 개발 시 소스코드 실시간 반영에 사용

tmpfs Mount:
  메모리에만 저장, 컨테이너 종료 시 삭제
  → 민감한 임시 데이터
```

```bash
# Named Volume 생성 및 사용
docker volume create mydb-data

docker run -d \
  --name mysql \
  -v mydb-data:/var/lib/mysql \   # Named Volume 마운트
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# 볼륨 목록 / 삭제
docker volume ls
docker volume rm mydb-data

# Bind Mount (개발 환경에서 소스코드 마운트)
docker run -d \
  -v /Users/me/project:/app \    # 호스트경로:컨테이너경로
  -v /app/node_modules \         # node_modules는 마운트 제외
  -p 3000:3000 \
  node:20 npm run dev
```

### docker-compose.yml에서 볼륨 설정

```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    volumes:
      - mysql-data:/var/lib/mysql  # Named Volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Bind Mount (초기화 SQL)

volumes:
  mysql-data:  # Named Volume 선언
```

---

## 6. 네트워크 (Network)

### 개념

컨테이너끼리 통신하려면 같은 **Docker 네트워크**에 있어야 한다.
같은 네트워크의 컨테이너는 **컨테이너 이름**으로 서로를 찾을 수 있다.

### 네트워크 종류

```
bridge (기본):
  단일 호스트에서 컨테이너 간 통신
  사용자 정의 bridge → 컨테이너 이름으로 DNS 자동 해석

host:
  컨테이너가 호스트 네트워크 그대로 사용
  포트 매핑 불필요, 성능 좋음, 격리성 없음

none:
  네트워크 없음, 완전 격리
```

```bash
# 사용자 정의 네트워크 생성
docker network create my-network

# 네트워크에 컨테이너 연결
docker run -d --name mysql --network my-network mysql:8.0
docker run -d --name app   --network my-network my-spring-app

# app 컨테이너에서 mysql이라는 이름으로 MySQL 접근 가능
# JDBC URL: jdbc:mysql://mysql:3306/mydb  ← 컨테이너 이름이 호스트명
```

---

## 7. Docker Compose

### 개념

여러 컨테이너를 **한 번에 정의하고 관리**하는 도구. `docker-compose.yml` 파일로 구성.

### Spring Boot + MySQL + Redis 구성

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: spring-app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/mydb?useSSL=false&serverTimezone=Asia/Seoul
      SPRING_DATASOURCE_USERNAME: user
      SPRING_DATASOURCE_PASSWORD: password
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
      SPRING_PROFILES_ACTIVE: docker
    depends_on:
      mysql:
        condition: service_healthy   # MySQL 헬스체크 통과 후 시작
      redis:
        condition: service_started
    networks:
      - backend-network
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: mydb
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    ports:
      - "3307:3306"  # 호스트 3307 → 컨테이너 3306 (호스트의 3306 충돌 방지)
    volumes:
      - mysql-data:/var/lib/mysql
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "user", "-ppassword"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend-network

  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes  # 데이터 영속성 활성화
    networks:
      - backend-network

networks:
  backend-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

### Docker Compose 명령어

```bash
# 시작 (백그라운드)
docker compose up -d

# 특정 서비스만 시작
docker compose up -d mysql redis

# 로그 확인
docker compose logs -f app         # 특정 서비스
docker compose logs -f             # 전체

# 상태 확인
docker compose ps

# 재시작
docker compose restart app

# 중지 (컨테이너 삭제, 볼륨은 유지)
docker compose down

# 중지 + 볼륨까지 삭제 (DB 초기화)
docker compose down -v

# 이미지 재빌드 후 시작
docker compose up -d --build app
```

### 환경별 Compose 파일 분리

```yaml
# docker-compose.yml (공통)
version: '3.8'
services:
  app:
    image: my-app
    networks:
      - backend-network
  mysql:
    image: mysql:8.0
    networks:
      - backend-network
networks:
  backend-network:
```

```yaml
# docker-compose.dev.yml (개발 환경)
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - ./src:/app/src  # 소스코드 실시간 반영
    environment:
      SPRING_PROFILES_ACTIVE: dev
  mysql:
    ports:
      - "3307:3306"  # 외부 접근 허용
```

```yaml
# docker-compose.prod.yml (운영 환경)
version: '3.8'
services:
  app:
    image: registry.example.com/my-app:${APP_VERSION}
    deploy:
      replicas: 3
    environment:
      SPRING_PROFILES_ACTIVE: prod
```

```bash
# 환경별 실행
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 8. 멀티 스테이지 빌드

### 개념

**빌드 환경과 실행 환경을 분리**해 최종 이미지 크기를 줄이는 기법.
빌드 도구(Gradle, Maven, Node 등)는 실행에 불필요하므로 최종 이미지에서 제외.

### Spring Boot 멀티 스테이지 빌드

```dockerfile
# Stage 1: 빌드 스테이지 (JDK + Gradle 포함, 빌드에만 사용)
FROM openjdk:21-jdk-slim AS builder
WORKDIR /workspace

# 의존성 캐시 최적화
COPY gradlew .
COPY gradle gradle
COPY build.gradle settings.gradle .
RUN ./gradlew dependencies --no-daemon

# 소스 빌드
COPY src src
RUN ./gradlew build --no-daemon -x test

# JAR를 레이어로 분리 (Spring Boot layertools)
RUN java -Djarmode=layertools -jar build/libs/*.jar extract

# Stage 2: 실행 스테이지 (JRE만, 빌드 도구 없음)
FROM openjdk:21-jre-slim AS runtime
WORKDIR /app

# 빌드 스테이지에서 레이어만 복사
COPY --from=builder /workspace/dependencies/ ./
COPY --from=builder /workspace/spring-boot-loader/ ./
COPY --from=builder /workspace/snapshot-dependencies/ ./
COPY --from=builder /workspace/application/ ./

# 보안: 일반 유저로 실행
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]

# 결과:
# 단순 빌드: JDK + Gradle + 빌드 산출물 = ~800MB
# 멀티 스테이지: JRE + 실행 파일만 = ~200MB
```

### Spring Boot 레이어 구조

```
Spring Boot JAR를 레이어로 나누면:
  dependencies/       → 의존성 라이브러리 (거의 변경 안 됨)
  spring-boot-loader/ → Spring Boot 로더 (변경 안 됨)
  snapshot-dependencies/ → 스냅샷 의존성
  application/        → 내 소스코드 (자주 변경됨)

배포 시 변경된 레이어만 전송 → 배포 속도 향상
```

---

## 9. 컨테이너 vs VM

### 가상 머신 (VM)

```
┌─────────────────────────────────┐
│  App A  │  App B  │  App C      │ 애플리케이션
│─────────│─────────│─────────    │
│  Guest OS│  Guest OS│  Guest OS │ 각각 독립된 OS 커널
│  (Ubuntu)│  (CentOS)│  (Alpine) │
│──────────────────────────────── │
│         Hypervisor              │ 하이퍼바이저 (VMware, VirtualBox)
│──────────────────────────────── │
│         Host OS                 │
│──────────────────────────────── │
│         Hardware                │
└─────────────────────────────────┘

특징:
  - 각 VM이 독립된 OS 커널 포함 → 무거움 (수 GB)
  - 강한 격리 (Hypervisor 레벨)
  - 부팅 시간 수 분
  - 리소스 오버헤드 큼
```

### 컨테이너

```
┌─────────────────────────────────┐
│  App A  │  App B  │  App C      │ 애플리케이션
│─────────│─────────│─────────    │
│Container│Container│Container    │ 컨테이너 (라이브러리 + 앱만 포함)
│──────────────────────────────── │
│         Docker Engine           │ 컨테이너 런타임
│──────────────────────────────── │
│         Host OS (Linux Kernel)  │ OS 커널 공유
│──────────────────────────────── │
│         Hardware                │
└─────────────────────────────────┘

특징:
  - 호스트 OS 커널 공유 → 가벼움 (수 MB ~ 수 백 MB)
  - 프로세스 레벨 격리 (Namespace, cgroup)
  - 부팅 시간 수 초 이내
  - 리소스 오버헤드 최소
```

### 비교

| | VM | Container |
|---|---|---|
| 격리 수준 | 강함 (Hypervisor) | 보통 (Namespace) |
| 이미지 크기 | 수 GB | 수 MB ~ 수 백 MB |
| 시작 시간 | 수 분 | 수 초 이내 |
| 리소스 효율 | 낮음 | 높음 |
| 보안 | 높음 | 상대적으로 낮음 |
| OS | 다양한 Guest OS | 호스트 커널 공유 |
| 사용 사례 | 강한 격리 필요, 다른 OS | 마이크로서비스, CI/CD |

---

## 10. Docker 보안

### 루트 사용자로 실행하지 않기

```dockerfile
# Bad: root로 실행
FROM openjdk:21-jre-slim
COPY app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
# 컨테이너 탈출 시 호스트에 root 권한

# Good: 일반 유저로 실행
FROM openjdk:21-jre-slim

RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup --no-create-home appuser

COPY --chown=appuser:appgroup app.jar app.jar
USER appuser  # 이후 모든 명령은 appuser로 실행

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 민감 정보를 이미지에 포함하지 않기

```dockerfile
# Bad: 이미지에 비밀번호 하드코딩
ENV DB_PASSWORD=secret123
COPY application.properties .  # DB 비밀번호 포함된 설정 파일

# Good: 환경 변수로 런타임에 주입
# Dockerfile에는 환경 변수 이름만 선언
ENV DB_PASSWORD=""  # 빈 값으로 선언

# 실행 시 외부에서 주입
docker run -e DB_PASSWORD=secret123 my-app
# 또는 Docker Secret, AWS Secrets Manager, Vault 사용
```

```yaml
# docker-compose.yml - .env 파일 사용
services:
  app:
    env_file:
      - .env  # .env 파일에서 환경 변수 로드 (.gitignore에 추가!)
    environment:
      DB_PASSWORD: ${DB_PASSWORD}  # .env에서 가져옴
```

### .dockerignore

```
# .dockerignore - 이미지에 불필요한 파일 제외 (빌드 속도 + 보안)
.git
.gitignore
*.md
target/
build/
.gradle/
.env            # 비밀 정보
*.log
node_modules/
__pycache__/
.DS_Store
Dockerfile*
docker-compose*
```

### 이미지 취약점 스캔

```bash
# Docker Scout (Docker Desktop에 내장)
docker scout cves my-app:1.0

# Trivy (오픈소스)
trivy image my-app:1.0
trivy image --severity HIGH,CRITICAL my-app:1.0

# CI 파이프라인에서 자동 스캔
# GitHub Actions 예시:
# - uses: aquasecurity/trivy-action@master
#   with:
#     image-ref: my-app:${{ github.sha }}
#     exit-code: '1'  # 취약점 발견 시 빌드 실패
#     severity: 'HIGH,CRITICAL'
```

---

## 핵심 면접 질문 모음

**Q. Docker 이미지와 컨테이너의 차이는?**
> 이미지는 애플리케이션 실행 환경을 패키징한 읽기 전용 템플릿(클래스)이고, 컨테이너는 이미지를 실행한 인스턴스(객체)다. 하나의 이미지로 여러 컨테이너를 실행할 수 있다.

**Q. Docker 레이어 캐시를 최적화하는 방법은?**
> 자주 변경되지 않는 것(베이스 이미지, 의존성)을 Dockerfile 앞부분에, 자주 변경되는 것(소스코드)을 뒷부분에 배치한다. 특히 `COPY` 전에 의존성 파일만 먼저 복사하고 의존성을 설치하면 소스코드 변경 시 의존성 레이어 캐시를 재사용할 수 있다.

**Q. 멀티 스테이지 빌드란? 왜 쓰나?**
> 빌드 환경과 실행 환경을 분리하는 기법이다. 빌드 스테이지에서 Gradle, Maven 등 빌드 도구로 컴파일하고, 실행 스테이지에서는 빌드 결과물(JAR)만 복사해 최종 이미지를 만든다. 불필요한 빌드 도구가 제거되어 이미지 크기가 크게 줄어들고 (JDK → JRE, 800MB → 200MB), 보안도 향상된다.

**Q. 볼륨(Volume)과 바인드 마운트(Bind Mount)의 차이는?**
> 볼륨은 Docker가 관리하는 영구 저장소(`/var/lib/docker/volumes/`)로 운영 환경 데이터(DB 데이터 등) 저장에 적합하다. 바인드 마운트는 호스트의 특정 디렉토리를 마운트하는 것으로 개발 시 소스코드 실시간 반영에 활용한다.

**Q. 컨테이너와 VM의 차이는?**
> VM은 각자 독립된 OS 커널을 가져 강한 격리를 제공하지만 수 GB로 무겁고 부팅이 느리다. 컨테이너는 호스트 OS 커널을 공유하며 Namespace/cgroup으로 격리하기 때문에 수 MB~수백 MB로 가볍고 수 초 내에 시작된다. 컨테이너가 가볍고 빠르지만 격리 수준은 VM이 더 강하다.

**Q. depends_on과 healthcheck의 차이는?**
> `depends_on`만 쓰면 컨테이너가 '시작됐다'는 것만 기다린다. MySQL 컨테이너가 시작됐어도 DB가 준비되지 않았을 수 있다. `depends_on: condition: service_healthy`를 사용하면 healthcheck가 통과할 때까지 기다리므로 DB 연결 오류를 방지할 수 있다.
