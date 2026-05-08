# compiler
컴파일 테스트용 기본 컨테이너. 이런 컴파일 전용 컨테이너는 큰 필요성이 없다.
아래 소스편집, 개발툴, 시스템 라이브러리, 데이터 등은 시스템에 별 영향을 주지 않기 때문이다.
다만, java 나 node.js 는 구동 목적에 따라 따로 컨테이너 두는 게 낫다.

## 소스편집
- `nano` 또는 `vim` : 파일 편집
- `git git-lfs`: 소스관리 깃
  * 반드시 `$ git lfs install` 로 한 번은 초기화 해줘야 한다.
- `curl` 또는 `wget` : 다운로드 도구


## 개발툴
- `build-essential`: 기본 컴파일러 묶음
- `python3-dev` : 파이썬 C 확장모듈 헤더
- `python3-pip` : 파이썬 패키지 매니저
- `python3-venv` : 파이썬 가상환경
- `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh` : 러스트 컴파일러
- `cmake` : 이건 C++ 빌드할 때
- `gdb` : 디버거


## 시스템 라이브러리
- `libxml2-dev` : xml 파싱 라이브러리
- `libssl-dev` : SSL/TLS 암호화 라이브러리. 통신쪽 개발에 쓴다.
- `libffi-dev` : C인터페이스 브릿지
- `libaio1t64` : 비동기 입출력 사용에 쓰는데, 개발에 쓰려면 `libaio-dev` 를 더 설치
  * 고집쟁이 오라클이 인식할 수 있게 심볼릭 만들어줘야 해. `libaio.so.1` 를 찾으니까.
  * `$ sudo ln -s /usr/lib/x86_64-linux-gnu/libaio.so.1t64 /usr/lib/x86_64-linux-gnu/libaio.so.1`
  * 이건 경로로 직접 들어가서 `$ ls -l libaio*.*` 로 직접 확인해보고 나서 `$ sudo ln -s libaio.so.1t64 libaio.so.1` 하는 게 안전.
  * 왜냐면 arm 계열은 `ln -s /usr/lib/aarch64-linux-gnu/libaio.so.1t64 /usr/lib/aarch64-linux-gnu/libaio.so.1` 니까.


## 데이터
- `pkg-config` : 종종 패키지 찾는다고 이걸 요구하는 경우가 있음
- `ca-certificates` :  신뢰할 수 있는 기관 인증서 모음
- `gnupg` : 인증 확인 툴
- `protobuf-compiler` : grpc 를 위한 프로토콜 버퍼. `protoc` 호출
- `unzip` : 압축풀기


## 모니터링
- `cron` : 크론탭 스케쥴러
- `htop` : 그냥 top 보다는 조금 더 보기 편해


## 방화벽
- `ufw` : iptables 상위 래퍼


## 그외 지저분한 녀석들
- `openjdk-17-jdk` : 자바17
  * amd64 환경변수 `~/.bashrc` 에 추가. 이런 건 직접 경로를 확인해보고 넣는 게 좋다.
    1. `export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64`
    2. `export PATH=$PATH:$JAVA_HOME/bin`
  * arm 환경변수 - 이름이 살짝 다르다
    1. `export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-arm64`
    2. `export PATH=$PATH:$JAVA_HOME/bin`
  * 그리고 `source ~/.bashrc` 로 로딩
- node.js 는 apt 로 설치하면 매우 낮은 버전이 깔린다. 24LTS 를 설치하려면 다음처럼.
  * nvm 다운로드 : `curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash`
  * node.js 설치 : `nvm install 24`
  * 확인
    1. `node -v`
    2. `nvm current`


## 인증서
```shell
# SSH 키 쌍(공개키와 개인키)
$ ssh-keygen -t ed25519 -C "your_email@example.com"
$ ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/my_custom_key   # 저장할 이름 지정

# 확장자 .pub 가 붙은 게 공개키이다. 이걸 출력해서 깃허브 등록하면 된다.
# 다른 계정에 등록했으면 중복 등록 안 된다.
# 전체 계정을 등록해도 되고, 개별 private 계정에 등록해서 컨트롤해도 된다.
$ cat ~/.ssh/id_ed25519.pub

# 퍼미션 - 자동으로 이렇게 되니 딱히 안 해도 된다.
$ chmod 700 ~/.ssh                    # (소유자만 진입 및 읽기/쓰기 가능)
$ chmod 600 ~/.ssh/id_ed25519         # (소유자만 읽기/쓰기 가능)
$ chmod 644 ~/.ssh/id_ed25519.pub     # (소유자는 읽기/쓰기, 나머지는 읽기 가능)
$ chmod 600 ~/.ssh/config

# 연결 테스트
$ ssh -T git@github.com

# 전역(Global) 설정: 이 서버의 모든 프로젝트에 적용
$ git config --global user.email "your_email@example.com"
$ git config --global user.name "Your Name"

# 확인
$ git config --list

# 기존에 http 기반으로 클론받았다면 아래와 같이 업데이트
$ git remote set-url origin git@github.com:유저명/저장소명.git

# 별칭을 달 수도 있다. config 파일에 넣는다
# Host Fast
#     HostName github.com
#     User git
#     IdentityFile ~/.ssh/id_ed25519
#     IdentitiesOnly yes
$ git clone Fast:유지/리포지토리.git

# 개인키는 서버 밖은 안 빠져나가는 게 좋지만, 관리 편의성을 위해 이렇게 접소키로 쓸 수 있다.
# 일단 우분투에서 공개키(.pub) 내용을 authorized_keys 파일에 추가
$ cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# 클라이언트 .ssh/config 파일
# Host myserver
#     HostName 서버아이피
#     User 유저이름
#     IdentityFile ~/.ssh/id_ed25519
$ ssh myserver

```

# 자주 쓰는 깃 명령
```
1. 시작하기
git init                        : 현재 디렉토리를 로컬 Git 저장소로 설정
git clone [URL]                 : 원격 저장소(GitHub 등)의 내용을 내 컴퓨터로 복사해 옵니다

2. 저장하기
git status                      : 현재 변경된 파일들의 상태 확인
git add [파일명]                  : 특정 파일을 Staging Area에 담는다. 모든 파일은 git add .
git commit -m "메시지"            : 한 덩어리로 묶어 저장소에 기록
git commit --amend              : 바로 직전의 커밋 메시지를 수정 혹은 파일 추가

3. 가지치기
git branch                      : 현재 브랜치 목록을 확인
git switch -c [브랜치명]          : 새로운 브랜치를 만들고 바로 이동
git switch [브랜치명]             : 해당 브랜치로 이동
git merge [브랜치명]              : 현재 브랜치에 다른 브랜치의 변경 사항을 합칩니다.
git branch -d [브랜치명]          : 사용이 끝난 브랜치 삭제

4. 동기화하기
git remote add origin [URL]     : 로컬 저장소와 원격 저장소 연결
git push origin [브랜치명]        : 로컬의 커밋들을 원격 저장소로 업로드. 그냥 git push 현재 브랜치에 업로드
git pull                        : 원격 저장소의 변경 내용을 가져와서 내 코드에 즉시 합칩니다.
git fetch                       : 원격 저장소의 변경 내용만 확인하고, 내 코드에 합치지는 않는다.

5. 되돌리기 및 확인
git log                         : 커밋 기록을 확인
git diff                        : 수정했지만 아직 add하지 않은 코드의 차이점
git reset --hard [커밋ID]        : 해당 커밋 시점으로 아예 되돌아간다. (이후 작업물 삭제됨)
git revert [커밋ID]              : 기존 커밋을 취소하는 '새로운 커밋'을 만듦. ((안전한 방법)
```

