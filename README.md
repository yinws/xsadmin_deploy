# XsAdmin项目部署说明
一步一步教你部署xsadmin，下文中使用CentOS 7 x64，默认用root，如用其他用户登录，执行命令前面要加`sudo`

## 1. 安装前准备
### 1.1 安装LNMP环境
#### 1.1.1 一键LNMP
为了方便建站，建议部署环境直接配置一键LNMP环境，虽然整个项目是用的是Python来实现的，但是LNMP作为非常常用的web环境，
安装后，对于我们管理网站（例如管理虚拟机、MySQL数据库操作等）都将带来很大的帮助。

参考：[LNMP一键安装包](https://blog.linuxeye.cn/31.html)

需要安装的组件：
* PHP：方便使用phpMyAdmin管理MySQL等
* Nginx：作为方向代理服务器
* MySQL：作为主站服务器
* Redis：作为高速Cache，消息队列、异步任务等需要用到的中间件
* phpMyAdmin：方便管理MySQL

请按照上面链接的教程，选择对应组件，安装好LNMP环境

另外，lnmp解压包里面有`vhost.sh`可以方便的管理虚拟主机，`addons.sh`可以方便的给我们安装扩展

#### 1.1.2 其他扩展
##### a 自动部署SSL证书扩展
参考：[OneinStack自动部署Let’s Encrypt证书](https://blog.linuxeye.cn/448.html) （`./addons.sh`脚本在上面安装lnmp环境的解压包文件夹下已经有了，无需再下载oneinstack）
##### b 阻止SSH暴力破解扩展
参考：[fail2ban阻止SSH暴力破解](https://blog.linuxeye.cn/454.html) （可选，也可自行设置，如限制只能用SSH Key登录）

### 1.2 安装Supervisor
#### Supervisor简介
Supervisor是一个Python开发的client/server系统，可以管理和监控*nix上面的进程。目前只支持Python2
#### 1.2.1 安装Supervisor
安装supervisor很简单，通过easy_install就可以安装
```
yum -y install python-setuptools
easy_install supervisor
```
安装完成之后，就可以用`echo_supervisord_conf`命令来生成配置文件
```
echo_supervisord_conf > /etc/supervisord.conf
```
### 1.2.2 supervisor开机脚本
```
wget https://github.com/Supervisor/initscripts/raw/master/redhat-init-mingalevme
mv redhat-init-mingalevme /etc/init.d/supervisord
chmod +x /etc/init.d/supervisord
chkconfig supervisord on  #开机自启动
service supervisord restart  #启动
```

### 1.3 升级Python到3.6
我们的项目环境要求是Python 3.5+，我们需要升级Linux系统默认带的Python2，这次我们升级到最新的Release版本：Python 3.6.0
#### 1.3.1 下载 & install
```
yum -y install xz wget gcc make gdbm-devel openssl-devel sqlite-devel zlib-devel bzip2-devel
wget https://www.python.org/ftp/python/3.6.0/Python-3.6.0.tgz
tar -vxzf Python-3.6.0.tgz
cd Python-3.6.0/
./configure --enable-optimizations --enable-loadable-sqlite-extensions --with-zlib
make && make install

```
这样，Python3就装好了，默认会安装在`/usr/local/bin/python3`，还附带安装了pip(`/usr/local/bin/pip3`)，不和原本的Pytho2冲突
可以查看Python3和pip3的版本
```
python3 -V
pip3 -V
```
#### 1.3.2 安装必要组件
项目在部署的时候，会用到virtualenv，她能很好的帮我们隔离Python运行环境

执行下面的命令进行安装：
```
pip3 install virtualenv
```

## 2. 初始化项目结构
执行如下命令：
```
cd /data/
yum install -y git
git clone https://github.com/alishtory/xsadmin_deploy.git
cd xsadmin_deploy
git clone https://github.com/alishtory/xsadmin.git
```

创建Python的virtualenv环境，执行以下命令：
```
virtualenv env
```

整个项目结构如下：
```
/data/xsadmin_deploy  ##项目根目录
├── env      ## virtualenv根目录
├── xsadmin  ## Django 项目目录
├── logs     ## supervisor日志文件
├── static   ## web静态文件目录
├── upload   ## web文件上传目录
├── conf     ## 一些配置文件
├── LICENSE
├── README.md
```

## 3. 配置你的XsAdmin项目
先确定在项目根目录`/data/xsadmin_deploy`，如果不在，切换到这里

首先，激活virtualenv
```
source env/bin/active
```
切换到Django项目目录
```
cd xsadmin
```
安装项目依赖
```
pip install -r requirements.txt
```
配置项目
```
vi xsadmin/settings_custom.py
```
编辑Django配置文件，
```
SECRET_KEY = '05bk@wyb%nm2-=59n08-mu@^t7+#%x$^kk8_%pm_wcnq6ga!2='  #建议随机改一下加密Key
ALLOWED_HOSTS = ('127.0.0.1', 'xsadmin.org',)  #自定义域名
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'xsadmin',
        'USER':'xsadmin',
        'PASSWORD':'xsadmin', #用phpMyAdmin建立用户和对应的数据库，这里配置对应数据库及密码
        'HOST':'127.0.0.1', #phpMyAdmin建立用户时，可选只允许本地127.0.0.1（注意不要填localhost）
        'PORT':'3306'  #默认的3306
    }
}
STATIC_ROOT = "/data/xsadmin_project/static/"  #改成静态文件目录
MEDIA_ROOT = '/data/xsadmin_project/upload/'   #上传文件目录
SITE_CONFIG = {
    'SITE_NAME':'XS Admin',  #网站标题
    'SITE_DESC':'One powerful tool...', #网站说明
}
INIT_TRANS_ENABLE = 6*1073741824 #默认用户流量是6G
#极验证，自己去geetest.com注册生成
GEE_CAPTCHA_ID = 'b46d1900d0a894591916ea94ea91bd2c'
GEE_PRIVATE_KEY = '36fc3fe98530eea08dfc6ce76e3d24c4'
```
同步数据库
```
python manage.py migrate
```
创建管理员帐号
```
python manage.py createsuperuser  #然后按照提示输入，完成创建超管帐号
```
同步静态资源文件
```
python manage.py collectstatic
```

## 4. 配置Supervisor
修改配置文件
```
vi /etc/supervisord.conf
```
`shift`+`G`，跳转至文件末尾，`i`插入，在配置文件末尾加入
```
[include]
files = /data/xsadmin_project/conf/supervisor_*.ini
```
重启Supervisor
```
service supervisord restart
```
查看Supervisor管理的进程状态
```
supervisorctl status
```
确定 `uwsgi_xsadmin`、`celery_worker`、`celery_beat` 都是`RUNNING`状态

## 5. 配置Nginx虚拟主机

进入lnmp解压文件夹下，执行`./vhost.sh`命令（可参考[LNMP一键安装包](https://blog.linuxeye.cn/31.html)），完成创建虚拟主机。

本次以`xsadmin.org`域名为例，来进行说明

创建完虚拟主机后，运行`vi /usr/local/nginx/conf/vhost/xsadmin.org`，编辑虚拟主机配置文件

### 5.1 注释root目录

找到`#root /data/wwwroot/xsadmin.org;`，可以把root注释掉，因为我们这里是Django项目

### 5.2 配置匹配规则

找到`location`开始的位置，删除默认配置，改为如下配置：
```
  charset utf-8; #默认编码方式
  client_max_body_size 75M;

  #文件上传目录
  location /upload {
    alias /data/xsadmin_project/upload;
    expires 30d;
    access_log off;
  }

  #静态资源目录，让nginx处理静态文件，速度更快。
  #需要在Django里面配置STATIC_ROOT并运行python manage.py collectstatic
  location /static {
    alias /data/xsadmin_project/static;
    expires 30d;
    access_log off;
  }

  # 其他的请求全部交给Python的uWSGI来处理
  location / {
     uwsgi_pass unix:///var/run/xsadmin.sock; #注意这里要和之前的匹配
     include uwsgi_params;
  }
```
改完后，`/usr/local/nginx/conf/vhost/xsadmin.org`配置文件如下：
```
server {
  listen 80;
  listen 443 ssl http2;
  ssl_certificate /usr/local/nginx/conf/ssl/xsadmin.org.crt;
  ssl_certificate_key /usr/local/nginx/conf/ssl/xsadmin.org.key;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
  ssl_prefer_server_ciphers on;
  ssl_session_timeout 10m;
  ssl_session_cache builtin:1000 shared:SSL:10m;
  ssl_buffer_size 1400;
  add_header Strict-Transport-Security max-age=15768000;
  ssl_stapling on;
  ssl_stapling_verify on;
  server_name xsadmin.org;
  index index.html index.htm index.php;
  include /usr/local/nginx/conf/rewrite/none.conf;
  #root /data/wwwroot/xsadmin.org;   #可以把root注释掉，我们是Django项目
  if ($ssl_protocol = "") { return 301 https://$server_name$request_uri; }  #我们开启了全站SSL

  #error_page 404 = /404.html;
  #error_page 502 = /502.html;

  #上面的配置可以用./vhost.sh默认生成的

  #下面是我们需要修改的地方

  charset utf-8;
  client_max_body_size 75M;
  location /upload {
    alias /data/xsadmin_project/upload;
    expires 30d;
    access_log off;
  }

  location /static {
    alias /data/xsadmin_project/static;
    expires 30d;
    access_log off;
  }

  location / {
     uwsgi_pass unix:///var/run/xsadmin.sock;
     include uwsgi_params;
  }
}
```

配置完成后，重启Nginx服务
```
nginx -s reload
```
访问您的网站，Done~