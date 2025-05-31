안전하지 않은 네트워크에서 [sniffing](http://terms.tta.or.kr/dictionary/dictionaryView.do?word_seq=056077-2)없이 안전하게 데이터를 전송할 수 있게 만드는 protocol이다.

client-server 모델이다.

### Public Key Authentication

비밀번호보다 강력한 인증 수단이며, 비밀번호 없이 자동 로그인(automated login)도 지원된다. 

한번 로그인하면 여러 SSH 서버에 로그인이 가능하다. (single sign-on)

key pair는 사용자가 `ssh-keygen`을 이용해서직접 생성하는 것이 일반적이다.

public key는 서버가 가지고 있는다. 그리고 public key를 가지고 데이터를 암호화한다.

private key는 유저가 가지고 있는다. public key에 매핑되는 private key를 가지고 있어야만 인증된다. private key는 복사되면 안되며 보호돼야한다. identity key라고도 부른다.

Public Key Authentication 알고리즘 중 하나가 RSA 알고리즘이다.

## passpharse

private key에 대한 보호는 passphrase를 통해 암호화된다.

private key를 사용할 때마다 passphrase 입력을 통해 decrpytion 된다.

매번 passphrase를 입력해야 하는 것은 아니고, SSH agent가 이를 알아서 처리해준다.

```
ssh-keygen -t rsa -C "your-email-address" -f "github-username"
```

위 명령어는 MAC OS 에서 ssh-keygen을 이용해서 key를 생성하는 명령어다.

## ssh-agent

identity key와 passphrase를 관리하는 프로그램이다.

## config file

SSH client config file로, server와 연결할 때 미리 설정된 명령어들로 연결한다.

## instructions


```sh
# 로컬에 저장된 SSH 키 목록 조회
ls -la ~/.ssh/

# SSH Agent에 로드된 키 확인
ssh-add -l

# SSH Agent에서 키 제거
ssh-add -d ~/.ssh/키파일명
```


[SSH Client Config File Example](https://goteleport.com/blog/ssh-client-config-file-example/)

[How To Work With Multiple Github Accounts on your PC](https://gist.github.com/rahularity/86da20fe3858e6b311de068201d279e3)

[What is SSH Public Key Authentication?](https://www.ssh.com/academy/ssh/public-key-authentication)

[What Is SSH? A Beginner’s Guide To Secure Shell - IPXO](https://www.ipxo.com/blog/what-is-ssh/)