# 볼륨 (Volume) / 네트워크 (Network)

---

## 1. 볼륨 (Volume)

### 개념

컨테이너 내부 파일 시스템은 **컨테이너 종료 시 사라진다**.
DB 데이터, 업로드 파일 등 영구 보존이 필요한 데이터는 볼륨을 사용해야 한다.

```
컨테이너 내부 저장:
  컨테이너 실행 → 파일 생성 → 컨테이너 중지 → 파일 사라짐 (Container Layer)

볼륨 사용:
  컨테이너 실행 → 볼륨에 파일 생성 → 컨테이너 중지 → 파일 유지
  → 새 컨테이너에 같은 볼륨 마운트 → 데이터 그대로
```

### 볼륨 종류 3가지

```
1. Named Volume (관리 볼륨)
   Docker가 직접 관리: /var/lib/docker/volumes/볼륨명/_data/
   → DB 데이터, 앱 데이터 저장 권장
   → docker volume rm으로 명시적 삭제 필요

2. Bind Mount (호스트 마운트)
   호스트의 특정 디렉토리를 컨테이너에 연결
   → 개발 시 소스코드 실시간 반영 용도
   → 호스트 경로가 있어야 함

3. tmpfs Mount
   메모리에만 저장 (디스크 기록 없음)
   → 컨테이너 재시작 시 삭제
   → 민감한 임시 데이터, 성능 중요한 임시 파일
```

### Named Volume

```bash
# 볼륨 생성
docker volume create mysql-data

# 볼륨 목록
docker volume ls

# 볼륨 상세 정보 (저장 경로 등)
docker volume inspect mysql-data

# 볼륨 마운트해서 컨테이너 실행
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \   # 볼륨이름:컨테이너경로
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# 볼륨 삭제 (컨테이너가 사용 중이 아닐 때)
docker volume rm mysql-data

# 사용하지 않는 볼륨 모두 삭제
docker volume prune
```

### Bind Mount

```bash
# 현재 디렉토리를 컨테이너에 마운트 (개발 환경)
docker run -d \
  -v $(pwd)/src:/app/src \       # 호스트경로:컨테이너경로
  -v $(pwd)/config:/app/config \ # 설정 파일 마운트
  -p 8080:8080 \
  my-spring-app

# 읽기 전용 마운트 (컨테이너가 수정 못하게)
docker run -v $(pwd)/config:/app/config:ro my-app
```

### 볼륨 백업과 복원

```bash
# 볼륨 데이터 백업 (tar 아카이브로)
docker run --rm \
  -v mysql-data:/data \               # 백업할 볼륨
  -v $(pwd)/backup:/backup \          # 백업 저장 위치
  alpine tar czf /backup/mysql-backup-$(date +%Y%m%d).tar.gz /data

# 볼륨 복원
docker run --rm \
  -v mysql-data:/data \
  -v $(pwd)/backup:/backup \
  alpine tar xzf /backup/mysql-backup-20240101.tar.gz -C /
```

---

## 2. 네트워크 (Network)

### 개념

컨테이너끼리 통신하려면 같은 **Docker 네트워크**에 연결되어야 한다.
같은 네트워크 안에서는 **컨테이너 이름이 DNS 호스트명**으로 동작한다.

```
컨테이너 A (spring-app) → mysql:3306  ← 컨테이너 이름으로 접근 가능
컨테이너 B (mysql)

JDBC URL: jdbc:mysql://mysql:3306/mydb  ← IP 아닌 컨테이너 이름 사용
```

### 네트워크 종류

```
bridge (기본):
  Docker의 기본 네트워크 드라이버
  단일 호스트에서 컨테이너 간 통신
  사용자 정의 bridge → 컨테이너 이름으로 DNS 자동 해석 가능

  기본 bridge (docker0): 컨테이너 이름으로 통신 불가, IP만 가능
  사용자 정의 bridge:    컨테이너 이름으로 통신 가능 (권장)

host:
  컨테이너가 호스트 네트워크를 그대로 사용
  포트 매핑(-p) 불필요, 네트워크 성능 최고
  격리성 없음 (Linux에서만 제대로 동작)

none:
  네트워크 없음. 완전한 네트워크 격리.
  배치 처리 등 네트워크 불필요한 경우

overlay:
  여러 Docker 호스트에 걸친 네트워크 (Docker Swarm / Kubernetes)
```

### 사용자 정의 브리지 네트워크

```bash
# 네트워크 생성
docker network create my-network
docker network create --driver bridge --subnet 172.20.0.0/16 my-network

# 네트워크 목록
docker network ls

# 네트워크 상세 정보 (연결된 컨테이너 등)
docker network inspect my-network

# 컨테이너를 특정 네트워크로 실행
docker run -d --name mysql     --network my-network mysql:8.0
docker run -d --name redis     --network my-network redis:7
docker run -d --name spring-app --network my-network my-spring-app

# spring-app에서 mysql, redis를 이름으로 접근 가능
# jdbc:mysql://mysql:3306/mydb
# redis://redis:6379

# 실행 중인 컨테이너를 네트워크에 연결/해제
docker network connect my-network existing-container
docker network disconnect my-network existing-container

# 네트워크 삭제
docker network rm my-network
docker network prune  # 사용 안 하는 네트워크 일괄 삭제
```

### 포트 매핑

```bash
# -p 호스트포트:컨테이너포트
docker run -p 8080:8080 my-app       # localhost:8080 → 컨테이너:8080
docker run -p 127.0.0.1:8080:8080 my-app  # 로컬호스트에서만 접근
docker run -p 3307:3306 mysql        # 호스트 MySQL(3306)과 충돌 방지

# 랜덤 포트 매핑
docker run -P my-app                 # 컨테이너의 EXPOSE 포트를 랜덤 호스트 포트로 매핑

# 매핑된 포트 확인
docker port my-app
```

### 네트워크 디버깅

```bash
# 컨테이너에서 다른 컨테이너로 ping
docker exec spring-app ping mysql

# DNS 해석 확인
docker exec spring-app nslookup mysql

# 포트 연결 확인
docker exec spring-app curl -v http://mysql:3306

# 컨테이너 네트워크 설정 확인
docker inspect spring-app | grep -A 20 '"Networks"'
```

---

## 3. 볼륨 + 네트워크 실전 구성 예시

```yaml
# docker-compose.yml (참고)
version: '3.8'

services:
  app:
    image: my-spring-app:1.0
    networks:
      - backend          # 내부 통신용 네트워크
    ports:
      - "8080:8080"      # 외부 노출

  mysql:
    image: mysql:8.0
    networks:
      - backend          # app과 같은 네트워크
    volumes:
      - mysql-data:/var/lib/mysql   # Named Volume
    # ports 없음 → 외부에서 접근 불가, app만 접근 가능

  redis:
    image: redis:7-alpine
    networks:
      - backend
    volumes:
      - redis-data:/data
    # ports 없음

networks:
  backend:
    driver: bridge

volumes:
  mysql-data:
  redis-data:

# app → mysql:3306 통신 O (같은 네트워크)
# 외부 → mysql:3306 통신 X (ports 미노출)  ← 보안
```
