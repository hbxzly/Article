# 前言
>因为某些综合原因，导致这次购买的服务器只能是 windows 系统的，网上很多windows 系统下使用 IIS 配置WEB服务器，但是对于新手来说都麻烦了些，下面介绍一种最简单的配置远程 WEB 服务器的方式

## 准备

1. XAMP 软件
2. 够了

## 步骤
1. 在你的服务器上安装 xamp 软件，安装完成后开启 Apache 服务和 mysql 服务（做 java 的同学也不用担心，这个软件同样支持 tomcat 一键开启），打开 localhost 查看安装成功没。
2. 安装好后访问 localhost/phpmyadmin，打开数据库，找到菜单中的 user_account，打开后找到用户名是 root，地址是 localhost 的哪一个，点编辑。
3. 选择 change password 修改密码，输入你想要的数据库登录密码即可，记好。
4. 打开 xamp 软件，点击 apache 服务的后 config 按钮选择config.inc.php文件，打开，按 ctrl+f 查找“password”，找到这句话“
$cfg['Servers'][$i]['password'] = '1234567'”，把里面的1234567换成你自己刚才修改的密码即可（这里是示范，实际你的文件中这个地方是空白的，没有这串数字）。
5. 打开你的域名管理界面，解析你的域名地址 A 类到服务器的 ip 上即可。
6. 然后你就可以使用域名直接进行访问了。


# 多站点配置：

1. 因为是 xampp 安装的，只需要打开两个文件即可：httpd-vhost.conf 和 httpd.conf。
2. 首先在 httpd.conf 文件中打开以下几个功能块（取消前面的#等）：

		>打开xampp\apache\conf\httpd.conf文件，搜索 “Include conf/extra/httpd-vhosts.conf”，确保前面没有 # 注释符，也就是确保引入了 vhosts 虚拟主机配置文件。

3. 开启了httpd-vhosts.conf，默认的httpd.conf默认配置失效（确保 httpd-vhosts.conf 文件里也开启了虚拟主机配置，见第3条），访问此IP的域名将全部指向 vhosts.conf 中的第一个虚拟主机。（注意是第一个，详见第4）

4. 在虚拟主机设置文件xampp\apache\conf\extra\httpd-vhosts.conf里设置。
5. 取消 NameVirtualHost *:80 前面的 ##，这样就启用了 vhosts.conf ，默认的httpd.conf默认配置失效。虚拟主机配置将只设置在 httpd-vhosts.conf 里。

		   <VirtualHost *:80>
			    ServerAdmin admin@admin.admin
			    DocumentRoot "C:/xampp/htdocs/admin"
			    ServerName admin.admin
			    ErrorLog "logs/admin.admin-error.log"
			    CustomLog "logs/admin.admin-access.log" common
			</VirtualHost>

6. 重启服务.
7. 在浏览器打开 http://admin.admin 即可访问
