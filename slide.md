## Dockerを使ったPerlアプリケーション開発

NDS#39 Niigata.pm
13 Dec 2014
@hayajo

***

### 自己紹介

- Hayato Imai / [@hayajo](https://twitter.com/hayajo)
- 社内SE
- Perl, Go, Docker, etc.
  - [CPAN](http://search.cpan.org/~hayajo/)
  - [NDS#36 Go言語入門](https://gist.github.com/hayajo/9559874)
  - [NDS#37 ゴルーチンと並行性パターン](https://gist.github.com/hayajo/f8917f39c6f0c0828c21)
  - [hayajo/ndsmeetup2-docker](https://github.com/hayajo/ndsmeetup2-docker)

***

## Dockerとは

アプリケーションの開発・配布・実行のためのオープンプラットフォーム
https://www.docker.com/whatisdocker/

***

### Language Stacks

[Docker Hub Registry](https://registry.hub.docker.com/)の[公式リポジトリ](https://registry.hub.docker.com/search?q=library&f=official)として公開されている各言語のDockerイメージ

Docker Hub Official Repos: Announcing Language Stacks | Docker Blog
http://blog.docker.com/2014/09/docker-hub-official-repos-announcing-language-stacks/

perl
https://registry.hub.docker.com/_/perl/

***

### PerlのDockerイメージを利用する(1)

バージョン確認

```
$ docker run --rm perl perl -v  # latest
$ docker run --rm perl:5.18.4 perl -v
```

サブルーチンシグネチャを試す(>= perl5.20)

```
$ docker run --rm perl \
perl -Mfeature=signatures -E \
'sub foo($x,$y){say "$x, $y"}; foo("Hello","World")'
```

***

### PerlのDockerイメージを利用する(2)

コンテナをバックグランドで起動して遊ぶ

```
$ docker run -d -t --name try_perl -v `pwd`:/workspace -w /workspace perl /bin/bash
```

バックグランドのコンテナ内でコマンドを実行する

```
$ docker exec try_perl cpanm --notest Mojolicious
$ docker exec -it try_perl /bin/bash
container$ cpanm --notest IO::Socket::SSL && exit
$ docker exec try_perl mojo get https://api.metacpan.org/v0/author/HAYAJO /name
```

手元のスクリプトをバックグラウンドのコンテナ内で実行する

```
$ cat <<EOF >myapp.pl
use Mojolicious::Lite;
any { text => 'Hello World' };
app->start;
EOF
$ docker exec -it try_perl perl myapp.pl daemon -l https://\*:3000
$ curl -v --insecure https://$(docker inspect -f '{{.NetworkSettings.IPAddress}}' try_perl):3000
```

***

### PerlアプリケーションのDockerイメージ作成

Dockerfile

```
FROM perl
COPY . /app/src
WORKDIR /app/src
RUN cpanm .
CMD ["myapp"]
```

アプリケーションのDockerイメージをビルド

```
docker build -t $USER/myapp .
```

コンテナを起動してアプリケーションを実行

```
docker run $USER/myapp
```

***

### デモ(1)

アプリケーションの準備

```
$ minil new --user $USER App::MyApp && cd App-MyApp
$ echo 'requires "Mojolicious", "0";' >> cpanfile
$ carton install
$ carton exec -- mojo generate lite_app script/myapp
$ git add .
$ carton exec -- minil test
$ git add . && git clean -fdx
```

***

### デモ(2)

Dokcerfile

```
$ cat <<EOF >Dockerfile
FROM perl
COPY . /app/src
WORKDIR /app/src
RUN cpanm .
ENTRYPOINT ["myapp"]
EOF
```

***

### デモ(3)

Dockerイメージのビルド

```
$ docker build -t $USER/myapp .
```

***

### デモ(4)

コンテナの起動とアプリケーションの実行

```
$ docker run --rm \
--name myapp \
-p 3000:3000 \
$USER/myapp prefork
```

***

### デモ(5)

確認

```
$ curl ${$(boot2docker ip 2>/dev/null):-'localhost'}:3000
<!DOCTYPE html>
<html>
  <head><title>Welcome</title></head>
  <body>Welcome to the Mojolicious real-time web framework!
</body>
</html>
```

***

###  ミドルウェアとの連携(1)

DockerのLink機能または環境変数を利用してミドルウェアと連携する

Link機能はlinkオプションで連携先コンテナを指定してコンテナを起動する

```
$ docker run -d --name redis redis
$ docker run --rm --link redis:redis busybox sh -c 'env | grep REDIS_PORT'
REDIS_PORT_6379_TCP_PROTO=tcp
REDIS_PORT_6379_TCP_ADDR=172.17.1.68
REDIS_PORT_6379_TCP_PORT=6379
REDIS_PORT_6379_TCP=tcp://172.17.1.68:6379
REDIS_PORT=tcp://172.17.1.68:6379
```

環境変数を指定する場合はenvオプションかDockerfileのENVを指定する

```
$ docker run --rm -e REDISTOGO_URL=redis://HOST:PORT busybox sh -c 'env | grep REDISTOGO_URL'
REDISTOGO_URL=redis://HOST:PORT
```

***

###  ミドルウェアとの連携(2)

コード例

```
if( my $url = $ENV{REDIS_PORT_6379_TCP} || $ENV{REDISTOGO_URL} ) {
    return Mojo::URL->new($url)->scheme('redis');
}
```

***

### デモ

Redisコンテナをリンクしてアプリケーション実行

```
$ docker run -d --name redis redis
$ (cd App-SimpleChat && docker build -t $USER/simplechat .)
$ docker run --rm \
--name simplechat \
-p 3000:3000 \
--link redis:redis \
$USER/simplechat prefork
```

確認

```
$ open ${$(boot2docker ip 2>/dev/null):-'localhost'}:3000
```

***

### まとめ

- Language Stacksで手軽にPerlの実行環境を用意できる
- アプリケーションの実行環境イメージを用意することで簡単に実行環境を再現できる

***

### デモ: Dockerを使ったマルチサイトホスティング

```
$ docker rm -f `docker ps -qa`
$ (cd /path/to/multi-hosting && fig up)
```

```
$ docker run --name myapp -d --expose 3000 -P $USER/myapp prefork
$ docker run --name redis -d redis \
&& docker run --name simplechat.example.com -d --link redis:redis --expose 3000 -P $USER/simplechat prefork
$ docker run --name jenkins.myhost.net -d -P jenkins
```

```
$ sudo echo "${$(boot2docker ip 2>/dev/null):-'127.0.0.1'} myapp simplechat.example.com jenkins.myhost.net" >> /etc/hosts
```

```
$ open http://myapp
$ open http://simplechat.example.com
$ open http://jenkins.myhost.net
```

***

### おまけ: PerlアプリケーションのHeroku対応

[kazeburo/heroku-buildpack-perl-procfile](https://github.com/kazeburo/heroku-buildpack-perl-procfile)

```
$ cd App-SimpleChat
$ cat Procfile
web: script/simplechat prefork -l http://*:$PORT

$ heroku create $(basename `pwd` | tr 'A-Z' 'a-z') --buildpack=https://github.com/kazeburo/heroku-buildpack-perl-procfile.git
$ heroku addons:add redistogo:nano
$ heroku config:set MOJO_LOG_LEVEL=debug
$ git push heroku master
```

***

