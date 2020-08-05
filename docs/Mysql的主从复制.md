```shell
# Master服务器：

1: vim /ect/my.cnf

    #添加如下内容：
       log-bin=master-a-bin       #日志文件名称
       binlog-format=ROW          #二进制日志格式，有ROW、statement和mixed三种
       server-id=1                #要求各个服务器的这个ID不许不一样
       binlog-do-db=xiaosa_mycat  #同步的数据库的名字

#2: 授权从服务器登录主数据库的账号
    grant replication slave  on *.* to ‘root’@'主服务器IP' identified by '数据库密码';
   
#3: 刷新授权
   flush privileges;

  
Slave服务器：

1: vim /ect/my.cnf

    #添加如下内容：
        log-bin=master-a-bin       #日志文件名称
        binlog-format=ROW          #二进制日志格式，有ROW、statement和mixed三种
        server-id=2                #要求各个服务器的这个ID不许不一样
        #log-slave-updates=true    #如果从服务器也要开启binlog日志就打开


#配置完了之后，重启Master服务器，
service mysqld restart

#查看Master服务器，登陆mysql
show master status;

#重启Slave服务器
service mysqld restart

#Slave服务器，登陆mysql
change master to master_host='Master的IP’, master_user='Master的账户’,master_password=‘Master的密码’,master_port=Master的端口号,master_log_file=‘用show master status 命令查看的Master的File的信息，例如：master-a-bin.000001’, master_log_pos=用show master status 命令查看的Master的Position的信息，例如：154;

#开启Slave
start slave;

#查看Slave服务器
show slave status \G
```