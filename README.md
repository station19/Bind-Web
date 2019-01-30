

<h1 align = "center">Bind-DLZ + Django   + Mysql  DNS管理平台 </h1>

系统环境:CentOS 6.5 X64

##### Mysql安装 : https://github.com/station19/MyDockerOS/blob/master/Config/DNS/%E4%BC%81%E4%B8%9A%E5%86%85%E7%BD%91DNS-Web%E7%AE%A1%E7%90%86%E5%B9%B3%E5%8F%B0.md

软件版本: 

      bind-9.9.5.tar.gz  
      mysql-5.6.16.tar.gz
	  Python 2.7.6 
	  Django 1.11.x
描述： 
数据库安装就不在絮叨，了解运维的同学都应该知道

<h2 align = "center">一．源码安装配置Bind: </h2>

1.源码编译安装

	 tar -zxvf  bind-9.9.5.tar.gz           #解压压缩包
	 cd bind-9.9.5
	 ./configure --prefix=/usr/local/bind/  \
	 --enable-threads=no \
	 --enable-newstats   \
	 --with-dlz-mysql    \
	 --disable-openssl-version-check
	 
     #官网说明强调编译关闭多线程，即--enable-threads=no
	 
     	  make
	  make install           #源码编译安装完成
	  # 如果提示 not found  libmysqlclient,执行下面命令
 	  # ln -s /usr/lib64/mysql/libmysqlclient.so /usr/lib/libmysqlclient.so

 
2.环境变量配置

	cat >>  /etc/profile  <<EOF 
	PATH=$PATH:/usr/local/bind/bin:/usr/local/bind/sbin
	export  PATH
	EOF

	 source  /etc/profile  #重新加载一下环境变量
	 named -v           #如下图，说明环境变量正常


	 
![](https://github.com/1032231418/doc/blob/master/images/1.png?raw=true)


3.用户添加授权目录

	 useradd  -s  /sbin/nologin  named
	 chown  -R named:named /usr/local/bind/





4.配置Bind
注意：  bind 的数据库即是 管理平台使用的数据库，这里配置的库名和，后面管理平台的数据库名一样

 vi /usr/local/bind/etc/named.conf
       
		options {
				directory       "/usr/local/bind/";
				version         "bind-9.9.5";
				listen-on port 53 { any; };
				allow-query-cache { any; };
				listen-on-v6 port 53 { any; };
				allow-query     { any; };
				recursion yes;    
				dnssec-enable yes;
				dnssec-validation yes;
				dnssec-lookaside auto;

		};
		 
		 
		key "rndc-key" {
				algorithm hmac-md5;
				secret "C4Fg6OGjJipHKfgUWcAh+g==";

		};
		 
		controls {
				inet 127.0.0.1 port 953
						allow { 127.0.0.1; } keys { "rndc-key"; };
		};
		 
		 
		view "ours_domain" {
				match-clients           {any; };
				allow-query-cache           {any; };
				allow-recursion          {any; };
				allow-transfer          {any; };
		 
				dlz "Mysql zone" {
						database        "mysql
						{host=localhost dbname=dns ssl=false port=3306 user=root pass=123456}
						{select zone from dns_records where zone='$zone$'}
						{select ttl, type, mx_priority, case when lower(type)='txt' then concat('\"', data, '\"') when lower(type) = 'soa' then concat_ws(' ', data, resp_person, serial, refresh, retry, expire, minimum) else data end from dns_records where zone = '$zone$' and host = '$record$'}"; 
				};
				zone "."  IN {
					type hint;
					file "/usr/local/bind/etc/named.ca";
				};
		 
		};

保存退出

5.生成 name.ca文件

	cd /usr/local/bind/etc/
	dig -t NS .  >named.ca





<h2 align = "center">二．配置Bind-Web 管理平台 </h2>

1.克隆管理平台

	#git  clone  https://github.com/1032231418/Bind-Web.git  #git  克隆下来
	
	git clone https://github.com/station19/Bind-Web.git
	cd Bind-Web
	find . -name "*.py" -exec chmod 755 {} \;

2.安装Django框架
```
	yum install python-devel
	wget https://bootstrap.pypa.io/get-pip.py
	python get-pip.py
	pip install -r  requirement.txt
```

3.数据库配置:   

        1.) CREATE DATABASE  dns   CHARACTER SET utf8 COLLATE utf8_general_ci;  #创建数据库
		
        2.) 配置文件devops/settings 里连接数据库
		
				DATABASES = {
					'default': {
						'ENGINE': 'django.db.backends.mysql',
						'NAME':'dns',
						'USER': 'root',
						'PASSWORD': '123456',
						'HOST': 'localhost',
						'PORT':'3306',
					}
				}
				
        3.) 表结构刷到数据库

				 python  manage.py makemigrations
				 
	
![image](https://github.com/1032231418/PYVM/blob/master/bind-web-images/makemigrations.png)	
				 
				 python  manage.py migrate		
				 
![image](https://github.com/1032231418/PYVM/blob/master/bind-web-images/migrate.png)				 			 
				 
        4.) 创建管理用户
		
				 python manage.py  createsuperuser  
				 
![image](https://github.com/1032231418/PYVM/blob/master/bind-web-images/createuser.png)				 
				 

        5.) 运行项目
		
				 python manage.py  runserver 0.0.0.0:80

	5.1) 或者用supervisor接管进程 运行项目
```
	pip install supervisor
	mkdir /etc/supervisor/
	echo_supervisord_conf > /etc/supervisor/supervisord.conf

cat>>/etc/supervisor/supervisord.conf<<EOF
[program:dns-manage]
command=python manage.py  runserver 0.0.0.0:80
directory=/root/Bind-Web-Django/devops
user=root
autorestart=true
;startsecs = 1
stdout_logfile=/var/log/dns-manage.log


[program:bind]
command=/usr/local/bind/sbin/named
user=root
autorestart=true
;startsecs = 1
stdout_logfile=/var/log/bind.log
EOF


supervisord -c /etc/supervisor/supervisord.conf
```


http://ip:80  访问WEB 界面 登录账户就是创建的管理用户


![image](https://github.com/1032231418/PYVM/blob/master/bind-web-images/login.png)	


![image](https://github.com/1032231418/PYVM/blob/master/bind-web-images/darshboard.png)	

			 
![image](https://github.com/1032231418/PYVM/blob/master/bind-web-images/userlist.png)
	
				 
![image](https://github.com/1032231418/PYVM/blob/master/bind-web-images/namelist.png)	


<h2 align = "center">三．启动Bind服务并设置开机启动脚本 </h2>

1.启动  Bind 服务并设置开机启动脚本

    	/usr/local/bind/sbin/named

2.监控系统日志：

	 tail -f /var/log/messages
	 
3.如下，说明服务启动正常

![](https://github.com/1032231418/doc/blob/master/images/3.png?raw=true)

	测试bind连接数据库是否正常:

![](https://github.com/1032231418/doc/blob/master/images/4.png?raw=true)


4.设置 Bind  开机启动脚本

	bind 本文档会附带，传到服务器  /etc/init.d/ 目录
	chmod  755 /etc/init.d/bind 
	mkdir  /var/run/named/ && chown  named:named -R /var/run/named 
	杀掉 named  服务，改用脚本启动

	pkill  named
	/etc/init.d/bind  start            #监控日志，查看启动状态
	chkconfig  --add bind            #加入开机启动
 tail -f /var/log/messages

![](https://github.com/1032231418/doc/blob/master/images/5.png?raw=true)
