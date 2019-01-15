* 本文简要介绍如何利用配置文件搭建过个二级域名的方法。

1. 首先说明服务器配置，我买的是阿里云的ECS，1g1核的，只是用来测试用用，系统是ubuntu14.0。环境是LAMP，即apache2+mysql+php5.至于如何在ubuntu中搭建LAMP环境在网站的其他文章中会有详细介绍，这里就不赘述了。
2. 其次因为我们需要的是公网访问，所以我们需要一个已经成功备案的域名，这里我以aaa.cn举例说明。
3. 在阿里云的控制台中查看到我们ECS的公网IP为1.1.1.1（本处为示例，详细IP请自行查看）。然后我们想要一个什么样子的二级域名呢，比如我们可以申请一个blog的二级域名，我们访问的时候输入的是blog.aaa.cn。
4. 我们在阿里云的控制台中找到域名管理，然后选择我们的域名aaa.cn后面的“解析”字样。在新打开的页面中我们可以看到一个提供域名解析的界面，选择解析类型为A类解析，然后输入blog（我们想要的二级域名的头），然后在地址里面输入我们刚才查看到的ECS的公网ip，然后点击确认，之后我们的解析就会生效了。
5. 现在我们可以进入服务器进行配置文件的配置了。
6. 我用的是ubuntu的系统，并且阿里云的一个好处是可以多终端访问，我们在终端控制面板中输入ssh  用户名@ip地址，点击回车，输入你ECS服务器的密码（如果不记得密码了可以在阿里云的控制台ECS中修改）。
7. 进入服务器后我们安装好LAMP环境后进入~/etc/apache2/site-available/的目录下，我们会发现一个以.conf结束的文件，例如我想要建立的是一个博客的二级域名，我二级域名是blog.aaa.cn，因此我可以把这个文件命名为blog.aaa.cn.conf（请注意这个也要有.conf结尾，以前可以不用，但是因为apache更新现在必须使用这个结尾了，不然不能加载成功），然后用vim命令打开这个文件，在这个文件里面输入内容：
    ```
    <VirtualHost *:80>
        ServerAdmin admin@admin.com  /*如果有问题可以联系的管理员邮箱*/
        ServerName blog.aaa.cn             /*你要访问的详细二级域名*/
        DocumentRoot /var/www/html/blog/         /*你二级域名的文件地址*/
        ErrorLog ${APACHE_LOG_DIR}/error.log             /*报错日志*/
        CustomLog ${APACHE_LOG_DIR}/access.log combined      /*记录日志*/
    </VirtualHost>
    ```
    
8. 完成后保存退出，切换到上一级目录：`cd ../site-enable/`
9. 然后输入：`ln -s ../sites-available/目标文件`
10. 然后在切换回去刚才的site-available目录下输入：`a2ensite blog.aaa.cn.conf`
11. 如果提示已经激活过了请输入一下代码：
    ```
        a2dissite blog.aaa.cn.conf
        service apache2 reload
        a2ensite blog.aaa.cn.conf
        service apache2 reload
    ```

* 这样就大功告成了，现在你就可以直接使用blog.aaa.cn来访问你的网站啦~\(≧▽≦)/~啦啦啦




