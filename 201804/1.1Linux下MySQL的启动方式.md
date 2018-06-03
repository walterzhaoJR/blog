# Linux下MySQL的启动方式

* 在Linux系统下，MySQL服务器通常有四种启动方式：**mysqld**守护进程启动，**mysqld_safe**启动，**mysql.server**启动，**mysqld_multi**多实例启动。

## 1.mysqld

* 启动mysql服务器:./mysqld --defaults-file=/etc/my.cnf --user=root
* 一般的，我们通过这种方式手动的调用mysqld，如果不是出去调试的目的，我们一般都不这样做。这种方式如果启动失败的话，错误信息只会从终端输出，而不是记录在错误日志文件中，这样，如果mysql崩溃的话我们也不知道原因，所以这种启动方式一般不用在生产环境中，而一般在调试（debug）系统的时候用到。 

## 2. mysqld_safe

* 启动mysql服务器:./mysqld_safe --defaults-file=/etc/my.cnf --user=root &
* **mysqld_safe**是一个启动脚本，该脚本会调用mysqld启动，如果启动出错，会将错误信息记录到错误日志中，mysqld_safe启动mysqld和monitor mysqld两个进程，这样如果出现mysqld进程异常终止的情况，mysqld_safe会重启mysqld进程。 



## 3. mysql.server

```linux
cp -v /usr/local/mysql/support-files/mysql.server /etc/init.d/

chkconfig --add mysql.server

启动mysql服务器:service mysql.server {start|stop|restart|reload|force-reload|status}
```

```linux
mysql.server同样是一个启动脚本，调用mysqld_safe脚本。它的执行文件在$MYSQL_BASE/share/mysql/mysql.server 和 support-files/mysql.server。 
主要用于系统的启动和关闭配置
```



## 4. mysqld_multi

* 多实例启动
* 配置不同的端口和配置文件