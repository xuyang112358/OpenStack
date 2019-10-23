# 阿里云服务器中安装配置MYSQL数据库完整教程
### 第一步： 确保服务器系统处于最新状态
 yum -y update<br>
### 第二步： 确保服务器系统处于最新状态
首先检查是否已经安装，如果已经安装先删除以前版本，以免安装不成功<br>
 rpm -qa | grep mysql<br>
或<br>
 yum list installed | grep mysql<br>
如果安装了的话，就使用下面这条命令删除原先的mysql,举例如下：<br>
 rpm -e  --nodeps        mysql-libs-5.1.73-5.e16_6.i686 <br>
### 第三步： 下载MySql安装包
 rpm -ivh http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm<br>
或<br>
 rpm -ivh http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm<br>
### 第四步： 安装MySql
 yum install -y mysql-server<br>
或<br>
 yum install mysql-community-server<br>
### 第五步： 设置开机启动mysql
 systemctl enable mysqld.service<br>
### 第六步： 检查是否已经安装了开机自动启动
 systemctl list-unit-files | grep mysqld<br>
 如果显示以下内容说明已经完成自动启动安装<br>
>> mysqld.service enabled<br>
### 第七步： 设置开启服务
 systemctl start mysqld.service
### 第八步： 查看MySql默认密码
 grep 'temporary password' /var/log/mysqld.log   
### 第九步： 登陆MySql，输入用户名和密码
mysql -uroot -p       //密码也就是第九步里面查看到的默认密码
### 第十步： 修改当前用户密码 注意看下面的报错
mysql>SET PASSWORD = PASSWORD('alliance');  //但是这样会报错的，具体错误看下面
注：直接复制粘贴上边的命令，会报错，错误如下<br>
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements<br>
解决方法：设置密码的验证强度等级，设置 validate_password_policy 的全局参数为 LOW 即可，<br>
输入设值语句 “ set global validate_password_policy=LOW; ” 进行设值<br>
重复密码设置
### 第十一步： 开启远程登录，授权root远程登录
（解释：不要以为阿里云服务器可以远程登录root用户，就以为我们也可以以mysql的root用户身份远程登录mysql数据库）<br>
mysql>GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'alliance' WITH GRANT OPTION; <br> 
//这里的alliance要换成你自己mysql数据库的密码
### 第十二步： 命令立即执行生效
mysql>flush privileges;<br>
//// 这一步一定要做，不然无法成功！ 这句表示从mysql数据库的grant表中重新加载权限数， 因为MySQL把权限都放在了cache中，所以在做完更改后需要重新加载。<br>
Mysql是为了安全考虑，初始的时候并没有开启Root用户<br>
（解释：mysql的root用户和我们云服务器的root用户是两个不同的，分开的）的远程访问权限，Root（解释：这里是指mysql的root用户，而不是云服务器的root用户）<br>
只能本地localhost，127.0.0.1访问，但是我们操作起来实在是不方便，下面我们就使用Xshell连接Linux服务器操作Mysql给Root用户（解释：这里是指mysql的root用户，<br>
而不是云服务器的root用户）添加远程访问权限。（**解释：**上面的第十二步，就是授权其他工具可以远程以mysql的root用户的身份登录mysql<br>
（解释：这里是指mysql的root用户，而不是云服务器的root用户））<br>
