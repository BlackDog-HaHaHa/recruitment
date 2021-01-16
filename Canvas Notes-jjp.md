# Canvas Notes

任务名称：Canvas
官方文档：https://github.com/instructure/canvas-lms/wiki/Production-Start
支持平台：Debian家族 | Kubernetes | Docker

任务提交者：蒋金朋

## 介绍

Canvas是为数不多的采用基于本地云服务的学习管理系统的平台，平台所有的功能均通过云服务来完成。本地与托管意味着无需关注用户系统的软硬件状态，无需升级、无需迁移，不存在暂停服务进行系统升级和数据转移的麻烦，也不存在系统超负荷运转的问题。Canvas平台整合了云管理、存储与共享、创建与编辑文件、收发邮件与办公自动化

Canvas是一款优秀的开源学习管理平台，服务端软件为Canvas-lms

## 环境要求

- 程序语言：ruby，JavaScript，HTML等
- OS版本：Ubuntu 16.04 LTS
- 应用服务器：Apache 2.4
- 数据库：PostgreSQL 9.5
- 依赖组件：Ruby2.6
- 服务器配置：2核8G
- 其他组件：Node.js 12.5+，yarn 1.19.1，gem 3.0.3，rake 0.8.7，bundler 2.1.4，Redis 6.0.6

## 安装说明

在本次搭建过程中，服务器商选用的是谷歌云，系统选择上采取了Canvas官方建议的Ubuntu 16.04，配置为2核8G，服务器位于香港地区，方便快速拉取GitHub。由于谷歌云对常见邮件端口做了限制，所以此处无法演示邮件功能。

由于**Ubuntu 14.04 LTS不再支持Node.js 10以后的版本，故放弃使用**，下面安装步骤是结合Canvas官方文档后，基于**Ubuntu 16.04 LTS**编写和测试的Canvas**生产环境**部署教程，如果您使用的是其他系统，请自行修改对应配置命令。

## 用户创建

出于安全考虑，不建议使用root账号搭建Canvas服务

```shell
# 新建普通用户canvas
sudo adduser canvas
# 使canvas用户具备管理员权限
sudo echo 'canvas ALL=(ALL:ALL) ALL' >> /etc/sudoers
```

## 数据库安装和配置

### 安装Postgres

Canvas支持多种数据库适配器，但主要使用**Postgres**和SQLite（用于测试），由于本教程用于设置生产环境，因此我们建议使用Postgres。

无论您是想将Postgres和Canvas安装在同一台服务器上还是单独安装，都没问题，只需要确保二者能够正常通信即可。

如果这是您第一次安装Postgres，你只需要

```shell
canvas@canvas:~$ sudo apt-get install postgresql-9.5
```

请确保您至少运行Postgres 9.5版本！！！

### 在其他服务器上运行Postgres

如果在与Canvas所在服务器之外的服务器上运行Postgres，则需要确保Postgres侦听在正确的地址上。您可以通过编辑`postgresql.conf`和`pg_hba.conf`来使得其可以接受外部连接，您可以参考[Postgres官方文档](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)。

如果您是完全按照步骤来的，您的配置文件应该在如下位置，以下是修改参考

```shell
# 配置文件位置：/etc/postgresql/9.5/main/postgresql.conf
# 修改前
#listen_addresses = 'localhost'         # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
port = 5432                             # (change requires restart)
# 修改后 “*”表示在本地所有地址上监听 如果需要修改监听端口，修改port即可
listen_addresses = '*'         	    # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
port = 5432                             # (change requires restart)


# 配置文件位置：/etc/postgresql/9.5/main/pg_hba.conf
# 修改前
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# 修改后 允许所有地址连接
host    all             all             0.0.0.0/0            md5
```

### 配置Postgres

为Canvas创建一个数据库普通用户**canvas1**（非超级用户，为区别与系统用户canvas，此处故意设置为canvas1，当然你也可以自定义），并为canvas1创建数据库（canvas_production），如果您的Postgres数据库在另外一台服务器上，请将`localhost`替换成运行Canvas的服务器的地址。

```shell
# createuser将提示您输入数据库用户的密码
# 注意 第一次是sudo操作，需输入系统用户canvas的密码 后两次是输入canvas1的密码和确认密码
# 创建cnavas1用户，并设置用户名密码
canvas@canvas:~$ sudo -u postgres createuser canvas1 --no-createdb --no-superuser --no-createrole --pwprompt
# 为canvas1创建数据库 canvas_production
canvas@canvas:~$ sudo -u postgres createdb canvas_production --owner=canvas1
```

将系统用户canvas设置为Postgres超级用户

```shell
canvas@canvas:~$ sudo -u postgres createuser $USER
canvas@canvas:~$ sudo -u postgres psql -c "alter user $USER with superuser" postgres
```

## 获取Canvas

安装Git

```shell
canvas@canvas:~$ sudo apt-get install git-core
```

获取Canvas最新的源代码，此处提醒一下，由于Canvas源代码体积较大，拉取速度可能会由于地区的不同导致速度缓慢，官方的建议是git下来后保留一份副本，由于本次搭建所用服务器位于香港，不用担心速度问题，所以此处不做副本备份。

官方建议我们将代码放在`/var`目录下，所以我们切换到`/var`目录下，并clone源码

```shell
canvas@canvas:~$ cd /var
canvas@canvas:/var$ sudo git clone https://github.com/instructure/canvas-lms.git canvas
canvas@canvas:/var$ cd canvas
canvas@canvas:/var/canvas$ sudo git checkout stable
```

将`/var/canvas`文件夹及其子文件夹的所有者改为当前系统用户canvas，因为我们所有部署工作都是通过canvas这个用户来完成的

```shell
canvas@canvas:/var/canvas$ sudo chown -R canvas /var/canvas
```

## 依赖安装

从2020-10-07版本开始，Canvas现在需要Ruby 2.6。Ruby 2.7+支持但未经测试

### 外部依赖

安装Canvas所需要的Ruby库和软件包，同时安装一些其他依赖。首先添加PPA以获取所需的Ruby版本

```shell
canvas@canvas:/var/canvas$ sudo apt-get install software-properties-common
canvas@canvas:/var/canvas$ sudo add-apt-repository ppa:brightbox/ruby-ng
canvas@canvas:/var/canvas$ sudo apt-get update
```

安装Ruby 2.6

```shell
canvas@canvas:/var/canvas$ sudo apt-get install ruby2.6 ruby2.6-dev zlib1g-dev libxml2-dev libsqlite3-dev postgresql libpq-dev libxmlsec1-dev curl make g++
```

安装Node.js

```shell
canvas@canvas:/var/canvas$ curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
canvas@canvas:/var/canvas$ sudo apt-get install nodejs
```

安装Ruby Gems和Bundler

```shell
canvas@canvas:/var/canvas$ sudo gem install bundler --version 2.1.4
canvas@canvas:/var/canvas$ bundle _2.1.4_ install --path vendor/bundle
```

现在，Canvas首选yarn而不是npm（截至写下这份指导2021/01/16，所需的yarn版本为1.19.1）

```shell
canvas@canvas:/var/canvas$ curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
canvas@canvas:/var/canvas$ echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
canvas@canvas:/var/canvas$ sudo apt-get update && sudo apt-get install yarn=1.19.1-1
```

另外，请确保已安装python（需要contextify软件包）

```shell
# python2.7即可，默认已安装
canvas@canvas:/var/canvas$ sudo apt-get install python
```

安装node模块，**注意：此步耗时较长，如果运行出错，可尝试再次执行这条命令**

```shell
canvas@canvas:/var/canvas$ yarn install
```

## Canvas配置

### 默认配置

Rails代码依赖于少量配置文件，我们用Canvas为我们提供的默认配置文件即可，只需要更改文件名即可

```shell
canvas@canvas:/var/canvas$ for config in amazon_s3 database \
   delayed_jobs domain file_store outgoing_mail security external_migration; \
   do cp config/$config.yml.example config/$config.yml; done
```

### 动态设置配置

当Canvas不使用consul集群的时候需要的配置文件，这里我们只修改文件名，使这个文件成为一个配置文件即可，不修改文件内容

```shell
canvas@canvas:/var/canvas$ cp config/dynamic_settings.yml.example config/dynamic_settings.yml
canvas@canvas:/var/canvas$ nano config/dynamic_settings.yml
```

### 数据库配置

在`config/database.yml`中，修改`production`部分的`username`和`password`为您之前设置的用户名和密码

```shell
# 修改后的production部分应该类似下面的配置
production:
  adapter: postgresql
  encoding: utf8
  database: canvas_production
  host: localhost  # 如果数据库单独安装，需修改为对应的数据库地址
  username: canvas1  # 此前设置的数据库账号
  password: P@ssw0rd  # 此前设置的数据库密码
  timeout: 5000
```

### 邮件配置

配置外发邮件的SMTP服务器，此配置用于配置一个账户，使canvas通过这个账户发送邮件给此canvas的用户

```shell
# 复制模板配置文件和修改配置文件
canvas@canvas:/var/canvas$ cp config/outgoing_mail.yml.example config/outgoing_mail.yml
canvas@canvas:/var/canvas$ nano config/outgoing_mail.yml

# 同样只需要修改production部分，此处演示的是Gmail，其他邮箱请参考相应的邮箱SMTP配置
# 国内邮箱无法登录的话可以尝试将authentication部分的plain修改为login
production:
  address: smtp.gmail.com
  port: 587
  user_name: USERNAME@gmail.com
  password: PASSWORD
  authentication: plain        # plain, login, or cram_md5
  domain: smtp.gmail.com
  outgoing_address: USERNAME@gmail.com
  default_name: Instructure Canvas
```

### 可信URL配置

```shell
test:
  domain: localhost

development:
  domain: "localhost:3000"
  # If you want to set up SSL and a separate files domain, use the following and set up puma-dev from github.com/puma/puma-dev
  # domain: "canvas-lms.test" # for puma-dev
  # files_domain: "canvas-lms.files" # for puma-dev
  # ssl: true

production:
  domain: "canvas.example.com"  # 配置可信域名
  # whether this instance of canvas is served over ssl (https) or not
  # defaults to true for production, false for test/development
  ssl: true
  # files_domain: "canvasfiles.example.com"  # 配置此域名，只有含此域名的请求才能上传文件
```

### 安全设置

给`production`部分的`encryption_key`设置一个至少20个字符的随机字符串（长度大于20即可）

```shell
canvas@canvas:/var/canvas$ cp config/security.yml.example config/security.yml
canvas@canvas:/var/canvas$ nano config/security.yml
```

## 生成资源（初始化资源文件）

Canvas需要先构建一些资产才能正常工作，首先，创建将存储生成的文件的目录。

```shell
canvas@canvas:/var/canvas$ mkdir -p log tmp/pids public/assets app/stylesheets/brandable_css_brands
canvas@canvas:/var/canvas$ touch app/stylesheets/_brandable_variables_defaults_autogenerated.scss
canvas@canvas:/var/canvas$ touch Gemfile.lock
canvas@canvas:/var/canvas$ touch log/production.log
canvas@canvas:/var/canvas$ sudo chown -R canvas config/environment.rb log tmp public/assets \
                              app/stylesheets/_brandable_variables_defaults_autogenerated.scss \
                              app/stylesheets/brandable_css_brands Gemfile.lock config.ru
```

然后运行

```shell
canvas@canvas:/var/canvas$ yarn install
canvas@canvas:/var/canvas$ RAILS_ENV=production bundle exec rake canvas:compile_assets # 耗时较长,中间还会卡顿一会儿，请耐心等待
canvas@canvas:/var/canvas$ sudo chown -R canvas public/dist/brandable_css
```

**后续更新（不需要初始部署）**

如果你修改了 cnavas 的 源码，最好执行下面的这条命令，然后在启动项目，才能让你的修改完全生效

```shell
canvas@canvas:/var/canvas$ RAILS_ENV=production bundle exec rake brand_configs:generate_and_upload_all
```

## 初始化数据库

我们需要生成数据库表并在数据库中添加初始数据。在canvas安装根目录执行下面命令即可，执行命令后，会要求你设置 canvas 默认管理员登录的电子邮件地址和登录密码，还会设置一个登陆后的提示名称，通常是您的组织名称，最后选择是否向canvas开发团队反馈使用信息：opt_in 所有信息，opt_out 不反馈任何信息或匿名反馈

```shell
canvas@canvas:/var/canvas$ RAILS_ENV=production bundle exec rake db:initial_setup
```

请注意，此初始设置将以交互方式提示您创建管理员帐户，默认帐户的名称以及是否向Instructure提交使用情况数据。可以通过设置以下环境变量来“预填充”提示：

| 环境变量                    | 支持的值                               |
| --------------------------- | -------------------------------------- |
| CANVAS_LMS_ADMIN_EMAIL      | 用于默认管理员登录的电子邮件地址       |
| CANVAS_LMS_ADMIN_PASSWORD   | 默认管理员登录密码                     |
| CANVAS_LMS_ACCOUNT_NAME     | 用户看到的帐户名称，通常是您的组织名称 |
| CANVAS_LMS_STATS_COLLECTION | opt_in，opt_out或匿名                  |

## Canvas所有权

出于安全考虑，确保运行Canvas的用户不会有太高的权限

```shell
# 末尾canvas即为您想要用来运行canvas的系统用户
canvas@canvas:/var/canvas$ sudo adduser --disabled-password --gecos canvas canvas
```

确保其他用户无法读取私有的Canvas文件

```shell
canvas@canvas:/var/canvas$ sudo chown canvasuser config/*.yml
canvas@canvas:/var/canvas$ sudo chmod 400 config/*.yml
```

请注意，一旦更改了这些设置，此后就必须使用sudo修改来修改配置文件。

## Apache配置

### 安装

```shell
# 安装Apache及passenger
canvas@canvas:/var/canvas$ sudo apt-get install passenger libapache2-mod-passenger apache2
# 启用mod_rewrite
canvas@canvas:/var/canvas$ sudo a2enmod rewrite
```

### 使Apache支持passenger

```shell
canvas@canvas:/var/canvas$ sudo a2enmod passenger
```

修改`/etc/apache2/mods-enabled/passenger.load`

```shell
# 修改后文件内容如下
LoadModule passenger_module /usr/lib/apache2/modules/mod_passenger.so
PassengerRoot /usr
PassengerRuby /usr/bin/ruby
```

修改`/etc/apache2/mods-enabled/passenger.conf`

```shell
# 修改后文件内容如下
<IfModule mod_passenger.c>
  PassengerRoot /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini
  PassengerDefaultRuby /usr/bin/ruby
  PassengerDefaultUser canvas  # 此处如果设置错误，将导致canvas web页面打开时变成文件列表
</IfModule>
```

### 启用SSL

Debian/Ubuntu默认并未默认启用Apache的SSL模块，因此我们需要手动来启用它

```shell
canvas@canvas:/var/canvas$ sudo a2enmod ssl
```

### 配置Canvas

首先，禁用所有您不想运行的`Apache VirtualHost`。在Debian / Ubuntu上，您可以简单地取消对您无用的`/etc/apache2/sites-enabled`子目录中的任何符号链接的链接。在其他设置中，您可以删除或注释掉不需要的VirtualHosts。

```shell
canvas@canvas:/var/canvas$ sudo unlink /etc/apache2/sites-enabled/000-default.conf
```

然后您需要新建一个VirtualHost

```shell
canvas@canvas:/var/canvas$ sudo nano /etc/apache2/sites-available/canvas.conf
```

配置内容如下，**请注意，在实际配置中，请删掉注释部分，避免程序报错**

```shell
<VirtualHost *:80>
  ServerName canvas.example.com  # 所在服务器地址或域名，在此填本机的地址或域名即可
#  ServerAlias canvasfiles.example.com # 服务器别名， 可不填
  ServerAdmin canvas@example.com
  DocumentRoot /var/canvas/public # 将canvas安装路径下public目录放到apache web容器内，供用户访问
  RewriteEngine On
  RewriteCond %{HTTP:X-Forwarded-Proto} !=https
  RewriteCond %{REQUEST_URI} !^/health_check
  RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [L]
  ErrorLog /var/log/apache2/canvas_errors.log
  LogLevel warn
  CustomLog /var/log/apache2/canvas_access.log combined
  SetEnv RAILS_ENV production
  <Directory /var/canvas/public>
    Allow from all
    Options -MultiViews
  </Directory>
</VirtualHost>
<VirtualHost *:443>  # 此处配置同 <VirtualHost *:80> 配置一样即可
  ServerName canvas.example.com
  ServerAlias canvasfiles.example.com
  ServerAdmin youremail@example.com
  DocumentRoot /var/canvas/public
  ErrorLog /var/log/apache2/canvas_errors.log
  LogLevel warn
  CustomLog /var/log/apache2/canvas_ssl_access.log combined
  SSLEngine on
  BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown
  # the following ssl certificate files are generated for you from the ssl-cert package.
  SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
  SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
  SetEnv RAILS_ENV production
  <Directory /var/canvas/public>
    Allow from all
    Options -MultiViews
  </Directory>
</VirtualHost>
```

**Apache 2.4用户：Apache 2.4中**的`<Directory /var/canvas/public>`已更改。您可能需要这样去修改，否则访问canvas时会报没有权限访问服务器( Forbidden You don't have permission to access / on this server. ）的错误。

```shell
  <Directory /var/canvas/public>
    Options All
    AllowOverride All
    Require all granted
  </Directory>
```

启用新的站点

```shell
canvas@canvas:/var/canvas$ sudo a2ensite canvas
```

SSL证书的说明，可以使用默认配置，但会报不安全网站的警告，如果需要修改，请在canvas.conf中修改以下两处为您自己的证书位置

```
SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
```

### 优化文件下载

如果要在本地而不是在S3中存储上载的文件，则可以使用X-Sendfile标头（nginx中的X-Accel-Redirect）优化文件下载

为 Apache 安装并启用 **mod_xsendfile** 模块

```shell
canvas@canvas:/var/canvas$ sudo apt-get install libapache2-mod-xsendfile
```

使用以下命令确保 **mod_xsendfile** 模块启动并正确运行

```shell
canvas@canvas:/var/canvas$ sudo apachectl -M | grep xsendfile_module
```

将`config/environments/production.rb`复制一份， 复制到`config/environments/production-local.rb`文件中，并将其中的注释的行全部删掉，只保留未注释的，防止以后合并代码时出现错误。

```shell
canvas@canvas:/var/canvas$ grep -v '#' config/environments/production-local.rb | grep -v '^$' > config/environments/production.rb
```

在`/etc/apache2/sites-available/canvas.conf`中添加优化下载的配置， 将下面两行加入到`<VirtualHost>` 中

```shell
XSendFile On
XSendFilePath /var/canvas
```

## Redis缓存配置

Canvas支持两种不同的缓存方法：Memcache和Redis，但是官方更为推荐使用Redis，建议Redis版本为2.6.x或更高版本

安装最新的Redis，执行下面的命令即可

```shell
canvas@canvas:/var/canvas$ sudo add-apt-repository ppa:chris-lea/redis-server
canvas@canvas:/var/canvas$ sudo apt-get update
canvas@canvas:/var/canvas$ sudo apt-get install redis-server
```

安装完成后，启动Redis，并配置其开机自启

```shell
# 启动redis
canvas@canvas:/var/canvas$ sudo systemctl start redis-server.service
# 开机自启
canvas@canvas:/var/canvas$ sudo systemctl enable redis-server.service
```

在Canvas中配置Redis

```
canvas@canvas:/var/canvas$ cd /var/canvas/
canvas@canvas:/var/canvas$ sudo cp config/cache_store.yml.example config/cache_store.yml
canvas@canvas:/var/canvas$ sudo nano config/cache_store.yml
canvas@canvas:/var/canvas$ sudo chown canvas config/cache_store.yml
canvas@canvas:/var/canvas$ sudo chmod 400 config/cache_store.yml
```

将下面内容添加到`cache_store.yml`中

```
test:
    cache_store: redis_store
development:
    cache_store: redis_store
production:
    cache_store: redis_store
```

执行以下命令，配置`redis.yml`

```
canvas@canvas:/var/canvas$ cd /var/canvas/
canvas@canvas:/var/canvas$ cp config/redis.yml.example config/redis.yml
canvas@canvas:/var/canvas$ nano config/redis.yml
canvas@canvas:/var/canvas$ sudo chown canvas config/redis.yml
canvas@canvas:/var/canvas$ sudo chmod 400 config/redis.yml
```

在`config/redis.yml`找到下面几行， 取消掉注释，修改为如下内容

```
production:
   servers:
#   # list of redis servers to use in the ring
   - redis://localhost
```

在我们的示例中，redis与Canvas在同一服务器上运行。这在生产设置中并不理想，因为Rails和Redis都需要大量内存。只需将`'localhost'`更改为您的Redis实例服务器的地址即可。

## 安装QTIMigrationTool

为确保该工具正常需要先安装 [lxml python库](http://lxml.de/)，否则导入课程报错，运行以下命令即可完成安装

```shell
canvas@canvas:/var/canvas$ sudo apt-get install python-lxml
```

此工具需要安装在 canvas 安装目录下的 vender 目录下， 即`/var/canvas/vender`下， 执行以下命令下载源码并配置

```shell
canvas@canvas:/var/canvas$ cd /var/canvas/vendor
canvas@canvas:/var/canvas/vendor$ git clone https://github.com/instructure/QTIMigrationTool.git QTIMigrationTool
canvas@canvas:/var/canvas/vendor$ cd QTIMigrationTool
canvas@canvas:/var/canvas/vendor/QTIMigrationTool$ chmod +x migrate.py
```

需要重新启动（或启动）延迟的作业守护程序

```shell
canvas@canvas:/var/canvas/vendor$ cd /var/canvas
canvas@canvas:/var/canvas$ script/delayed_job restart
```

安装后，请确保插件在Site Admin->插件-> QTI Converter中处于活动状态（它应该检测到QTIMigrationTool并自动激活）。

## 自动化工作配置

Canvas有一些需要偶尔运行的自动化作业，例如电子邮件报告，统计信息收集和其他一些事情。如果不支持自动作业，您的Canvas安装将无法正常运行，因此我们也需要进行设置。Canvas附带一个守护进程，可以监视和管理任何需要发生的自动化作业。我们需要做的： 安装此守护进程，首先将`/var/canvas/script/canvas_init`中的符号链接添加到`/etc/init.d/canvas_init`，然后将此脚本配置为运行在有效的运行级别，运行以下命令即可

```
canvas@canvas:/var/canvas$ sudo ln -s /var/canvas/script/canvas_init /etc/init.d/canvas_init
canvas@canvas:/var/canvas$ sudo update-rc.d canvas_init defaults
canvas@canvas:/var/canvas$ sudo /etc/init.d/canvas_init start
```

## 启动

重新启动Apache，然后就可以在浏览器打开Canvas了

```
canvas@canvas:/var/canvas$ sudo /etc/init.d/apache2 restart
```

## 自启动配置

```shell
# Canvas
sudo ln -s /var/canvas/script/canvas_init /etc/init.d/canvas_init
# Apache
sudo systemctl enable apache2
# PostgreSQL
sudo systemctl enable postgresql
# Redis
sudo systemctl enable redis-server
```

## 使用说明

### 路径

- 程序路径：
  canvas：/var/canvas/
  apache：/usr/sbin/apache2
  postgresql：/var/lib/postgresql/9.5/main
  redis：/etc/redis/
- 日志路径：
  canvas：/var/canvas/log
  apache：/var/log/apache2/canvas_errors.log，/var/log/apache2/canvas_access.log
  postgresql：/etc/postgresql/9.5/main/pg_log
  redis：/var/log/redis/redis-server.log
- 配置文件路径：
  canvas：/var/canvas/config
  apache：/etc/apache2/sites-enabled/canvas.conf
  postgresql：/etc/postgresql/9.5/main/postgresql.conf
  redis：/etc/redis/redis.conf

### 账号密码

数据库账号密码

- 数据库安装方式：包管理安装

- 账号密码：
  账号：postgres
  密码：P@ssw0rd
  账号：canvas1
  密码：P@ssw0rd

- 密码修改方案

  ```shell
  # 修改 /etc/postgresql/9.5/main/pg_hba.conf 文件
  # 文件原有配置
  host    all             all             127.0.0.1/32            md5
  # 修改为
  host    all             all             127.0.0.1/32            trust
  
  # 重启postgresql服务
  sudo systemctl restart postgresql
  
  # 登录
  psql -h 127.0.0.1 -U postgres
  # 修改密码
  alter user postgres with password '新密码';
  # 退出
  \q
  
  # 此时即可使用新密码进行登录
  ```

后台账号密码

- 登录地址：https://34.92.107.145/

- 账号密码：
  账号：3089476870@qq.com
  密码：websoft9

- 密码修改方案

  ```shell
  # 此方案适用于电子邮件不可用时修改管理员账号密码
  canvas@canvas:/var/canvas$ cd /var/canvas
  # canvas官方提供的脚本，运行后会进到新的命令行交互界面
  canvas@canvas:/var/canvas$ sudo RAILS_ENV=production script/rails console
  
  # 以下是在新交互界面运行需要运行的
  user = User.find(1)  # 管理员账号ID一般为1，如果不确定，可以先运行User.find()
  pseudonym = user.pseudonym
  pseudonym.password = pseudonym.password_confirmation = 'websoft9'  # websoft9即为新密码
  pseudonym.save  # 返回true即为修改成功
  exit  # 退出命令行交互界面
  
  # 此时返回登录界面，即可使用新密码进行登录
  ```

### 版本号

您可以通过如下命令获取主要组件的版本号

```shell
# Canvas
访问 https://github.com/instructure/canvas-lms/releases
# Apache
apache2 -v
# PostgreSQL
psql --version
# Redis
redis-cli -v  # 客户端
redis-server -v  # 服务端
```

### 端口号

| 服务       | 端口    |
| ---------- | ------- |
| Apache     | 80，443 |
| PostgreSQL | 5432    |
| Redis      | 6379    |

## 日志

- 2021-01-15 完成Canvas的搭建
- 2021-01-16 完成Canvas搭建帮助文档的编写