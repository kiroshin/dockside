# compiler
컴파일 테스트용 기본 컨테이너. 다만 이런 컴파일 전용 컨테이너는 큰 필요성이 없음.
아래 소스편집, 개발툴, 시스템 라이브러리, 데이터 등은 시스템에 별 영향을 주지 않음.
다만, java 나 node.js 는 구동 목적에 따라 따로 컨테이너 두는 게 낫다.

## 소스편집
- `nano` 또는 `vim` : 파일 편집
- `git git-lfs`: 소스관리 깃
  * 반드시 `$ git lfs install` 로 한 번은 초기화 해줘야 한다.
- `curl` 또는 `wget` : 다운로드 도구
- `unzip` : 압축풀어


## 개발툴
- `build-essential`: 기본 컴파일러 묶음
- `python3-dev` : 파이썬 C 확장모듈 헤더
- `python3-pip` : 파이썬 패키지 매니저
- `python3-venv` : 파이썬 가상환경
- `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh` : 러스트 컴파일러
- `protobuf-compiler` : 프로토콜버퍼 컴파일러
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
- `ca-certificates` : 공인 인증서 모음
- `gnupg` : 인증 확인 툴


## 지저분한 녀석들
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



