# 이미지 / 컨테이너 / Dockerfile

---

## 1. Docker란?

애플리케이션을 **컨테이너(Container)**라는 격리된 환경에서 실행할 수 있게 해주는 플랫폼이다.

"내 컴퓨터에서는 되는데 서버에서 안 돼요" 문제를 해결한다.
실행 환경(OS 패키지, 라이브러리, 설정)을 컨테이너 안에 함께 패키징하기 때문이다.

```
Before Docker:
  개발 환경: Java 11, MySQL 5.7, Redis 6
  운영 환경: Java 17, MySQL 8.0, Redis 7
  → 환경 차이로 예상치 못한 버그 발생

After Docker:
  개발 = 운영 = 테스트 환경
  → 이미지 하나로 어디서나 동일하게 실행
```

---

## 2. 이미지 (Image)

컨테이너를 만들기 위한 **읽기 전용 템플릿**. 소스코드, 라이브러리, 설정, 실행 명령이 포함됨.
Java의 Class와 비슷한 개념이다.

```bash
# 이미지 다운로드 (Docker Hub에서)
docker pull openjdk:21-jdk-slim

# 로컬 이미지 목록
docker images

# 이미지 삭제
docker rmi openjdk:21-jdk-slim

# 이미지 상세 정보 (레이어 구조 등)
docker inspect openjdk:21-jdk-slim
```

### 이미지 레이어 구조

이미지는 여러 레이어가 쌓인 구조다. 각 레이어는 읽기 전용이고, 컨테이너 실행 시 맨 위에 쓰기 가능한 레이어가 추가된다.

```
Layer 4: COPY app.jar /app/app.jar     ← 내 앱 코드
Layer 3: RUN apt-get install curl      ← 라이브러리 설치
Layer 2: FROM openjdk:21-slim          ← JDK
Layer 1: Base OS (Debian/Alpine)       ← 베이스 OS
─────────────────────────────────────────
Container 쓰기 레이어                   ← 컨테이너 종료 시 사라짐
```

레이어를 공유하기 때문에 여러 이미지가 같은 베이스 이미지를 쓸 때 디스크를 공유한다.

---

## 3. 컨테이너 (Container)

이미지를 실행한 **인스턴스**. 이미지 위에 쓰기 가능한 레이어가 추가됨.
하나의 이미지로 여러 컨테이너를 실행할 수 있다.

```bash
# 컨테이너 실행
docker run -d \
  --name my-app \
  -p 8080:8080 \                          # 호스트:컨테이너 포트 매핑
  -e SPRING_PROFILES_ACTIVE=prod \        # 환경 변수
  -v /data/logs:/app/logs \               # 볼륨 마운트
  --memory="512m" \                       # 메모리 제한
  --cpus="1.0" \                          # CPU 제한
  my-spring-app:1.0

# 실행 중인 컨테이너
docker ps

# 모든 컨테이너 (중지된 것 포함)
docker ps -a

# 로그 확인
docker logs my-app         # 현재까지 로그
docker logs -f my-app      # 실시간 follow
docker logs --tail 100 my-app  # 마지막 100줄

# 컨테이너 내부 접속
docker exec -it my-app /bin/bash
docker exec -it my-app sh   # bash 없는 Alpine 이미지

# 컨테이너 자원 사용량 모니터링
docker stats my-app

# 컨테이너 중지 / 시작 / 재시작
docker stop my-app
docker start my-app
docker restart my-app

# 컨테이너 삭제 (중지 상태여야 함)
docker rm my-app
docker rm -f my-app  # 강제 삭제 (실행 중이어도)
```

### 컨테이너 생명주기

```
created → running → paused
                ↓
             stopped → removed

docker run  → created + running
docker stop → running → stopped  (SIGTERM 전송, 컨테이너는 남음)
docker kill → running → stopped  (SIGKILL, 즉시 종료)
docker rm   → stopped → removed  (컨테이너 삭제)
```

---

## 4. Dockerfile

### 기본 명령어

```dockerfile
# 베이스 이미지 (필수 첫 줄)
FROM openjdk:21-jdk-slim

# 이미지 메타데이터
LABEL maintainer="dev@example.com" version="1.0"

# 환경 변수 설정 (이미지 빌드 + 컨테이너 실행 시 모두 사용)
ENV APP_HOME=/app \
    TZ=Asia/Seoul

# RUN으로 설정된 ENV는 후속 레이어에서도 사용 가능
RUN echo $APP_HOME

# 작업 디렉토리 (없으면 자동 생성)
WORKDIR $APP_HOME

# 파일 복사 (호스트 → 이미지)
COPY build/libs/app.jar app.jar     # 단순 복사
ADD archive.tar.gz /app/            # 복사 + 압축 해제 (가능하면 COPY 권장)

# 포트 노출 선언 (실제 포트 열리는 건 -p 옵션, 여기선 문서화 역할)
EXPOSE 8080

# 헬스체크 (Docker가 컨테이너 상태 모니터링)
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# 컨테이너 시작 명령
ENTRYPOINT ["java", "-jar", "app.jar"]

# ENTRYPOINT에 전달할 기본 인자 (docker run 시 덮어쓸 수 있음)
CMD ["--spring.profiles.active=prod"]
```

### ENTRYPOINT vs CMD 차이

```dockerfile
# CMD만 사용: docker run으로 전체 명령 덮어쓰기 가능
CMD ["java", "-jar", "app.jar"]
# docker run my-app java -version  → java -version 실행 (CMD 무시됨)

# ENTRYPOINT만 사용: 고정 명령, docker run 인자는 뒤에 추가됨
ENTRYPOINT ["java", "-jar", "app.jar"]
# docker run my-app --server.port=9090  → java -jar app.jar --server.port=9090

# 조합 사용 (권장): ENTRYPOINT는 고정, CMD는 기본 인자
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=local"]
# docker run my-app                           → java -jar app.jar --spring.profiles.active=local
# docker run my-app --spring.profiles.active=prod → java -jar app.jar --spring.profiles.active=prod
```

### Spring Boot 실전 Dockerfile

```dockerfile
FROM openjdk:21-jdk-slim

# 필요한 패키지 설치 + 타임존 설정 (레이어 하나로 합쳐서 이미지 크기 최소화)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        tzdata \
        curl && \
    ln -snf /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    echo "Asia/Seoul" > /etc/timezone && \
    rm -rf /var/lib/apt/lists/*   # apt 캐시 제거 (이미지 크기 감소)

WORKDIR /app

# JAR 복사
COPY build/libs/*.jar app.jar

# 보안: root 대신 일반 유저로 실행
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup --no-create-home appuser && \
    chown appuser:appgroup app.jar
USER appuser

EXPOSE 8080

# 컨테이너 환경에 맞는 JVM 옵션
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \       # 컨테이너 CPU/메모리 제한 인식
    "-XX:MaxRAMPercentage=75.0", \      # 컨테이너 메모리의 75% 사용
    "-Djava.security.egd=file:/dev/./urandom", \  # 난수 생성 속도 개선
    "-jar", "app.jar"]
```

### 이미지 빌드 & 배포

```bash
# 빌드
docker build -t my-app:1.0 .
docker build -t my-app:1.0 -f Dockerfile.prod .  # Dockerfile 지정

# 태그 추가 (같은 이미지에 여러 태그)
docker tag my-app:1.0 my-app:latest
docker tag my-app:1.0 registry.example.com/my-app:1.0

# 레지스트리 로그인 & 푸시
docker login registry.example.com
docker push registry.example.com/my-app:1.0

# 레지스트리에서 pull
docker pull registry.example.com/my-app:1.0
```

---

## 5. .dockerignore

빌드 컨텍스트에서 불필요한 파일을 제외한다. 이미지 크기 감소 + 빌드 속도 향상 + 보안.

```
# .dockerignore

# 빌드 산출물 (Gradle/Maven)
build/
target/
.gradle/

# 소스코드 관련
.git/
.gitignore
*.md

# 로컬 설정 (민감 정보 포함 가능)
.env
.env.*
application-local.yml

# IDE 설정
.idea/
*.iml
.vscode/

# 테스트 관련
src/test/

# 로그
*.log

# OS 파일
.DS_Store
Thumbs.db
```
