# 참고: https://github.com/github/pages-gem/blob/master/.ruby-version
#      https://github.com/github/pages-gem/blob/master/Dockerfile

# 현재 깃허브 버전은 아주 최신 버전은 아님. 최대한 호환되게.
FROM ruby:3.3-slim-bookworm

# 빌드 도구 설치하고 나서 청소
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    git \
    libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# 컨터이너 내부 작업 디렉토리
WORKDIR /srv

# 지킬과 번들러 설치
RUN gem install jekyll bundler

# 포트 설정
EXPOSE 4000

# 번들러 확인 환경설정
ENV BUNDLE_CONFIRM_INSTALL=1

# 깃허브 환경과 동일하게 실행(Gemfile에 github-pages 설정, 파일변경 감지를 위한 polling)
CMD ["bash", "-c", "bundle install && bundle exec jekyll serve --watch --force_polling --host 0.0.0.0 --port 4000"]
