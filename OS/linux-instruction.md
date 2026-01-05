- sudo lsof -i :8081
	- 포트 8081을 사용중인 모든 프로세스 정보 출력
	- lsof (list open files) : 열려있는 파일과 네트워크 연결 확인
	- -i : 네트워크 연결만 확인
	- :8081 포트 8081을 사용하는 네트워크 연결만 확인

- unzip 파일명 -d 목적지
- 경로의 마지막 저장 basename "$PWD" | pbcopy