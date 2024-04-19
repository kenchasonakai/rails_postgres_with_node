## Docker環境用のファイルを作成する
Dockerfile.dev

```
FROM ruby:3.2.3
ENV LANG C.UTF-8
ENV TZ Asia/Tokyo
RUN apt-get update -qq \
&& apt-get install -y ca-certificates curl gnupg \
&& mkdir -p /etc/apt/keyrings \
&& curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
&& NODE_MAJOR=20 \
&& echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list \
&& wget --quiet -O - /tmp/pubkey.gpg https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
&& echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
RUN apt-get update -qq && apt-get install -y build-essential libpq-dev nodejs yarn vim
RUN mkdir /myapp
WORKDIR /myapp
RUN gem install bundler
COPY . /myapp
```

compose.yml

```
version: '3'
services:
  db:
    image: postgres
    restart: always
    environment:
      TZ: Asia/Tokyo
      POSTGRES_PASSWORD: password
      TZ: "Asia/Tokyo"
    volumes:
      - postgresql_data:/var/lib/postgresql
    ports:
      - 5432:5432
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
      command: bash -c "bundle install && bundle exec rails db:prepare && rm -f tmp/pids/server.pid && ./bin/dev"
    tty: true
    stdin_open: true
    volumes:
      - .:/myapp
      - bundle_data:/usr/local/bundle:cached
      - node_modules:/myapp/node_modules
    environment:
      TZ: Asia/Tokyo
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
volumes:
  bundle_data:
  postgresql_data:
  node_modules:
```

## rails new

### Bootstrapを使用する場合

```
docker compose build
docker compose run --rm web gem install rails
docker compose run --rm web rails new . -d postgresql -j esbuild --css=bootstrap
```

### Tailwind CSSを使用する場合

```
docker compose build
docker compose run --rm web gem install rails
docker compose run --rm web rails new . -d postgresql -j esbuild --css=tailwind
```

### サーバーを立ち上げる前の準備

config/database.ymlのdefaultにhost・username・passwordを追加(compose.ymlの内容に合わせます)
```
default: &default
  adapter: postgresql
  encoding: unicode
  # For details on connection pooling, see Rails configuration guide
  # https://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: db
  username: postgres
  password: password
```

Procfile.devのwebに`-b 0.0.0.0 -p 3000`をつける

```
web: env RUBY_DEBUG_OPEN=true bin/rails server -b 0.0.0.0 -p 3000
js: yarn build --watch
css: yarn watch:css
```

### サーバーを立ち上げる

```
docker compose up
```

### ブラウザで確認

http://localhost:3000
