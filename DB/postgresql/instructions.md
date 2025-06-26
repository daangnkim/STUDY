- brew services start postgresql
	- Homebrew 패키지 매니저를 이용하여 PostgreSQL 실행

- psql -U $(whoami) -d postgres
	- `-U` 접속할 사용자 지정
	- `whoami` 현재 로그인한 사용자 이름을 반환
	- `-d` 접속할 데이터베이스 이름 지정
	- `postgres` postgresql 설치시 기본적으로 생성되는 데이터베이스

- \l
	- 데이터베이스 목록 확인

- \q
	- 종료

- CREATE USER your_username WITH ENCRYPTED PASSWORD 'your_password';
	- `CREATE USER` 새 사용자 생성 명령어
	- `your_username` 생성할 사용자명
	- `WITH ENCRYPTED PASSWORD` 암호화된 비밀번호 설정
	- `'your_password'` 실제 비밀번호

- ALTER USER your_username WITH SUPERUSER;
	- `ALTER USER` 기존 사용자의 속성 변경
	- `WITH SUPERUSER` 슈퍼유저 권한 부여

- GRANT ALL PRIVILEGES ON DATABASE blog_nextjs_crud TO your_username;
	- `GRANT ALL PRIVILEGES` 모든 권한 (생성, 읽기, 쓰기, 삭제 등)
	- `ON DATABASE blog_nextjs_crud` blog_nextjs_crud 데이터베이스에 대해
	- `TO your_username` your_username 사용자에게 권한 부여

- 