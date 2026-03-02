# SSH (Secure Shell) 완전 정리

---

## 1. SSH가 뭔가

SSH는 **네트워크를 통해 원격 컴퓨터를 안전하게 조작하기 위한 프로토콜**이다.

예전에는 `telnet`, `rsh` 같은 프로토콜로 원격 접속을 했는데, 이것들은 데이터를 **평문(plaintext)으로 전송**했다. 즉, 중간에서 패킷을 가로채면(스니핑) 비밀번호가 그대로 보였다.

SSH는 이걸 해결하기 위해 1995년 만들어졌다.

```
Telnet (옛날 방식):
  내 컴퓨터 →[비밀번호: abc123]→ 인터넷 →[비밀번호: abc123]→ 서버
                      ↑
               누군가 패킷 가로채면 그냥 보임

SSH (현재):
  내 컴퓨터 →[암호화된 데이터: x7k#@!]→ 인터넷 →[복호화]→ 서버
                      ↑
               가로채도 암호화되어 있어 해석 불가
```

### SSH로 할 수 있는 것들

```
1. 원격 서버 쉘 접속 (가장 기본)
   ssh user@server-ip

2. 파일 전송 (SCP, SFTP)
   scp file.txt user@server-ip:/home/user/

3. 포트 포워딩 (터널링)
   로컬 포트 ↔ 원격 포트 연결 (DB 접근 등)

4. 원격 명령어 실행 (쉘 열지 않고 명령만)
   ssh user@server "ls -la /var/log"

5. Git push/pull (GitHub, GitLab이 SSH로 인증)
```

---

## 2. 어떻게 안전한가 - 암호화 원리

SSH는 **공개키 암호화 (Asymmetric Encryption)** 를 기반으로 한다.

### 핵심 개념: 키 쌍 (Key Pair)

```
비밀키(Private Key):
  - 내 컴퓨터에만 존재 (~/.ssh/id_rsa)
  - 절대 외부에 공유 안 함
  - "이게 나임을 증명하는 도장"

공개키(Public Key):
  - 서버에 등록해두는 키 (~/.ssh/id_rsa.pub)
  - 외부에 공유해도 됨
  - "이 도장이 맞는지 확인하는 틀"

원리:
  공개키로 암호화한 데이터 → 비밀키로만 복호화 가능
  비밀키로 서명한 데이터   → 공개키로만 검증 가능
```

### SSH 접속 시 실제 일어나는 일 (Handshake)

```
1. 클라이언트가 서버에 연결 요청

2. 서버가 공개키(Host Key) 전송
   → 클라이언트가 처음 접속 시:
     "The authenticity of host '...' can't be established.
      Are you sure you want to continue connecting? (yes/no)"
   → yes 입력하면 ~/.ssh/known_hosts에 서버 공개키 저장

3. Diffie-Hellman 키 교환으로 세션 키 생성
   → 이후 통신은 이 세션 키로 대칭 암호화 (AES 등)
   → 공개키 암호화는 느리므로, 세션 키 교환에만 사용

4. 사용자 인증
   → 비밀번호 인증: 비밀번호를 암호화해서 전송
   → 공개키 인증: 아래 설명

5. 쉘 세션 시작
```

### 공개키 인증 과정 상세

```
사전 설정:
  내 공개키를 서버의 ~/.ssh/authorized_keys에 등록

접속 시:
  클라이언트: "공개키 A로 인증할게"
  서버: authorized_keys에서 공개키 A 찾음
  서버: 랜덤 값을 공개키 A로 암호화해서 전송 (Challenge)
  클라이언트: 비밀키로 복호화 → 서버에 응답 (Response)
  서버: 응답이 맞으면 인증 성공

→ 비밀키가 내 컴퓨터에 있다 = 내가 맞다는 증명
→ 비밀번호를 네트워크로 전송하지 않아도 됨 (더 안전)
```

---

## 3. 키 생성 및 관리

### 키 생성

```bash
# 기본 (RSA 4096 bit)
ssh-keygen -t rsa -b 4096 -C "설명(보통 이메일)"

# Ed25519 (최신, 더 안전하고 짧음 - 권장)
ssh-keygen -t ed25519 -C "your@email.com"

# 파일명 지정 (여러 키 관리 시)
ssh-keygen -t ed25519 -C "github" -f ~/.ssh/id_github
ssh-keygen -t ed25519 -C "work-server" -f ~/.ssh/id_work

# 실행하면:
# Enter passphrase: 비밀번호 입력 (빈칸 = 설정 안 함)
# ~/.ssh/id_ed25519      ← 비밀키 (절대 공유 금지)
# ~/.ssh/id_ed25519.pub  ← 공개키 (서버에 등록할 것)
```

```bash
# 생성된 키 확인
ls -la ~/.ssh/
cat ~/.ssh/id_ed25519.pub  # 공개키 내용 확인 (서버에 붙여넣을 것)
```

### 키 파일 권한 설정 (중요!)

```bash
# SSH는 권한이 잘못되면 키 인식을 거부함
chmod 700 ~/.ssh            # 디렉토리
chmod 600 ~/.ssh/id_ed25519 # 비밀키 (본인만 읽기)
chmod 644 ~/.ssh/id_ed25519.pub  # 공개키
chmod 600 ~/.ssh/authorized_keys # 서버의 authorized_keys
chmod 600 ~/.ssh/config          # config 파일
```

### 서버에 공개키 등록

```bash
# 방법 1: ssh-copy-id (편리)
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server-ip
# 자동으로 서버의 ~/.ssh/authorized_keys에 추가

# 방법 2: 수동으로 복사
cat ~/.ssh/id_ed25519.pub
# 출력된 내용을 서버의 ~/.ssh/authorized_keys에 붙여넣기
# 서버에서: echo "공개키내용" >> ~/.ssh/authorized_keys

# 방법 3: 파이프로 원격 실행
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

## 4. SSH Config 파일 - 핵심 꿀팁

매번 `ssh -i ~/.ssh/id_work -p 2222 user@192.168.1.100` 치는 건 너무 불편하다. config 파일에 미리 정의해두면 짧은 별칭으로 접속할 수 있다.

```bash
# ~/.ssh/config

# 개인 GitHub
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_github

# 회사 GitHub (계정이 두 개인 경우)
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_github_work

# 운영 서버
Host prod
  HostName 10.0.2.100
  User ubuntu
  IdentityFile ~/.ssh/id_work
  Port 22

# 개발 서버 (비표준 포트)
Host dev
  HostName 10.0.3.50
  User ec2-user
  IdentityFile ~/.ssh/id_dev
  Port 2222
  ServerAliveInterval 60  # 연결 유지 (60초마다 신호)
  ServerAliveCountMax 3

# Bastion 통해서 Private 서버 접속
Host private-app
  HostName 10.0.2.10      # Private IP
  User ubuntu
  IdentityFile ~/.ssh/id_prod
  ProxyJump bastion       # bastion Host를 거쳐서 접속

Host bastion
  HostName 3.34.x.x       # Bastion 퍼블릭 IP
  User ubuntu
  IdentityFile ~/.ssh/id_prod
```

```bash
# config 설정 후 접속
ssh prod             # ssh ubuntu@10.0.2.100 와 동일
ssh dev              # ssh -p 2222 ec2-user@10.0.3.50 와 동일
ssh private-app      # bastion 거쳐서 private 서버 접속

# GitHub 계정이 두 개일 때 클론
git clone git@github-work:company/repo.git  # 회사 계정 사용
git clone git@github.com:myname/repo.git    # 개인 계정 사용
```

---

## 5. SSH Agent

비밀키에 Passphrase를 설정하면 매번 입력해야 해서 불편하다. SSH Agent는 비밀키를 **메모리에 올려두고** 자동으로 인증을 처리한다.

```bash
# SSH Agent 시작 (보통 로그인 시 자동 시작)
eval "$(ssh-agent -s)"

# Agent에 키 추가 (Passphrase 한 번만 입력)
ssh-add ~/.ssh/id_ed25519
ssh-add ~/.ssh/id_github

# 추가된 키 목록 확인
ssh-add -l

# 특정 키 제거
ssh-add -d ~/.ssh/id_ed25519

# 모든 키 제거
ssh-add -D

# 8시간 후 자동 만료로 추가
ssh-add -t 8h ~/.ssh/id_ed25519
```

```bash
# macOS: 키체인에 저장 (재부팅 후에도 유지)
ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# ~/.ssh/config에도 추가
Host *
  AddKeysToAgent yes
  UseKeychain yes  # macOS
  IdentityFile ~/.ssh/id_ed25519
```

---

## 6. SCP / SFTP - 파일 전송

### SCP (Secure Copy)

```bash
# 로컬 → 서버
scp 파일명 user@server:/경로/
scp ./app.jar ubuntu@prod:/home/ubuntu/app/

# 서버 → 로컬
scp user@server:/경로/파일명 ./로컬경로/
scp ubuntu@prod:/var/log/app.log ./logs/

# 디렉토리 전송 (-r 옵션)
scp -r ./dist ubuntu@prod:/var/www/html/

# config에 설정된 별칭 사용
scp ./app.jar prod:/home/ubuntu/

# 포트 지정 (SCP는 -P 대문자)
scp -P 2222 file.txt dev:/home/user/
```

### rsync (대용량 파일 동기화)

```bash
# SCP보다 효율적 - 변경된 파일만 전송
rsync -avz ./dist/ ubuntu@prod:/var/www/html/
# -a: archive (권한, 타임스탬프 등 유지)
# -v: verbose
# -z: 압축 전송

# 삭제된 파일도 반영 (--delete)
rsync -avz --delete ./dist/ ubuntu@prod:/var/www/html/

# dry-run (실제 실행 전 미리 확인)
rsync -avz --dry-run ./dist/ ubuntu@prod:/var/www/html/
```

---

## 7. 포트 포워딩 (터널링)

SSH의 강력한 기능. 방화벽으로 막혀있는 포트를 SSH를 통해 우회해서 접근할 수 있다.

### 로컬 포트 포워딩 (가장 많이 쓰임)

```bash
# 서버의 포트를 내 로컬 포트로 끌어오기
ssh -L 로컬포트:목적지호스트:목적지포트 user@서버

# 예시: Private 서브넷의 RDS에 로컬 DB 툴로 접근
ssh -L 5433:rds-endpoint.amazonaws.com:5432 ubuntu@bastion
# 이제 localhost:5433으로 RDS에 접속 가능
# DB 툴에서: host=localhost, port=5433, db=mydb

# 예시: Private EC2의 웹 앱 확인
ssh -L 8080:10.0.2.100:8080 ubuntu@bastion
# 브라우저에서 localhost:8080 접속 → Private EC2의 8080 포트로 연결

# 백그라운드 실행 (-f: 백그라운드, -N: 명령 실행 안 함)
ssh -fNL 5433:rds-endpoint:5432 ubuntu@bastion
```

### 리모트 포트 포워딩

```bash
# 내 로컬 포트를 서버에 공개 (외부에서 내 개발 서버에 접근)
ssh -R 서버포트:localhost:로컬포트 user@서버

# 예시: 서버의 8080으로 오는 요청을 내 로컬 3000으로
ssh -R 8080:localhost:3000 ubuntu@server
# 외부에서 server:8080 접속 → 내 localhost:3000으로

# 웹훅 테스트, 외부 시연 등에 활용 (ngrok 비슷한 용도)
```

### 동적 포트 포워딩 (SOCKS 프록시)

```bash
# SSH를 SOCKS5 프록시로 사용
ssh -D 1080 ubuntu@server
# 브라우저 프록시 설정: SOCKS5, localhost:1080
# 모든 트래픽이 server를 통해 나감 (VPN 비슷한 효과)
```

---

## 8. 원격 명령어 실행

```bash
# 서버에 쉘 열지 않고 명령만 실행
ssh ubuntu@server "ls -la /var/log"
ssh ubuntu@server "sudo systemctl restart app"

# 여러 명령어
ssh ubuntu@server "cd /app && git pull && ./deploy.sh"

# 로컬 스크립트를 서버에서 실행
ssh ubuntu@server 'bash -s' < ./deploy.sh

# config 별칭 사용
ssh prod "docker ps"
ssh prod "tail -f /var/log/app/app.log"

# 파이프 활용
ssh prod "cat /var/log/nginx/access.log" | grep "ERROR" | wc -l
```

---

## 9. 서버 SSH 설정 강화 (/etc/ssh/sshd_config)

```bash
# /etc/ssh/sshd_config 주요 보안 설정

# 루트 로그인 금지 (절대 루트로 직접 로그인 안 함)
PermitRootLogin no

# 비밀번호 인증 비활성화 (공개키 인증만 허용)
PasswordAuthentication no
ChallengeResponseAuthentication no

# 공개키 인증 활성화
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# 빈 비밀번호 금지
PermitEmptyPasswords no

# 로그인 시도 제한 (3번 실패하면 끊김)
MaxAuthTries 3

# 최대 동시 접속 제한
MaxSessions 10

# 비활성 연결 타임아웃 (300초 = 5분)
ClientAliveInterval 300
ClientAliveCountMax 2

# 포트 변경 (기본 22 → 다른 포트, 보안 효과 미미하지만 로그 노이즈 감소)
Port 22  # 실제로는 22222 같은 포트로 변경하기도 함

# 설정 변경 후 재시작
sudo systemctl restart sshd
# 또는
sudo service ssh restart
```

---

## 10. known_hosts - 서버 신원 확인

```bash
# 처음 접속 시 나오는 메시지
The authenticity of host '10.0.2.100 (10.0.2.100)' can't be established.
ED25519 key fingerprint is SHA256:abc123...
Are you sure you want to continue connecting (yes/no/[fingerprint])?

# yes 입력 시 ~/.ssh/known_hosts에 저장
cat ~/.ssh/known_hosts
# 10.0.2.100 ssh-ed25519 AAAA...

# 서버 재설치 등으로 호스트 키가 바뀌면:
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
# → 중간자 공격(MITM) 가능성 경고
# → 서버 재설치했다면 known_hosts에서 해당 항목 제거
ssh-keygen -R 10.0.2.100        # 특정 호스트 제거
ssh-keygen -R "[hostname]:2222" # 포트 포함된 경우

# 보안 검증 없이 접속 (개발 환경에서만, 운영에서 절대 사용 금지)
ssh -o StrictHostKeyChecking=no ubuntu@server
```

---

## 11. 실전 시나리오

### AWS EC2 접속

```bash
# 1. EC2 생성 시 키 페어 지정 (AWS가 공개키를 EC2에 자동 등록)
# 2. 다운받은 .pem 파일 권한 설정
chmod 400 ~/Downloads/my-key.pem

# 3. 접속
ssh -i ~/Downloads/my-key.pem ec2-user@ec2-퍼블릭IP  # Amazon Linux
ssh -i ~/Downloads/my-key.pem ubuntu@ec2-퍼블릭IP    # Ubuntu

# 4. config에 등록해두기
# Host ec2-prod
#   HostName 3.34.x.x
#   User ubuntu
#   IdentityFile ~/Downloads/my-key.pem

ssh ec2-prod  # 이제 이것만 치면 됨
```

### GitHub SSH 설정

```bash
# 1. 키 생성
ssh-keygen -t ed25519 -C "github" -f ~/.ssh/id_github

# 2. 공개키 복사
cat ~/.ssh/id_github.pub  # 내용 복사

# 3. GitHub Settings → SSH Keys → New SSH Key에 붙여넣기

# 4. config 설정
# Host github.com
#   HostName github.com
#   User git
#   IdentityFile ~/.ssh/id_github

# 5. 테스트
ssh -T git@github.com
# Hi username! You've successfully authenticated...

# 6. 클론할 때 SSH URL 사용
git clone git@github.com:username/repo.git
```

### 배포 스크립트 예시

```bash
#!/bin/bash
# deploy.sh - SSH로 원격 배포

SERVER="ubuntu@prod-server"
APP_DIR="/home/ubuntu/app"
JAR_NAME="app.jar"

echo "=== 빌드 ==="
./gradlew bootJar

echo "=== 서버에 파일 전송 ==="
scp build/libs/$JAR_NAME $SERVER:$APP_DIR/

echo "=== 서버에서 재시작 ==="
ssh $SERVER "
  cd $APP_DIR
  sudo systemctl stop app || true
  cp $JAR_NAME ${JAR_NAME}.backup
  sudo systemctl start app
  echo '배포 완료'
"

echo "=== 헬스체크 ==="
sleep 5
curl -f http://prod-server/actuator/health || echo "헬스체크 실패!"
```

---

## 12. 자주 쓰는 SSH 명령어 모음

```bash
# 접속
ssh user@host
ssh -p 2222 user@host          # 포트 지정
ssh -i ~/.ssh/key user@host    # 키 파일 지정
ssh -v user@host               # verbose (디버깅용)

# 파일 전송
scp file.txt user@host:/path/
scp -r ./dir user@host:/path/
scp user@host:/path/file.txt .

# 원격 명령
ssh user@host "command"
ssh user@host 'bash -s' < script.sh

# 포트 포워딩
ssh -L 8080:localhost:8080 user@host   # 로컬 포워딩
ssh -fNL 5433:rds:5432 user@bastion    # 백그라운드

# 키 관리
ssh-keygen -t ed25519 -C "email"
ssh-copy-id user@host
ssh-add ~/.ssh/id_ed25519
ssh-add -l                   # 로드된 키 목록
ssh-keygen -R hostname       # known_hosts에서 제거

# 연결 테스트
ssh -T git@github.com
ssh -vvv user@host           # 매우 상세한 디버그
```

---

## 13. 왜 이게 중요한가

```
서버를 운영하면 SSH는 거의 유일한 접근 수단이다.

EC2, GCE, Azure VM 등 클라우드 서버 → SSH로 접근
GitHub, GitLab push/pull → SSH 인증
CI/CD 파이프라인 → SSH로 서버 배포
Private 서브넷 DB 접근 → SSH 터널링
Kubernetes 노드 디버깅 → SSH 접근

실무에서 자주 마주치는 상황:
  "서버가 응답이 없어요" → SSH 접속해서 로그 확인
  "배포를 해야 해요" → SSH로 파일 전송 + 재시작
  "로컬에서 운영 DB 봐야 해요" → SSH 터널링으로 접근
  "GitHub 푸시가 안 돼요" → SSH 키 문제
```
