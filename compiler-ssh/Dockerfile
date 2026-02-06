# 순정 우분투 24.04
FROM ubuntu:24.04

# 질문 없이 SSH 설치, 필수 폴더 생성
RUN apt update && \
    DEBIAN_FRONTEND=nointeractive apt install -y openssh-server && \
    mkdir -p /run/sshd && \
    rm -rf /var/lib/apt/lists/*

# 루트 비번은 일단 0000
RUN echo 'root:0000' | chpasswd

# 외부접속 루트 로그인 허용: sed로 주석 처리된 부분 치환
RUN sed -i.bak 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# 실행(포그라운드) - 반드시 쌍따옴표로
CMD ["/usr/sbin/sshd", "-D"]
