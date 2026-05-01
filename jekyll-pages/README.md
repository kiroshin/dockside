# jekyll-pages

- 테마 제작 수정용 설정으로, 저장 순간 즉각 반영된다.
- 서버의 `/srv/jekyll` 폴더와 컨테이너의 `srv` 폴더가 동기화된다.
- Gemfile 파일을 `/srv/jekyll/Gemfile` 로 복사하고 `_config.yml` 과 `index.html` 을 작성한다.

```
접속: http://주소:4000
컨테이너 빌드: docker compose up --build
컨테이너 정지하고 삭제: docker compose down
컨테이너 정지만 : docker compose stop
컨테이너 정지만 : docker compose stop
컨테이너 실기간 로그 : docker compose logs -f
```
