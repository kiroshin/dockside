# postgres

## 환경설정파일 빼오기
```shell
# 이미지를 받아서 임시로 실행하기
#   -d : Detached mode 컨테이너를 백그라운드에서 실행. 터미널 닫아도 종료 안됨.
#   --name 컨테이너이름: Container Name 실행될 컨테이너에 이름을 붙임
#   -e : Environment variable 컨테이너 내부 환경변수 설정
#
$ docker run -d --name temp_db -e POSTGRES_PASSWORD=temp postgres:16-bookworm


# 환경설정파일 찾기
#  / : 검색위치(루트)
#  -name "문자열" : 찾을 대상
#  -type f : 검색대상을 파일(file) 로만 한정(디렉토리 제외)
#  2>/dev/null : 리눅스 표준에러(Standard Error) 블랙홀로 츨력전달
#
$ docker exec temp_db find / -name "postgresql.conf.sample" -type f 2>/dev/null
/usr/share/postgresql/postgresql.conf.sample
$ docker exec temp_db find / -name "pg_hba.conf.sample" -type f 2>/dev/null
/usr/share/postgresql/16/pg_hba.conf.sample
$ docker exec temp_db find / -name "pg_hba.conf.sample" -type f 2>/dev/null
/usr/share/postgresql/16/pg_hba.conf.sample

# 데이터 디렉토리 확인하기




# 환경설정파일 복사
$ docker cp temp_db:/usr/share/postgresql/postgresql.conf.sample ./postgresql.conf
$ docker cp temp_db:/usr/share/postgresql/16/pg_hba.conf.sample ./pg_hba.conf


# 컨테이너 내부 postgres 계정의 id 확인: 999 일 것임
$ docker exec temp_db id postgres


# 임시 컨테이너 삭제 - 이미지는 지우지 않음
$ docker rm -f temp_db
```



## postgresql.conf


