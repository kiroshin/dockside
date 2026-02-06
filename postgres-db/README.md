# postgres-db

## 환경설정파일 빼오기
```shell
# 이미지를 받아서 임시로 실행하기
#   -d : Detached mode 컨테이너를 백그라운드에서 실행. 터미널 닫아도 종료 안됨.
#   --name 컨테이너이름: Container Name 실행될 컨테이너에 이름을 붙임
#   -e : Environment variable 컨테이너 내부 환경변수 설정
#
$ docker run -d --name temp-db -e POSTGRES_PASSWORD=temp postgres:16-bookworm

# 환경설정파일 찾기
#  / : 검색위치(루트)
#  -name "문자열" : 찾을 대상
#  -type f : 검색대상을 파일(file) 로만 한정(디렉토리 제외)
#  2>/dev/null : 리눅스 표준에러(Standard Error) 블랙홀로 츨력전달
#
$ docker exec temp-db find / -name "postgresql.conf.sample" -type f 2>/dev/null
/usr/share/postgresql/postgresql.conf.sample
$ docker exec temp-db find / -name "pg_hba.conf.sample" -type f 2>/dev/null
/usr/share/postgresql/16/pg_hba.conf.sample

# 데이터 디렉토리 확인하기
# 일반용 설치판은 버전별 /var/lib/postgresql/[버전]/main 형태임
# 도커용 배포판은 보통 /var/lib/postgresql/data 로 고정됨

# 환경설정파일 복사
$ docker cp temp-db:/usr/share/postgresql/postgresql.conf.sample ./postgresql.conf
$ docker cp temp-db:/usr/share/postgresql/16/pg_hba.conf.sample ./pg_hba.conf

# 컨테이너 내부 postgres 계정의 id 확인: 999 일 것임
$ docker exec temp-db id postgres

# 임시 컨테이너 삭제 - 이미지는 지우지 않음
$ docker rm -f temp-db
```

## 데이터를 저장할 공간
머신의 `/srv/pgdata` 디렉토리에 데이터가 저장되게 할 것이다.

1. `pgdata` 디렉토리 만든다.
2. 소유권을 컨테이너의 postgres UID 인 999 로 해준다.
3. compose 의 volumes 작성 시 반영한다.

```shell
kiro@server:~$ cd /srv
kiro@server:/srv$ mkdir -p pgdata
kiro@server:/srv$ sudo chown -R 999:999 pgdata
kiro@server:/srv$ ls -l
total 20
drwx------ 2 kiro kiro             16384 Jan  8 01:59 lost+found
drwxrwxr-x 2  999 systemd-journal   4096 Jan  8 08:52 pgdata
kiro@server:/srv$
```

머신에는 999 유저가 없지만 999 그룹은 systemd-journal 이다. 따라서 위와 같이 표시된다. 정상이다.

## postgresql.conf 가이드

| Key                  | 1GB   | 1.5GB  | 2GB    | 4GB   | 8GB   | 16GB  | 24GB  | note  |
| -------------------- | ----- | ------ | ------ | ----- | ----- | ----- | ----- | ----- |
| listen_addresses     | *     | *      | *      | *     | *     | *     | *     | localhost 로 되어있다면 * 로 바꿔야 외부접속 됨 |
| shared_buffers       | 256MB | 384MB  | 512MB  | 1GB   | 2GB   | 4GB   | 6GB   | 전체 할당량의 25%. 자주 요청하는 데이터 |
| work_mem             | 2MB   | 4MB    | 8MB    | 16MB  | 24MB  | 32MB  | 48MB  | 정렬 조인 작업당 메모리 = 램 * 0.05 / Max 그러나 좀 줄여서 보수적으로 계산|
| maintenance_work_mem | 64MB  | 96MB   | 128MB  | 256MB | 512MB | 1GB   | 1.5GB | 인덱스 생성, 백업 등 관리용 메모리 |
| effective_cache_size | 768MB | 1.1GB  | 1.5GB  | 3GB   | 6GB   | 12GB  | 18GB  | 전체 할당량의 약 75%. 쿼리 수행 캐시 |
| wal_buffers          | 8MB   | 12MB   | 16MB   | 16MB  | 16MB  | 16MB  | 16MB  | shared_buffers의 약 3% 정도(최대 16MB) 쓰기 버퍼 |
| max_wal_size         | 512MB | 1GB    | 1.5GB  | 2GB   | 2~4GB   | 4~8GB   | 8~16GB  | 로그 캐시: 쓰기속도와 복구시간 사이의 균형을 맞춰야 |
| min_wal_size         | 80MB  | 128MB  | 256MB  | 512MB | 1GB   | 2GB   | 4GB   | 로그 캐시:  |
| max_connections      | 10    | 20     | 30     | 50    | 80    | 100   | 150   | 최대 접속 커넥션 |
|                      | 50명   | 80명   | 100명   | 200명  | 500명 | 1,000명 | 2,000명 | 대략적인 최대 유저 규모 |
| max_worker_processes            | 2   | 4   | 4   | 8   | 8   | 12  | 16  | (CPU 코어의 2배) 시스템 전체의 DB 총 프로세스 - 설정 안해도... |
| max_parallel_workers            | 0   | 2   | 2   | 4   | 4   | 8   | 8   | (CPU 코어에 따라) 병렬쿼리 전체에 투입할 프로세스 수 - 설정 안해도... |
| max_parallel_workers_per_gather | 0   | 1   | 2   | 2   | 2~3   | 4   | 4~6   | (워크프로세스의 절반수준) 개별쿼리 처리할 때 추가로 작업시킬 코어. 0 이면 싱글 |
| checkpoint_completion_target    | 0.9 | 0.9 | 0.9 | 0.9 | 0.9 | 0.9 | 0.9 | 다음 주기 전까지 90%의 시간을 들여서 천천히 나눠서 해 |
| checkpoint_timeout   | 5min  | 5min   | 5min   | 10~15min | 10~15min | 15~30min | 15~30min | 데이터를 디스크로 강제로 옮기는 시간 간격 |
| random_page_cost     | 1.1   | 1.1    | 1.1    | 1.1    | 1.1  | 1.1   | 1.1    | SSD 는 1.1, HDD는 4.0(기본값) |
| autovacuum           | on    | on     | on     | on     | on   | on    | on     | 자동 진공 기능 켜야지 |
| autovacuum_vacuum_scale_factor | 0.05 | 0.05 | 0.05 | 0.05 | 0.05 | 0.05 | 0.05 | 테이블 데이터 5% 가 변경되면 청소 시작 |
| autovacuum_analyze_scale_factor | 0.02 | 0.02 | 0.02 | 0.02 | 0.02 | 0.02 | 0.02 | 테이블 데이터 2% 가 변경되면 정보 갱신 |
| log_temp_files       | 0     | 0      | 0     | 0      | 0     | 0     | 0     | 크기와 상관없이 디스크 캐시를 사용했다면 로그에 기록해 |



## pg_hba.conf 가이드

파일 내부의 `@authmethodhost@` 처럼 `@`로 둘러쌓인 곳은 대체한다. 지워야 한다.

```
# 로컬(유닉스 소켓) 접속: 모든 접속 허용
local   all             all                                     trust
# IPv4 루프백 접속
host    all             all             127.0.0.1/32            scram-sha-256
# IPv4 외부 접속 허용
host    all             all             0.0.0.0/0               scram-sha-256
# IPv6 접속 (필요시)
host    all             all             ::1/128                 scram-sha-256
# 복제용 설정 (나중에 확장 대비)
host    replication     all             127.0.0.1/32            scram-sha-256
```



## 접속
```
# 내부 접속
$ docker exec -it postgres-db psql -U postgres

# 설정한 버퍼가 맞는지 확인
postgres=# SHOW shared_buffers;

# 테스터 유저 만들기
postgres=# CREATE USER tester WITH ENCRYPTED PASSWORD 'tester';

# 비번도 변경 가능
postgres=# ALTER USER tester WITH ENCRYPTED PASSWORD '0000';

# 테스터 데이터베이스 만들기
postgres=# CREATE DATABASE tester_db OWNER tester;

# 테스터에 자기 데이터베이스의 모든 권한 부여하기
postgres=# GRANT ALL PRIVILEGES ON DATABASE tester_db TO tester;

# 테스터 데이터베이스 들어가기
postgres=# \c tester_db

# 테스터 데이터베이스 나오기
tester_db=# \q

# 물리머신에서 테스터 데이터베이스로 접속해보기
$ docker exec -it postgres-db psql -U tester -d tester_db

# 아무 테이블이나 하나 만들어봐
tester_db=> CREATE TABLE person (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    age INT,
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

# 값도 하나 넣어보고
tester_db=> INSERT INTO person (name, age, phone) VALUES ('tom', 20, '010-1234-5678');

# 출력도 해봐
tester_db=> SELECT * FROM person;
```
