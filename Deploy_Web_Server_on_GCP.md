# (Backend)Deploy Web Server on Cloud

專案以djangodemo為例
內部資料夾結構:
djangodemo --- blog --- media
djangodemo --- project 
djangodemo --- venv---venv

## GCP VM
* ### Permission denied (publickey)
ssh準備連接遠程服務器卻遭提示” Permission denied (publickey) “, 這是由於您沒有將公鑰( publickey ) 添加到本地ssh 環境造成的，或者是由於多日未進行ssh 登錄操作，本地publickey 失效造成的。
mac os x 環境隔幾天沒有登錄ssh 就會報 “Permission denied ” 啦。只要 使用 ssh-add 命令再次添加一下公鑰即可。

```
ssh-add your_ publickey
```
* ### 設定 Python 開發環境
[GCP文檔](https://cloud.google.com/python/setup?hl=zh-tw)
## Virtualenv

* ### step 1. Install
```
pip install --upgrade virtualenv
```

* ### step 2. Creative virtual environment

```
mkdir [project folder name]

cd [project folder name]

virtualenv --python python3  [venv folder name ex: venv]

```

* ### step 3. Activate virtual environment
```
source [venv folder name]/bin/activate
```

* ### step 4. Deactivate virtual environment

```
deactivate
```

## uWSGI
* ### step 1. Install uWSGI

```
source [venv folder name]/bin/activate

pip3 install uwsgi
```

* ### step 2. Basic test

1.  Create a file called test.py and run uWSGI
    *  記得打開cloud server的port 
```
uwsgi --http :8000 --wsgi-file test.py

## http :8000: 指定使用8000 port 
## wsgi-file test.py: load 指定文件(test.py)
```



2. Check by web browser,if success that means the follow stack component work.

    the web client <-> uWSGI <-> Python
```
http://example.com:8000
```
* ### step 3. Test your Django project
1. check your django  project could work
```
python manage.py runserver 0.0.0.0:8000
```

2. run it using uWSGI:
```
uwsgi --http :8000 --module mysite.wsgi

## module mysite.wsgi: load the specified wsgi module
```

## Nginx

* ### Structure
/etc/nginx/sites-available  存放可用的網站配置
/etc/nginx/sites-enabled    存放已經啟用的網站配置
/etc/nginx/nginx.conf       Nginx的基本配置文件

* ### some command 
read log file
```
sudo vi /var/log/nginx/error.log
```
stop/restart/start nginx
```
sudo /etc/init.d/nginx [stop,start,restart]
```

* ### step 1. Install Nginx & Check that nginx is serving by visiting

```
sudo apt-get install nginx
sudo /etc/init.d/nginx start
```

* ### step 2. Configure nginx for your site

在 /etc/nginx/sites-available 創建一個mysite_nginx.conf文件。

```
# mysite_nginx.conf

# the upstream component nginx needs to connect to
upstream django {
    # server unix:///path/to/your/mysite/mysite.sock; # for a file socket
    server 127.0.0.1:8001; # for a web port socket (we'll use this first)
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
        alias /home/lionel771128/djangodemo/blog/media;  
        # your Django project's media files - amend as required
    }

    location /static {
        alias /home/lionel771128/djangodemo/blog/static; 
        # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /etc/nginx/uwsgi_params; 
        # the uwsgi_params file you installed
    }
}
```

* ### step 3. Enable the nginx config file for your project
```
sudo ln -s [path of the conf file] /etc/nginx/sites-enabled/

EX:
sudo ln -s ~/etc/nginx/sites-available/djangodemo_nginx.conf /etc/nginx/sites-enabled/

```

* ### Deploying static files (Important)

Before running nginx, you have to collect all Django static files in the static folder. First of all you have to edit mysite/settings.py adding:

```python
STATIC_ROOT = os.path.join(BASE_DIR, "static/")
```
```
python3 manage.py collectstatic
```

## Running the Django application with uwsgi and nginx
* ### Configuring uWSGI to run with a .ini file
對ini檔編輯過後，必須kill掉已經在運行的程序，並重新啟動uwsgi，並停止nginx在重新啟動。
```
kill掉全部uwsgi程序
sudo pkill -f uwsgi -9
重新啟動uwsgi
uwsgi --ini /home/lionel771128/djangodemo/djangodemo_wsgi/djangodemo.ini
```
[django-wsgi文檔](https://docs.djangoproject.com/zh-hans/2.2/howto/deployment/wsgi/uwsgi/)
```
[uwsgi]
chdir = /home/lionel771128/djangodemo           #專案主目錄
home = /home/lionel771128/djangodemo/venv/venv  # virutalenv位置
module = project.wsgi:application               #project下的wsgi檔

master = true
process = 4                                     #分配cpu核心數
harakiri = 60                                   #超過該時間無回應就斷線
max-request = 5000

socket = 127.0.0.1:8001                         #使用socket協議連接port
uid = nginx
gid = nginx

pidfile = /home/lionel771128/djangodemo/djangodemo_wsgi/master.pid
daemonize = /home/lionel771128/djangodemo/djangodemo_wsgi/djangodemo.log
vacuum = true
~             
```
