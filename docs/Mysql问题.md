```shell
安装
sudo yum install mysql-community-server

启动
service mysqld start

查询临时密码，用临时密码登里。
grep 'temporary password' /var/log/mysqld.log

修改密码：
set password for root@localhost=password('wangsiwei0914');

可能会报错：
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

原因：密码安全问题，

解决，并再次设置密码。
set global validate_password_policy=0;

授予root用户远程访问：
grant all privileges on *.* to 'root' @'%' identified by ‘wangsiwei0914' with grant option;

刷新权限，使设置生效， OK。
flush privileges;

如果远程报连不上，可能是防火墙的问题
sudo firewall-cmd --zone=public --permanent --add-service=mysql sudo systemctl restart firewalld
```