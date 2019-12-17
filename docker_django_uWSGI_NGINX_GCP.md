# (Backend)Deploy Web Server by Docker - From Scott
[Centos安裝docker&docker-compose](https://www.centos.bz/2019/01/centos7-%E5%AE%89%E8%A3%85-docker-%E5%92%8C-docker-compose/)

1、安裝docker
```shell=
# 安装依赖
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

# 添加docker下载仓库
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# 安装docker-ce
sudo yum install docker-ce

# 启动docker-ce
sudo systemctl start docker

# 验证
sudo docker --version

sudo docker run hello-world

```
2、安裝 docker-compose
```shell=
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version
```


# 將當前用戶加入 docker 群組

在 【 Terminal 】中輸入下方指令
```shell=
sudo gpasswd -a ${USER} docker
```

退出當前用戶
在 【 Terminal 】中輸入下方指令
```shell=
sudo su
```

再次切换到 ${USER} 用戶
在 【 Terminal 】中輸入下方指令
```shell=
su ${USER}
```

啟動 docker-compose
在 【 Terminal 】中輸入下方指令
```shell=
docker-compose up -d
```

# 啟動/重啟docker服務

```shell=
## 啟動
sudo systemctl start docker

## 重啟
sudo systemctl restart docker
```

# Prune unused Docker objects
[docker Ref](https://docs.docker.com/config/pruning/)
```shell=
docker image prune

## To remove all images which are not used by existing containers,
## use the -a flag:
docker image prune -a

docker container prune

docker volume prune

docker network prune

docker system prune

docker system prune --volumes
```


[GitHub](https://github.com/Lionel771128/docker_test)
docker-compose.yml
```yaml
version: '3.5'
services:

    db:
      container_name: 'postgres'
      image: postgres
      environment:
        POSTGRES_PASSWORD: password123
      ports:
        - "5432:5432"
        # (HOST:CONTAINER)

      volumes:
        - pgdata:/var/lib/postgresql/data/
        #postgresql unix socket 透過volumes讓container可連入
        - target:/var/run/postgresql/

    nginx:
      container_name: nginx-container
      build: ./nginx
      restart: always
      ports:
        # - "8080:80"
        - "8000:8000"
      volumes:
        # web服務的volumes，Nginx可透過這裡連入
        - api_data:/docker_api
        - ./log:/var/log/nginx
      depends_on:
        - web


    web:
      container_name: djangodemo_docker_test
      build: ./djangodemo_docker_test
      command: uwsgi --ini /docker_api/djangodemo_uwsgi/djangodemo.ini
      restart: always
      volumes:
        # web服務的volumes，Nginx可透過這裡連入
        - api_data:/docker_api
        #連接postgresql unix socket
        - target:/var/run/postgresql/
        # (HOST:CONTAINER)
      depends_on:
        - db
        
# ***重要!!volumes位置，docker container之間透過volumns共享資源
volumes:
  api_data:
  pgdata:
  target:
```
Dockerfile





## uWSGI/Django
.ini file 寫法

```
[uwsgi]
chdir = /docker_api
# 部屬環境是global不需要home
# home = /docker_api/venv

module = project.wsgi:application

master = true
process = 4
harakiri = 60
max-request = 5000

# docker透過volumn在container之間共享資料
socket = /docker_api/djangodemo_uwsgi/socket.sock

chmod-socket = 666
# uid = nginx
# gid = nginx

# pidfile = /docker_api/djangodemo_uwsgi/master.pid
# daemonize = /docker_api/djangodemo_uwsgi/djangodemo.log
vacuum = true
```
Dockerfile

```
FROM python:3.6.8
LABEL maintainer = lionel771128
ENV PYTHONUNBUFFERED 1
RUN mkdir /docker_api
WORKDIR /docker_api

將本機Dockerfile所在目錄之檔案copy到指定資料夾
COPY . /docker_api

安裝套件
RUN pip install -r requirements.txt

```


settings.py檔

```python
# postgresql db設定HOST:
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'PASSWORD': 'password123',
        
        # 走unix socket，postgresql.conf註解預設
        'HOST': '/var/run/postgresql',
        'PORT': 5432,
    }
}
```
requirements.txt

## Nginx
Dockerfile
```
FROM nginx:latest

COPY nginx.conf /etc/nginx/nginx.conf
COPY mysite_nginx.conf /etc/nginx/sites-available/

RUN mkdir -p /etc/nginx/sites-enabled/\
    && ln -s /etc/nginx/sites-available/mysite_nginx.conf /etc/nginx/sites-enabled/


CMD ["nginx", "-g", "daemon off;"]
```
mysite_nginx.conf
```
# mysite_nginx.conf

# the upstream component nginx needs to connect to
upstream django {
    # for a file socket
    server unix://docker_api/djangodemo_uwsgi/socket.sock; 
}

# configuration of the server
server {
    # the port your site will be served on
    listen      8000;
    
    # the domain name it will serve for
    server_name .example.com; # substitute your machine's IP address or FQDN
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    location /media  {
    
        # your Django project's media files - amend as required
        alias /docker_api/blog/media;  
    }

    location /static {
    
        # your Django project's static files - amend as required
        alias /docker_api/blog/static; 
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        
        # the uwsgi_params file you installed
        include     /etc/nginx/uwsgi_params; 
    }
}
```


nginx.conf

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}


http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}


#mail {
#       # See sample authentication script at:
#       # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
#       # auth_http localhost/auth.php;
#       # pop3_capabilities "TOP" "USER";
#       # imap_capabilities "IMAP4rev1" "UIDPLUS";
#
#       server {
#               listen     localhost:110;
#               protocol   pop3;
#               proxy      on;
#       }
#
#       server {
#               listen     localhost:143;
#               protocol   imap;
#               proxy      on;
#       }
#}
```
