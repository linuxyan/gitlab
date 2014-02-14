## centos6.2安装gitlab6.4环境准备 ##

### python版本2.6 ###
### git版本	1.8.4.1 ###
### ruby版本ruby-2.0.0-p353 ###
### gitlab-shell版本 v1.8.0 ###
### gitlab版本6.4.3 ###

因centos6系列的python版本是2.6的，已经支持，所以不必升级python版本。
在centos5下面需要升级python版本>2.5

安装epel的yum源

    yum -y install http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

安装必要的软件包

    yum -y install libicu-devel patch gcc-c++ readline-devel zlib-devel libffi-devel openssl-devel make autoconf automake libtool bison libxml2-devel libxslt-devel libyaml-devel zlib-devel openssl-devel cpio expat-devel gettext-devel curl-devel perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker

## 安装git ##
因为git需要1.8版本以上，所以需要重新编译安装
移除当前git

    yum remove git

下载1.8.4.1的git并安装

    curl --progress https://git-core.googlecode.com/files/git-1.8.4.1.tar.gz | tar xz
    cd git-1.8.4.1/
    make prefix=/usr/local all
    make prefix=/usr/local install
    ln -fs /usr/local/bin/git* /usr/bin/

## 安装ruby环境 ##

    mkdir /tmp/ruby && cd /tmp/ruby
    curl --progress ftp://ftp.ruby-lang.org/pub/ruby/2.0/ruby-2.0.0-p353.tar.gz | tar xz
    cd ruby-2.0.0-p353/
    ./configure --disable-install-rdoc
    make && make install
    gem install bundler --no-ri --no-rdoc
    ln -s /usr/local/bin/ruby /usr/bin/ruby
    ln -s /usr/local/bin/gem /usr/bin/gem
    ln -s /usr/local/bin/bundle /usr/bin/bundle
    #替换ruby源为淘宝源
    gem source -r https://rubygems.org/
    gem source -a http://ruby.taobao.org/

## 添加git帐号并允许sudo ##

    useradd --comment 'GitLab' git
    echo "git ALL=(ALL)       NOPASSWD: ALL" >>/etc/sudoers

## 安装git-shell ##

    cd /home/git
    sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-shell.git -b v1.8.0
    cd gitlab-shell/
    sudo -u git -H cp config.yml.example config.yml
    vim config.yml
    修改gitlab_url为gitlab的域名
    gitlab_url: "http://localhost/"
    修改为
    gitlab_url: "http://git.linuxyan.com/"
    #安装git-shell
    sudo -u git -H ./bin/install

## 安装mysql以及建立gitlab数据库 ##

    yum install mysql mysql-devel mysql-server -y
    /etc/init.d/mysqld start
    chkconfig mysqld on
    登录mysql创建gitab的帐号和数据库
    mysql> CREATE USER 'gitlab'@'localhost' IDENTIFIED BY 'gitlab';
    mysql> CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
    mysql> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'gitlab'@'localhost';

    测试是否可以用git帐号登录数据库

    sudo -u git -H mysql -u gitlab -p -D gitlabhq_production

## 安装redis ##

    yum -y install redis

    /etc/init.d/redis start

    chkconfig redis on

## 安装gitlab ##

    cd /home/git
    sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 6-4-stable gitlab
    cd /home/git/gitlab
    sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml

    vim config/gitlab.yml
    修改host为刚才git-shell里面设置的域名
    ## GitLab settings
    gitlab:
    ## Web server settings
      host: git.linuxyan.com
      port: 80
      https: false

    修改git的path
    git:
      bin_path: /usr/local/bin/git

    给文件夹添加相应的权限
    chown -R git log/
    chown -R git tmp/
    chmod -R u+rwX  log/
    chmod -R u+rwX  tmp/
    创建必要的文件夹，以及复制配置文件
    sudo -u git -H mkdir /home/git/gitlab-satellites
    sudo -u git -H mkdir tmp/pids/
    sudo -u git -H mkdir tmp/sockets/
    sudo chmod -R u+rwX  tmp/pids/
    sudo chmod -R u+rwX  tmp/sockets/
    sudo -u git -H mkdir public/uploads
    sudo chmod -R u+rwX  public/uploads
    sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb
    sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb
### 设置gitlab的全局帐号 ###
    sudo -u git -H git config --global user.name "GitLab"
    sudo -u git -H git config --global user.email "gitlab@localhost"
    sudo -u git -H git config --global core.autocrlf input

### 设置数据库链接地址和权限 ###
    sudo -u git cp config/database.yml.mysql config/database.yml
    sudo -u git -H vim  config/database.yml
    修改链接数据库信息
    production:
      adapter: mysql2
      encoding: utf8
      reconnect: false
      database: gitlabhq_production
      pool: 10
      username: gitlab
      password: "gitlab"
      # host: localhost
      # socket: /tmp/mysql.sock

### 安装需要ruby的gems ###
    cd /home/git/gitlab
    sudo -u git -H bundle install --deployment --without development test postgres aws
这个安装时间有点长，安装过程如下图：
![请输入图片描述][1]
安装完成后:
![请输入图片描述][2]

### 初始化数据库  ###

    sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production
初始化数据库之后，会告诉你默认的管理员用户和密码:
![请输入图片描述][3]

### 安装启动文件以及日志切割文件 ###

    cp lib/support/init.d/gitlab /etc/init.d/gitlab
    cp lib/support/init.d/gitlab.default.example /etc/default/gitlab
    cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab

### 检测当前环境 ###

    sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production
如下：
![请输入图片描述][4]

### 安装nginx ###

    yum -y install nginx
vim /etc/nginx/nginx.conf

    user              root git;
    worker_processes  2;
    pid        /var/run/nginx.pid;
    
    events {
        worker_connections  1024;
    }
    
    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;
    
        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';
    # GITLAB
    # Maintainer: @randx
    # App Version: 5.0
    
    upstream gitlab {
      server unix:/home/git/gitlab/tmp/sockets/gitlab.socket;
    }
    
    server {
      listen *:80 default_server;         # e.g., listen 192.168.1.1:80; In most cases *:80 is a good idea
      server_name YOUR_SERVER_FQDN;     # e.g., server_name source.example.com;
      server_tokens off;     # don't show the version number, a security best practice
      root /home/git/gitlab/public;
    
      # Set value of client_max_body_size to at least the value of git.max_size in gitlab.yml
      client_max_body_size 5m;
    
      # individual nginx logs for this gitlab vhost
      access_log  /var/log/nginx/gitlab_access.log;
      error_log   /var/log/nginx/gitlab_error.log;
    
      location / {
        # serve static files from defined root folder;.
        # @gitlab is a named location for the upstream fallback, see below
        try_files $uri $uri/index.html $uri.html @gitlab;
      }
    
      # if a file, which is not found in the root folder is requested,
      # then the proxy pass the request to the upsteam (gitlab unicorn)
      location @gitlab {
        proxy_read_timeout 300; # https://github.com/gitlabhq/gitlabhq/issues/694
        proxy_connect_timeout 300; # https://github.com/gitlabhq/gitlabhq/issues/694
        proxy_redirect     off;
    
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   Host              $http_host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    
        proxy_pass http://gitlab;
      }
    }
    
    }
### 更改权限，启动nginx ###

    nginx -t
    chown -R git.git /var/lib/nginx/
    /etc/init.d/nginx start

### 拉取gitlab静态资源文件 ###

    sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production

### 启动gitlab ###

    /etc/init.d/gitlab start

### 检测各个组件是否正常工作 ###

    sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
检测没有错误就表示已经安装好了gitlab，如图：
![请输入图片描述][5]

这个时候就可以用浏览器打开http://git.linuxyan.com
初始管理员帐号和密码为：
admin@local.host
5iveL!fe
登录之后如下：
![请输入图片描述][6]


  [1]: http://segmentfault.com/img/bVbRpU
  [2]: http://segmentfault.com/img/bVbRp6
  [3]: http://segmentfault.com/img/bVbRp8
  [4]: http://segmentfault.com/img/bVbRp9
  [5]: http://segmentfault.com/img/bVbRqy
  [6]: http://segmentfault.com/img/bVbRqF
