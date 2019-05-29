参考的博客：https://cubercsl.cn/notes/DOMjudge-Note.html

### DOMserver 部分
##### 安装依赖包和功能
```
sudo apt install gcc g++ make zip unzip mariadb-server \
        apache2 php php-cli libapache2-mod-php php-zip \
        php-gd php-curl php-mysql php-json php-xml php-intl php-mbstring \
        acl bsdmainutils ntp phpmyadmin python-pygments \
        libcgroup-dev linuxdoc-tools linuxdoc-tools-text \
        groff texlive-latex-recommended texlive-latex-extra \
        texlive-fonts-recommended texlive-lang-european
```
##### 安装时选择 apache2
```
sudo apt install libcurl4-gnutls-dev libjsoncpp-dev libmagic-dev

sudo phpenmod json
```
##### 编译 DOMjudge
```
wget https://www.domjudge.org/releases/domjudge-7.0.0.tar.gz

tar -zxvf domjudge-7.0.0.tar.gz

cd domjudge-7.0.0

./configure --prefix=$HOME/domjudge --with-baseurl=127.0.0.1
make domserver && sudo make install-domserver
make judgehost && sudo make install-judgehost
make docs && sudo make install-docs
```
##### 配置数据库
```
cd ~/domjudge/domserver
sudo bin/dj_setup_database -u root install
```
##### 配置 Web 服务器
```
cd ~/domjudge/domserver
sudo ln -s /home/jscpc/domjudge/domserver/etc/apache.conf /etc/apache2/conf-available/domjudge.conf
sudo a2enmod rewrite
sudo a2enconf domjudge
sudo systemctl reload apache2
```
到这里，就可以访问http://127.0.0.1/domjudge，并使用admin作为用户名和密码登陆后台了。

密码见/home/jscpc/domjudge/domserver/etc/initial_admin_password.secret

##### 配置 MySQL

编辑/etc/mysql/my.cnf，追加内容：
```
cd /etc/mysql

sudo vim my.cnf
```
之后重启：
```
[mysqld]
max_connections = 1000
max_allowed_packet = 500MB
innodb_log_file_size = 5000MB

sudo systemctl restart mysql
```
其中 max_allowed_packet 数值改成两倍于题目测试数据文件的大小，  
innodb_log_file_size 数值改成十倍于题目测试数据文件的大小。
##### 配置 PHP
编辑 ~/domjudge/domserver/etc/apache.conf,
```
cd ~/domjudge/domserver/etc

sudo vim apache.conf
```
取消以下几行内容前的注释：
```
<IfModule mod_php7.c>
php_value max_file_uploads      1000
php_value upload_max_filesize   500M
php_value post_max_size         500M
php_value memory_limit          512M
</IfModule>
```
编辑 /etc/php/7.0/apache2/php.ini，搜索 date.timezone 关键字，取消其行前注释，并将其值设为 Asia/Shanghai。
```
sudo systemctl restart apache2
```
##### 配置 Apache
编辑 /etc/apache2/apache2.conf，
```
cd /etc/apache2/

sudo vim apache2.conf
```
搜索 KeepAlive 关键字，将其值设为 Off，并在其后新增一行内容：
```
MaxClients 1000
```
重新启动
```
sudo systemctl restart apache2
```
##### 配置 DOMserver
访问 http://127.0.0.1/domjudge (或者外网IP/域名，这是指服务器的IP地址)，使用用户名admin密码***登陆。
//密码见/home/jscpc/domjudge/domserver/etc/initial_admin_password.secret

##### 添加队伍账号：
https://cjc7373.github.io/2018/12/19/Domjudge-config/

    直接用sql语句导入
    或者通过http://127.0.0.1/domjudge 通过网页导入teams.tsv和accounts.tsv（先导入team.tsv）
###### 学校的校徽

    可以放在~/domjudge/domserver/webapp/web/images/affiliations文件夹下。
    图片大小需要自行调整，文件名为<ExternalID>.png。
    

### Judgehost 部分
#### 准备工作
###### 安装依赖包和功能
    
```
sudo apt install make sudo debootstrap libcgroup-dev \
        php-cli php-curl php-json php-xml php-zip procps \
        gcc g++ openjdk-8-jre-headless \
        openjdk-8-jdk ghc fp-compiler \
        libcurl4-gnutls-dev libjsoncpp-dev libmagic-dev \
        supervisor
```
###### 编译 DOMjudge

```
#这里只针对单独配置 Judgehost 的机器，如果配置在同一台机器上不需要这些操作。
cd Downloads
wget https://www.domjudge.org/releases/domjudge-7.0.0.tar.gz

tar -zxvf domjudge-7.0.0.tar.gz

cd domjudge-7.0.0

./configure --prefix=$HOME/domjudge --with-baseurl=127.0.0.1 
make judgehost && sudo make install-judgehost
```
#### 配置 Judgehost
###### 添加用户
```
useradd -d /nonexistent -U -M -s /bin/false domjudge-run 
# 单核只需要第一条命令即可
# 如果 judgehost 拥有多个 CPU 核心，你可以添加额外的用户来支持绑定
# 不同的 judgehost 进程到不同的 CPU 核心上，如下：
useradd -d /nonexistent -U -M -s /bin/false domjudge-run-0
useradd -d /nonexistent -U -M -s /bin/false domjudge-run-1
useradd -d /nonexistent -U -M -s /bin/false domjudge-run-2
useradd -d /nonexistent -U -M -s /bin/false domjudge-run-3
# ... 如果有更多的 CPU 核心，请自行添加更多的用户
```
###### 配置 sudoers

    将 $HOME/domjudge/judgehost/etc/sudoers-domjudge 复制到 /etc/sudoers.d/ 目录下。
    
```
sudo cp $HOME/domjudge/judgehost/etc/sudoers-domjudge /etc/sudoers.d/
```
###### 修改 rest 密码

    使用 vim 等文本编辑器  编辑 ~/domjudge/judgehost/etc/restapi.secret 这个文件。
    把 DOMserver 中相应文件夹下的对应文件抄一下。 

    这是为了能够连接上domserver,能够接收domserver分配的任务？

###### 构建 chroot 环境

    使用 vim 等文本编辑器编辑 ~/domjudge/judgehost/bin/dj_make_chroot 脚本，
    将 ubuntu 镜像改为国内源。

    在第172行，换成清华源吧，校园网阿里源好像用不了，这一步需要时间下载软件包
```

# Ubuntu mirror, modify to match closest mirror
[ -z "$DEBMIRROR" ] && DEBMIRROR="http://mirrors.aliyun.com/ubuntu/"

```
###### 设置 cgroup
```
# 使用 vim 等文本编辑器编辑 /etc/default/grub 这个文件，对其中的这一行做如下修改：
GRUB_CMDLINE_LINUX_DEFAULT="quiet cgroup_enable=memory swapaccount=1"
# 然后执行
update-grub
# 之后重启计算机
#使用下面的命令查看是否添加成功
cat /proc/cmdline
```
#### 配置 supervisor(未实践过，不清楚流程)

    对于单核来讲，直接运行 ~/domjudge/judgehost/bin/judgedaemon即可。使用./ 执行脚本？。
    每次重启都需要执行下面脚本？
        每次开机的时候记得执行
        ~/domjudge/judgehost/bin/create_cgroup。
        否则判题的时候会因为权限不够 CE 。

    下面介绍配置多核的方法：
    使用 vim 等文本编辑器在 /etc/supervisor/conf.d 下新建一个文本文件叫做judgehost.conf，
    写入下列内容：
    
```
[program:judgehost]
command=/home/yaun/domjudge/judgehost/bin/judgedaemon -n %(process_num)d
autorestart=true
process_name=%(program_name)s_%(process_num)02d
stdout_logfile=/var/log/judgehost/judgehost.log
stderr_logfile=/var/log/judgehost/judgehost.err.log
numprocs=4
numprocs_start=0
# 上面的 numprocs 应该根据实际的 CPU 核心数。
# 根据实际情况把 ubuntu 改为对应用户名。
# 根据实际情况配置 stdout_logfile 和 stderr_logfile。这个应该是日志文件
```

    最后，使用如下命令重新加载 supervisor ：
        sudo systemctl restart supervisord.service
    启动 judgehost
        sudo supervisorctl start judgehost
    查看 judgehost 进程的运行情况：
        sudo supervisorctl status
        
#### 到此配置就全部完成了

    具体配置有没有问题，可以在 管理员界面 的一项***中查看，忘记了（应该是configure ***）

#### 打印服务

    https://github.com/wsc500/codePrint 
    
    这是一个独立于domjudge的系统，当初没有搞定domjudge自带的打印功能，
    让系统指定远程打印机打印，也不会配置。

#### 滚榜

    https://github.com/hex539/scoreboard
    
    一点都没接触。。。
