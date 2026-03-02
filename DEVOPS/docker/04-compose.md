# Docker Compose

---

## 개념

여러 컨테이너를 **단일 YAML 파일로 정의하고 한 번에 관리**하는 도구.
개발 환경, 테스트 환경 구성에 필수적으로 사용된다.

```bash
# 단순 비교

# Compose 없이 MySQL + Redis + App 실행 (번거로움)
docker network create my-net
docker volume create mysql-data
docker run -d --name mysql --network my-net -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret mysql:8.0
docker run -d --name redis --network my-net redis:7
docker run -d --name app --network my-net -p 8080:8080 my-app

# Compose 사용 (한 줄)
docker compose up -d
```

---

## 기본 구조

```yaml
version: '3.8'

services:       # 실행할 컨테이너들
  app: ...
  mysql: ...
  redis: ...

networks:       # 사용할 네트워크 (없으면 자동 생성)
  backend: ...

volumes:        # 사용할 볼륨 (없으면 자동 생성)
  mysql-data: ...
```

---

## Spring Boot + MySQL + Redis 풀 구성

```yaml
version: '3.8'

services:

  # ── 애플리케이션 ──────────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: spring-app
    ports:
      - "8080:8080"
    environment:
      # Spring 설정 (환경 변수로 application.yml 오버라이드)
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/mydb?useSSL=false&serverTimezone=Asia/Seoul&characterEncoding=UTF-8
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}       # .env 파일에서 로드
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PORT: 6379
    depends_on:
      mysql:
        condition: service_healthy   # MySQL 헬스체크 통과 후에 앱 시작
      redis:
        condition: service_started
    networks:
      - backend-network
    restart: unless-stopped          # 비정상 종료 시 자동 재시작
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s              # 앱 기동 시간 여유

  # ── MySQL ────────────────────────────────────────────────────
  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: mydb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "3307:3306"                  # 호스트 3306 충돌 방지
    volumes:
      - mysql-data:/var/lib/mysql
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql  # 초기화 SQL
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost",
             "-u", "appuser", "-p${DB_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - backend-network

  # ── Redis ────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    networks:
      - backend-network

networks:
  backend-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

```bash
# .env 파일 (git에 올리지 말 것!)
DB_PASSWORD=mySecurePassword123
MYSQL_ROOT_PASSWORD=rootSecurePassword
REDIS_PASSWORD=redisPassword
```

---

## depends_on vs healthcheck

```yaml
# depends_on만 사용: 컨테이너가 '시작됨' 상태만 기다림
# MySQL 컨테이너가 시작됐어도 DB 준비 안 됐을 수 있음 → 앱 Connection 오류
depends_on:
  - mysql

# depends_on + condition: healthcheck 통과할 때까지 기다림
depends_on:
  mysql:
    condition: service_healthy    # mysql의 healthcheck가 healthy 상태가 될 때까지 대기
  redis:
    condition: service_started    # 단순히 시작됐으면 OK (Redis는 빠르게 준비됨)
```

---

## 주요 Compose 명령어

```bash
# 시작 (백그라운드)
docker compose up -d

# 특정 서비스만 시작
docker compose up -d mysql redis

# 로그 확인
docker compose logs -f app           # app 서비스 로그 실시간
docker compose logs -f               # 전체 로그
docker compose logs --tail=100 app   # 최근 100줄

# 서비스 상태 확인
docker compose ps

# 서비스 재시작
docker compose restart app

# 서비스 스케일링
docker compose up -d --scale app=3  # app 컨테이너 3개로 늘리기

# 이미지 재빌드 후 시작
docker compose up -d --build app

# 중지 (컨테이너 삭제, 볼륨/네트워크 유지)
docker compose down

# 중지 + 볼륨도 삭제 (DB 완전 초기화)
docker compose down -v

# 중지 + 이미지도 삭제
docker compose down --rmi all

# 컨테이너 내부 명령 실행
docker compose exec app /bin/bash
docker compose exec mysql mysql -u appuser -p mydb
```

---

## 환경별 Compose 파일 분리

공통 설정은 기본 파일에, 환경별 차이점만 오버라이드 파일에 작성한다.

```yaml
# docker-compose.yml (공통 - 모든 환경 공유)
version: '3.8'
services:
  app:
    image: my-spring-app
    networks:
      - backend-network
  mysql:
    image: mysql:8.0
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - backend-network
  redis:
    image: redis:7-alpine
    networks:
      - backend-network
networks:
  backend-network:
volumes:
  mysql-data:
```

```yaml
# docker-compose.dev.yml (개발 환경)
version: '3.8'
services:
  app:
    build: .                         # 로컬 빌드
    ports:
      - "8080:8080"
    volumes:
      - ./src:/app/src               # 소스코드 실시간 반영
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_JPA_SHOW_SQL: "true"   # SQL 로그 활성화
  mysql:
    ports:
      - "3307:3306"                  # 로컬 DB 클라이언트 접근 허용
  redis:
    ports:
      - "6379:6379"
```

```yaml
# docker-compose.prod.yml (운영 환경)
version: '3.8'
services:
  app:
    image: registry.example.com/my-app:${APP_VERSION}
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512m
          cpus: '1.0'
    environment:
      SPRING_PROFILES_ACTIVE: prod
    restart: always
  mysql:
    # 운영에서는 포트 외부 노출 안 함 (보안)
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
```

```bash
# 개발 환경 실행
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# 운영 환경 실행
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 환경 변수 파일 지정
docker compose --env-file .env.prod -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Compose에서 볼륨/네트워크 외부 참조

```yaml
# 이미 존재하는 볼륨/네트워크를 재사용할 때
volumes:
  existing-volume:
    external: true   # 이미 존재하는 볼륨 참조 (없으면 에러)

networks:
  existing-network:
    external: true   # 이미 존재하는 네트워크 참조
```

---

## 프로파일 (Compose Profiles)

특정 서비스를 선택적으로 실행할 때 사용.

```yaml
version: '3.8'
services:
  app:
    image: my-app
    # profiles 없음 → 항상 실행

  mysql:
    image: mysql:8.0
    profiles:
      - full               # --profile full 일 때만 실행

  prometheus:
    image: prom/prometheus
    profiles:
      - monitoring         # --profile monitoring 일 때만 실행

  grafana:
    image: grafana/grafana
    profiles:
      - monitoring
```

```bash
# 기본 (app만 실행)
docker compose up -d

# 모니터링 포함
docker compose --profile monitoring up -d

# 전체 실행
docker compose --profile full --profile monitoring up -d
```
