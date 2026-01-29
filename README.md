# dockside
Docker Test

## 설치
```shell
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

# 도커 저장소 등록 확인
cat /etc/apt/sources.list.d/docker.list
--> deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable

# 도커 엔진 세트 설치
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 권한 해결하기
## 현재 사용자($USER)를 docker 그룹에 포함시킵니다.
##    usermod: 사용자 정보 수정
##        -a : append - 기존에 내가 속해 있던 그룹들에서 탈퇴하지 않고, 새로운 그룹을 추가하겠다는 뜻
##        -G : Groups - 그룹 이름
##        항상 -aG를 함께 쓴다.
sudo usermod -aG docker $USER

## 그룹 변경 사항을 현재 터미널 세션에 즉시 적용하고, 해당 사용자 그룹으로 로그인
## 이 그룹을 나오려면 exit 치면 된다.
newgrp docker

# 엔진과 CLI 버전 확인
docker version

# 컴포즈 플러그인 확인
docker compose version
```
