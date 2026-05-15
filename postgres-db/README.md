# postgres-db

## 환경설정파일 빼오기
```shell
# 이미지를 받아서 임시로 실행하기
#   -d : Detached mode 컨테이너를 백그라운드에서 실행. 터미널 닫아도 종료 안됨.
#   --name 컨테이너이름: Container Name 실행될 컨테이너에 이름을 붙임
#   -e : Environment variable 컨테이너 내부 환경변수 설정
#
$ docker run -d --name temp-db -e POSTGRES_PASSWORD=temp postgres:17-bookworm

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


## 도커 내부 포스트그레와 통싱한 유닉스 소켓
```shell
# 호스트에 디렉트리 만들기
$ sudo mkdir -p /var/run/postgresql

# 권한 부여 (999는 postgres 이미지의 기본 UID입니다)
# 이 디렉토리에서 컨테이너 내부의 postgres 유저가 소켓 파일을 생성하게 된다.
$ sudo chown -R 999:999 /var/run/postgresql

# 접근 권한 설정
# 나중에 다른 프로그램에서 이 소켓을 읽어야 하므로 775 또는 777로 설정한다.
$ sudo chmod -R 775 /var/run/postgresql

# postgresql.conf 수정
# 기본 설정은 /tmp 에 만드는 것으로 되어 있지만 호스트에서 직접 실행되는 것이 아니라 도커 위에서 동작하며
# 컨테이너가 내려가도 호스트와 연결된 볼륨의 파일은 그대로 남는다.
# 그래서 문제가 생겼을 경우 sudo rm -rf /var/run/postgresql/* 한 방으로 상황을 깔끔하게 정리할 수 있다.
# 또한 포스트그레가 Lock 을 만들게 되므로 디렉토리 권한에 예민하다. 따라서 전용 공간을 만드는 것이 좋다.
# 이 소켓(.s.PGSQL.5432)은 PostgreSQL 재시작 시 지우고 다시 만들게 된다.
unix_socket_directories = '/var/run/postgresql'
unix_socket_permissions = 0777

```

그러나 위 방법으로는 재부팅하면 권한이 초기화된다.
```shell
$ sudo nano /etc/tmpfiles.d/postgresql-socket.conf
# 다음 한 줄 입려하고 저장
# 이렇게 하면 재부팅 시마다 OS가 컨테이너보다 먼저 해당 디렉토리를 만들고 권한을 `999`로 세팅해준다.
# d /var/run/postgresql 0775 999 999 - -
# 의미: *   d: 디렉토리를 생성하라.
#      *   0775: 권한을 부여하라.
#      *   999 999: 소유자와 그룹을 999로 설정하라.

```

혹은 postgresql.conf 에서 소켓 경로를 램디스크로 관리되는 /var/run/ 말고 다른 영구 디스크를 지정해도 된다.
디민 양구디스크에 있으면 프로그램이 비정상종료됐을 때 소켓파일을 못 지워서 재시작할 때 기존 소켓 때문에 에러날 수 있다.
그래서 램디스크에 올리는 게 제일 깔끔하다.

러스트 연결 설정
```rust
// (TCP): postgres://user:password@localhost:5432/dbname
// (UDS): host=/var/run/postgresql user=postgres password=your_password dbname=danbi

// 문제는 중간에 퍼센트인코딩을 해야 한다는 것이다. / => "%2F"
// URL 형식을 쓸 때 (인코딩 필요할 수 있음)
let url = r#"postgres://postgres:password@/var%2Frun%2Fpostgresql/danbi"#;
// Danbi 프로젝트에서 사용하실 설정 예시
let conn_str = r#"host=/var/run/postgresql user=유저이름 password=패스워드 dbname=디비이름"#;

// 결국 이렇게 하면 된다.
let user = &config.db_user;
let password = &config.db_password;
let dbname = &config.db_name;
let socket_path = "/var/run/postgresql"; // 유닉스 UDS 경로

// format! 매크로로 연결 문자열 생성
let db_path = format!("host={} port={} user={} password={} dbname={} sslmode=disable", path, port, username, password, dbname);
// 혹은 그냥 이렇게
let db_path = format!("host={path} port={port} user={username} password={password} dbname={dbname} sslmode=disable");

```


## postgresql.conf 가이드 (자료양 100G 가정)

| Key                             | 1GB   | 1.5GB  | 2GB    | 4GB    | 8GB    | **10GB** | 16GB | 24GB   | note  |
| ------------------------------- | ----- | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ----- |
| listen_addresses                | *     | *      | *      | *      | *      |   *    | *      | *      | localhost 로 되어있다면 * 로 바꿔야 외부접속 됨 |
| shared_buffers                  | 256MB | 384MB  | 512MB  | 1GB    | 2GB    | 2560MB | 4GB    | 6GB    | 전체 할당량의 25%. 자주 요청하는 데이터 |
| work_mem                        | 4MB   | 6MB    | 8MB    | 12MB   | 16MB   | 20MB   | 24MB   | 32MB   | 정렬 조인 작업당 메모리 = 램 * 0.05 / Max 그러나 좀 줄여서 보수적으로 계산|
| maintenance_work_mem            | 64MB  | 96MB   | 128MB  | 256MB  | 512MB  | 768MB  | 1GB    | 1.5GB  | 인덱스 생성, 백업 등 관리용 메모리 |
| effective_cache_size            | 768MB | 1126MB | 1536MB | 3GB    | 6GB    | 7680MB | 10GB   | 16GB   | 전체 할당량의 약 75%. 쿼리 수행 캐시 |
| wal_buffers                     | 8MB   | 12MB   | 16MB   | 16MB   | 16MB   | 16MB   | 16MB   | 16MB   | shared_buffers의 약 3% 정도(최대 16MB) 쓰기 버퍼 |
| max_wal_size                    | 1GB   | 1536MB | 2GB    | 4GB    | 6GB    | 8GB    | 8GB    | 12GB   | 로그 캐시: 쓰기속도와 복구시간 사이의 균형을 맞춰야 |
| min_wal_size                    | 256MB | 384MB  | 512MB  | 1GB    | 1536MB | 2GB    | 2GB    | 3GB    | 로그 캐시:  |
| max_connections                 | 10    | 20     | 30     | 50     | 80     | 100    | 150    | 200    | 최대 접속 커넥션 |
| max_worker_processes            | 1     | 2      | 3      | 4      | 6      | 8      | 12     | 16     | (CPU 코어의 2배) 시스템 전체의 DB 총 프로세스 - 설정 안해도... |
| max_parallel_workers            | 0     | 1      | 2      | 3      | 3      | 4      | 8      | 8      | (CPU 코어에 따라) 병렬쿼리 전체에 투입할 프로세스 수 - 설정 안해도... |
| max_parallel_workers_per_gather | 0     | 0      | 1      | 1      | 2      | 2      | 4      | 4      | (워크프로세스의 절반수준) 개별쿼리 처리할 때 추가로 작업시킬 코어. 0 이면 싱글 |
| max_parallel_maintenance_workers | 0     | 0      | 1      | 1      | 2      | 2      | 4      | 4      | (워크프로세스의 절반수준) 청소할 때 프로세스. |
| checkpoint_completion_target    | 0.9   | 0.9    | 0.9    | 0.9    | 0.9    | 0.9    | 0.9    | 0.9    | 다음 주기 전까지 90%의 시간을 들여서 천천히 나눠서 해 |
| checkpoint_timeout              | 10min | 10min  | 15min  | 20min  | 30min  | 30min  | 30min  | 30min  | 데이터를 디스크로 강제로 옮기는 시간 간격 |
| random_page_cost                | 1.1   | 1.1    | 1.1    | 1.1    | 1.1    | 1.1    | 1.1    | 1.1    | SSD 는 1.1, HDD는 4.0(기본값) |
| autovacuum                      | on    | on     | on     | on     | on     | on     | on     | on     | 자동 진공 기능 켜야지 |
| autovacuum_vacuum_scale_factor  | 0.05  | 0.05   | 0.05   | 0.02   | 0.01   | 0.01   | 0.01   | 0.01   | 테이블 데이터 0.05 => 5% 가 변경되면 청소 시작 |
| autovacuum_analyze_scale_factor | 0.02  | 0.02   | 0.02   | 0.01   | 0.005  | 0.005  | 0.005  | 0.002  | 테이블 데이터 0.03 => 2% 가 변경되면 정보 갱신 |
| log_temp_files                  | 0     | 0      | 0      | 0      | 0      | 0      | 0      | 0      | 크기와 상관없이 디스크 캐시를 사용했다면 로그에 기록해 |
| temp_file_limit                 | 25GB  | 25GB   | 25GB   | 25GB   | 25GB   | 50GB   | 50GB   | 50GB   | work_mem보다 큰 작업이 들어오면 DB는 디스크에 임시파일 만든다. 50% 정도 할당 |
| statement_timeout               | 30s   | 30s    | 30s    | 30s    | 30s    | 30s    | 30s    | 30s    | 쿼리 하나가 너무 오래 걸리면 강제종료 |
| lock_timeout                    | 10s   | 10s    | 10s    | 10s    | 10s    | 10s    | 10s    | 10s    | 다른 작업으로 락이 걸렸을 때 일정 시간이 지나면 포기 |



## pg_hba.conf 가이드

파일 내부의 `@authmethodhost@` 처럼 `@`로 둘러쌓인 곳은 대체한다. 지워야 한다.

```
# 로컬(유닉스 소켓) 접속: 모든 접속 허용 - 비번 넣을 경우 trust 도 scram-sha-256 로 변경
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

# DB를 안전하게 끄려면 아래와 같이 하고 docker compose stop
postgres=# CHECKPOINT;

# 관리자 비번변경
postgres=# ALTER USER postgres WITH PASSWORD '새로운비밀번호';

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

#
SELECT
    datname AS database_name,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

# ------------------------------------------------------------
# DB 삭제
postgres=# DROP DATABASE tester_db;
# 누가 접속중이면 강제로 끊고 지워
postgres=# SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'tester_db'; DROP DATABASE tester_db;
# 유저 tester가 가진 모든 권한(테이블 소유권 등)을 강제 정리
DROP OWNED BY tester;
# 유저 삭제
DROP USER tester;
# 확인
\du

# ------------------------------------------------------------
# 테이블 백업(pg_dump)
#   -F d: 디렉토리(Directory) 만들어서 거기에 종류별로 풀어낸다.
#   -F c : 커스텀 압축 파일 (.dump)
#   -F t : 타르 압축 파일 (.tar) 복원은 pg_restore 로
#   -j 숫자: 코어 숫자개를 동시에 써서 병렬로 백업
#   -f 경로: 백업 데이터가 저장될 경로
#   -d 디비: (생략가능)대상 데이터베이스
#   => pg_dump -F d -j 4 -f /backups/dump_file -d humandb
#   -t 테이블: 특정 테이블만 백업
#   => pg_dump -t person -f ~/Desktop/person_dump_file.sql humandb
#
# 테이블 복원(psql)
#   -d 디비: 대상 데이터베이스
#   -f 경로: 백업 데이터가 저장될 경로
#  => psql -d humandb -f ~/backups/person_dump_file.sql
#
# 테이블 복원(pg_restore)
#   -c: (Clean)기존 테이블을 자동으로 드롭한 뒤 복원
#   -d 디비명: 복원할 데이터베이스
# ------------------------------------------------------------
# 전체 백업(pg_dumpall)
#   => pg_dumpall -f ~/backups/humandb_backup.sql -d humandb
#   => pg_dump -F d -j 4 -f ~/backups/humandb_dir_backup -d humandb
#
# 전체 복원
#   => psql -f ~/backups/humandb_backup.sql humandb
#   => pg_restore -c -j 4 -d humandb ~/backups/humandb_dir_backup
# ------------------------------------------------------------
부분 humandb=> pg_dump -F t -t person -f ~/backups/2026-01-01-humandb-person.tar humandb
부분 humandb=> pg_restore -c -d humandb ~/backups/2026-01-01-humandb-person.tar

전체 humandb=> pg_dump -F t -f /backups/humandb_total.tar humandb
전체 humandb=> pg_restore -c -d humandb /backups/humandb_total.tar

도커밖으로 처리는 ==>>> $ docker exec -t postgres-db pg_dump -F t -U postgres humandb > /backups/humandb-2026-05-15.tar
도커안에서 처리는 ==>>> $ docker exec -it postgres-db pg_dump -F t -U postgres -f /backups/humandb-2026-05-15.tar humandb

잘 됐는지 스캔: 테이블 하나를 조회해본다. ==>>> $ docker exec -it postgres-db pg_restore -l /backups/humandb-2026-05-15.tar | grep person

```



