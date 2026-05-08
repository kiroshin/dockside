# dockside
Docker Test

## 명령
```
# 컨테이너 목록
docker container ls                 : 구 docker ps
docker container ls -a              : 전체 목록
docker container ls -s              : 사이즈

# 컨테이너 삭제
docker container rm 컨이름            : 구 docker rm 컨테이너이름
docker container rm -f 컨이름         : 사용중이어도 강제삭제
docker container rm -v 컨이름         : 볼륨까지 삭제
docker container prune              : 정지된 컨테이너 일괄 삭제(볼륨은 남는다)

# 컨테이너 볼륨
docker volume rm 볼륨이름             : 볼륨 삭제
docker volume prune                 : 안쓰는 볼륨 정리

# 컨테이너 상태
docker container stats              : 구 docker stats

# 컨테이너 내부 명령 실행
# -i (interactive): 표준 입력(stdin)을 열어두기
# -t (tty): 가상 터미널 할당
docker container exec -it 컨테이너 명령어   : 구 docker exec -it 컨테이너 내부명령 => docker exec -it aswood /usr/bin/bash

# 이미지
docker image ls                    : 과거 docker images
docker image ls -a                 : 중간 레이어도 보여줌
docker image build                 : Dockerfile 를 가지고 이미지를 만듦
docker image rm                    : 구 docker rmi 이미지 => 컨테이너 먼저 지우고 이미지 지워라. 이 명령은 태그만 지움.
docker image rm -f 이미지명          : 구 docker rmi -f 이미지 => 강제 삭제. 다 날려버리고 다 지움
docker image prune -a              : 사용 안하는 이미지 모두 지움

# 빌드 캐쉬
docker builder prune                : 필요없는 빌드 캐쉬 정리
docker builder prune -a             : 모든 빌드 캐쉬 정리 - 빌드 이력이 완전히 사라지므로, 다음 빌드 때는 처음부터 모든 레이어를 다시 내려받고 실행하게 된다.
docker system prune                 : 컨테이너, 네트워크, 이미지(dangling)와 함께 빌드 캐시도 세트로 삭제

# 도커 전체 리소스 사용량 요약
docker system df
docker system df -v                 : 상세출력

# 컴포즈 명령어
docker compose up -d                : 컨테이너 없으면 있으면 생성, 데몬으로 실행. 여기서 -d 는 Detached
docker compose up -d --build        : Dockerfile 로 이미지 빌드 후 컨테이너 시작
docker compose down                 : 컨테이너 중지 + 컨테이너 삭제 + 네트워크 삭제(볼륨 안 지움)
docker compose down -v              : 컨테이너 중지 + 컨테이너 삭제 + 네트워크 삭제(볼륨 까지 지움)
docker compose stop                 : 그냥 중지만
docker compose start                : 컨테이너 이미 있을 때 데몬으로 실행
docker compose restart              : 컨테이너 실행 그대로 다시 재시작
docker compose logs                 : 로그 출력
docker compose logs -f              : 로그 실시간 출력

```


## 설치
```shell
# -------------------------
# ca-certificates          (증명서 보관함) - 도커 접속할 때, 그 서버가 신뢰할 수 있는 기관의 것인지 확인
# curl                     (파일 다운로더) - 도커 설치 과정에서 도커의 GPG 키(보안 키)나 설치 파일 다운로드. 기본설
# gnupg                    (디지털 도장 확인 도구) - 파일이 변조되지 않았는지 검증하거나 암호화하는 도구
# docker-ce                    도커엔진. 컨테이너를 관리하고 실행함.
# docker-ce-cli                사용자가 터미널에 docker ...라고 명령을 내리는 도구.
# containerd.io                실제 컨테이너의 생명주기(시작, 정지)를 하부에서 관리함.
# docker-buildx-plugin       이미지를 더 빠르고 스마트하게 빌드해주는 확장 도구.    최신형 터보 부스터
# docker-compose-plugin      여러 개의 컨테이너(DB+앱)를 한 번에 관리함.
# -------------------------

# 시스템 업데이트
sudo apt update && sudo apt upgrade -y

# 사전에 필요한 도구들 설치
sudo apt install -y ca-certificates curl gnupg

# 도커 공식 GPG 키 추가
## 보안 키를 보관할 폴더 생성
sudo install -m 0755 -d /etc/apt/keyrings
## 도커 공식 GPG 키를 다운로드하여 저장
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
## 모든 사용자가 읽을 수 있도록 권한 설정
sudo chmod a+r /etc/apt/keyrings/docker.gpg


# 도커 저장소(Repository)를 시스템에 등록
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 도커 저장소 등록 확인. 위에 등록한 시스템 변수가 등록됨.
cat /etc/apt/sources.list.d/docker.list
--> deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable

# 도커 엔진 세트 설치
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 권한 해결하기
## 현재 사용자($USER)를 docker 그룹에 포함
##    usermod: 사용자 정보 수정
##        -a : append - 기존에 내가 속해 있던 그룹들에서 탈퇴하지 않고, 새로운 그룹을 추가하겠다는 뜻
##        -G : Groups - 그룹 이름
##        항상 -aG를 함께 쓴다. 조심해야 한다.
sudo usermod -aG docker $USER

## 그룹 변경 사항을 현재 터미널 세션에 즉시 적용하고, 해당 사용자 그룹으로 로그인
## 이 그룹을 나오려면 exit 치면 된다.
newgrp docker

# 엔진과 CLI 버전 확인
docker version

# 컴포즈 플러그인 확인
docker compose version

# srv 디렉토리 소유 변경
sudo chown $USER:$USER /srv
```
