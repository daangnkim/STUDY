# 레이어 캐시 / 멀티 스테이지 빌드

---

## 1. 레이어 캐시 (Layer Cache)

### 개념

Dockerfile의 각 명령어(RUN, COPY, ADD 등)는 별도 레이어를 생성한다.
**이전 빌드와 동일한 레이어는 캐시를 재사용**하므로 순서 설계가 빌드 속도에 직결된다.

### 캐시 무효화 규칙

```
레이어 N이 변경되면 → N 이후의 모든 레이어 캐시가 무효화됨

COPY 명령: 복사할 파일 내용이 바뀌면 캐시 무효화
RUN 명령: 명령어 텍스트가 같으면 캐시 사용 (실제 결과와 무관)
FROM 명령: 베이스 이미지 다이제스트가 같으면 캐시 사용
```

### 잘못된 순서 vs 올바른 순서

```dockerfile
# Bad: 소스코드가 조금만 바뀌어도 의존성을 처음부터 다시 다운로드
FROM openjdk:21-jdk-slim
WORKDIR /app
COPY . .                    # 소스 전체 복사 → 코드 변경 시 아래 모든 레이어 재실행
RUN ./gradlew build         # 의존성 다운로드 + 빌드 (매번 수 분 소요)
```

```dockerfile
# Good: 의존성 파일과 소스코드를 분리해서 캐시 최대 활용
FROM openjdk:21-jdk-slim
WORKDIR /app

# Step 1: 의존성 관련 파일만 먼저 복사 (거의 변경 안 됨)
COPY gradlew .
COPY gradle gradle
COPY build.gradle settings.gradle ./

# Step 2: 의존성 다운로드
# → build.gradle이 바뀌지 않으면 이 레이어 캐시 재사용 (수 분 절약)
RUN ./gradlew dependencies --no-daemon

# Step 3: 소스코드 복사 (자주 변경됨)
COPY src ./src

# Step 4: 빌드 (소스 변경 시에만 여기부터 재실행)
RUN ./gradlew build --no-daemon -x test

# 결과:
# 소스코드만 바꿨을 경우:
#   Step 1,2는 캐시 사용 (수 분 절약)
#   Step 3,4만 재실행 (수십 초)
```

### Node.js 예시

```dockerfile
FROM node:20-alpine
WORKDIR /app

# package.json만 먼저 복사 → npm install 캐시 활용
COPY package.json package-lock.json ./
RUN npm ci --only=production   # npm install보다 캐시 친화적

# 소스코드 복사 (의존성과 분리)
COPY src ./src
COPY tsconfig.json ./

RUN npm run build
```

### apt 패키지 설치 캐시 주의사항

```dockerfile
# Bad: apt-get update 따로 RUN → 캐시 불일치로 오래된 패키지 받을 수 있음
RUN apt-get update
RUN apt-get install -y curl

# Good: 같은 RUN에서 update + install + 캐시 정리
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        tzdata && \
    rm -rf /var/lib/apt/lists/*
#         ↑ apt 캐시 삭제 (이미지 크기 줄임)
```

### 캐시 강제 무효화

```bash
# 캐시 무시하고 처음부터 빌드
docker build --no-cache -t my-app:1.0 .

# 특정 단계부터 재빌드 (ARG 활용)
ARG CACHE_BUST=1
RUN echo $CACHE_BUST && apt-get update  # 빌드 시 --build-arg CACHE_BUST=$(date) 로 캐시 무효화
```

---

## 2. 멀티 스테이지 빌드 (Multi-Stage Build)

### 개념

**빌드 환경과 실행 환경을 분리**해 최종 이미지 크기를 최소화하는 기법.
빌드 도구(Gradle, Maven, Node 등)는 실행에 필요 없으므로 최종 이미지에서 제거한다.

```
단일 스테이지:
  JDK + Gradle + 빌드 산출물 + 소스코드 = ~800MB

멀티 스테이지:
  빌드 스테이지: JDK + Gradle + 소스코드 (사용 후 버림)
  실행 스테이지: JRE + JAR만 = ~200MB
```

### Spring Boot 멀티 스테이지 빌드

```dockerfile
# ============================
# Stage 1: 빌드 스테이지
# ============================
FROM openjdk:21-jdk-slim AS builder
WORKDIR /workspace

# 의존성 캐시 최적화
COPY gradlew .
COPY gradle gradle
COPY build.gradle settings.gradle ./
RUN ./gradlew dependencies --no-daemon

# 빌드
COPY src src
RUN ./gradlew build --no-daemon -x test

# Spring Boot Layertools로 JAR를 레이어별로 분리
# (다음 스테이지에서 레이어별로 COPY → 캐시 효율 향상)
RUN java -Djarmode=layertools -jar build/libs/*.jar extract --destination layers


# ============================
# Stage 2: 실행 스테이지
# ============================
FROM openjdk:21-jre-slim AS runtime
# JRE만 있으면 됨 (JDK, Gradle 없음)

WORKDIR /app

# 보안: 일반 유저 생성
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup --no-create-home appuser

# 빌드 스테이지에서 레이어만 가져오기 (--from=builder)
# 변경 빈도 낮은 것 먼저 복사 → 이 스테이지의 캐시도 최적화
COPY --from=builder --chown=appuser:appgroup /workspace/layers/dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /workspace/layers/spring-boot-loader/ ./
COPY --from=builder --chown=appuser:appgroup /workspace/layers/snapshot-dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /workspace/layers/application/ ./
# application/ 레이어만 소스코드 변경에 반응 → 나머지 레이어는 캐시 재사용

USER appuser
EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "org.springframework.boot.loader.launch.JarLauncher"]
```

### Spring Boot 레이어 구조

```
Spring Boot JAR 레이어 (변경 빈도 오름차순):
  dependencies/         → Maven Central 의존성 (거의 안 바뀜)
  spring-boot-loader/   → Spring Boot 로더 (Spring Boot 버전 올릴 때만)
  snapshot-dependencies/ → 스냅샷 의존성 (가끔)
  application/          → 내 소스코드 (자주 바뀜)

배포 시 변경된 레이어만 push/pull → 배포 시간 대폭 단축
예: 코드만 변경 → application/ 레이어만 전송 (수 MB)
    의존성 추가 → dependencies/ + application/ 레이어 전송
```

### Node.js 멀티 스테이지 빌드

```dockerfile
# Stage 1: 빌드
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build   # TypeScript 컴파일, 번들링 등

# Stage 2: 실행
FROM node:20-alpine AS runtime
WORKDIR /app

# 프로덕션 의존성만 설치
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# 빌드 결과물만 복사 (소스코드 없음)
COPY --from=builder /app/dist ./dist

USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### 특정 스테이지만 빌드 (디버깅용)

```bash
# 특정 스테이지까지만 빌드 (--target)
docker build --target builder -t my-app:builder .
docker run --rm -it my-app:builder /bin/bash   # 빌드 환경 접속해서 디버깅

# 최종 이미지 빌드
docker build -t my-app:1.0 .
```

### 이미지 크기 비교

```bash
# 빌드 전후 이미지 크기 확인
docker images my-app

# 레이어별 크기 확인
docker history my-app:1.0

# 상세 레이어 분석
docker inspect my-app:1.0 | jq '.[0].RootFS.Layers'
```
