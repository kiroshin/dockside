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

# 데이터 디렉토리 확인하기
# 일반용 설치판은 버전별 /var/lib/postgresql/[버전]/main 형태임
# 도커용 배포판은 보통 /var/lib/postgresql/data 로 고정됨

# 환경설정파일 복사
$ docker cp temp_db:/usr/share/postgresql/postgresql.conf.sample ./postgresql.conf
$ docker cp temp_db:/usr/share/postgresql/16/pg_hba.conf.sample ./pg_hba.conf

# 컨테이너 내부 postgres 계정의 id 확인: 999 일 것임
$ docker exec temp_db id postgres

# 임시 컨테이너 삭제 - 이미지는 지우지 않음
$ docker rm -f temp_db
```

## postgresql.conf 가이드
| Key                  | 1GB   | 1.5GB  | 2GB    | 4GB   | 8GB   | 16GB  | 24GB  | note  |
| -------------------- | ----- | ------ | ------ | ----- | ----- | ----- | ----- | ----- |
| shared_buffers       | 256MB | 384MB  | 512MB  | 1GB   | 2GB   | 4GB   | 6GB   | 전체 할당량의 25%. 자주 요청하는 데이터 |
| work_mem             | 2MB   | 4MB    | 8MB    | 16MB  | 24MB  | 32MB  | 48MB  | 정렬, 조인 작업당 메모리 = 램 * 0.05 / Max : 접속자 많으면 더 적게 |
| maintenance_work_mem | 64MB  | 96MB   | 128MB  | 256MB | 512MB | 1GB   | 1.5GB | 인덱스 생성, 백업 등 관리용 메모리 |
| effective_cache_size | 768MB | 1.1GB  | 1.5GB  | 3GB   | 6GB   | 12GB  | 18GB  | 전체 할당량의 약 75%. 쿼리 수행 캐시 |
| wal_buffers          | 8MB   | 12MB   | 16MB   | 16MB  | 16MB  | 16MB  | 16MB  | shared_buffers의 약 3% 정도(최대 16MB) 쓰기 버퍼 |
| max_wal_size         | 512MB | 1GB    | 1.5GB  | 2GB   | 2~4GB   | 4~8GB   | 8~16GB  | 로그 캐시: 쓰기속도와 복구시간 사이의 균형을 맞춰야 |
| min_wal_size         | 80MB  | 128MB  | 256MB  | 512MB | 1GB   | 2GB   | 4GB   | 로그 캐시:  |
| max_connections      | 10    | 20     | 30     | 50    | 80    | 100   | 150   | 최대 접속 커넥션 |
|                      | 50명   | 80명   | 100명   | 200명  | 500명 | 1,000명 | 2,000명 | 대략적인 최대 유저 규모 |
| max_worker_processes            | 2   | 4   | 4   | 8   | 8   | 12  | 16  | (CPU 코어의 2배) 시스템 전체의 DB 총 프로세스 |
| max_parallel_workers            | 0   | 2   | 2   | 4   | 4   | 8   | 8   | (CPU 코어에 따라) 병렬쿼리 전체에 투입할 프로세스 수 |
| max_parallel_workers_per_gather | 0   | 1   | 2   | 2   | 2~3   | 4   | 4~6   | (워크프로세스의 절반수준) 개별쿼리 처리할 때 추가로 작업시킬 코어. 0 이면 싱글 |
| checkpoint_completion_target    | 0.9 | 0.9 | 0.9 | 0.9 | 0.9 | 0.9 | 0.9 | 다음 주기 전까지 90%의 시간을 들여서 천천히 나눠서 해 |
| checkpoint_timeout   | 5min  | 5min   | 5min   | 10~15min | 10~15min | 15~30min | 15~30min | 데이터를 디스크로 강제로 옮기는 시간 간격 |
| random_page_cost     | 1.1   | 1.1    | 1.1    | 1.1    | 1.1  | 1.1   | 1.1    | SSD 는 1.1, HDD는 4.0(기본값) |
| autovacuum           | on    | on     | on     | on     | on   | on    | on     | 자동 진공 기능 켜야지 |
| autovacuum_vacuum_scale_factor | 0.05 | 0.05 | 0.05 | 0.05 | 0.05 | 0.05 | 0.05 | 테이블 데이터 5% 가 변경되면 청소 시작 |
| autovacuum_analyze_scale_factor | 0.02 | 0.02 | 0.02 | 0.02 | 0.02 | 0.02 | 0.02 | 테이블 데이터 2% 가 변경되면 정보 갱신 |
| log_temp_files       | 0     | 0      | 0     | 0      | 0     | 0     | 0     | 크기와 상관없이 디스크 캐시를 사용했다면 로그에 기록해 |

