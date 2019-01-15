### 开始粘贴
```
sudo apt-get update 
sudo apt-get install python-pip python-dev nginx
```

粘粘粘
```
sudo pip install virtualenv
```
好了，粘到这了我要说一下，virtualenv这个python中的神器不需要我再细说了，下面我要说的是我们要创建一个目录也是一样，粘粘粘，这个路径你也可以自己设置
```
mkdir /home/project
cd /home/project
```
我们现在在当前的目录下/home/project/继续粘贴
```
virtualenv venv
```
粘贴完成后你会发现在/home/project/下会有一个venv的文件夹，我们继续
```
cd venv
```
现在路径就是/home/project/venv 这个路径下会有几个文件夹，管它呢？我就要结果，好的
```
source bin/activate
```
执行完后你会发现我们已经在虚拟环境中了
```
(venv)user@host:~/project类似这样的前面有个括号里面是venv
```
### ----------------------------------------这是分割线---------------------------------------

我们已经在虚拟环境中了，下面不要退出我们继续在虚拟环境中搞事情。
```
pip install uwsgi flask
```
在虚拟环境中继续安装.......

正常情况下，此时别的教程就开始了，我们要新建一个test.py的项目类似这样的
```
from flask import Flask
app = Flask(__name__)

.....然后让你请求返回hello world

if __name__ == "__main__":
    app(host='0.0.0.0')
```
然后让你python test.py ,又让你在浏览器输入，滚滚滚滚，md老子哪里的浏览器，都是大黑屏。

上传项目
我们先把我们的项目传到服务器上面去
```
sudo scp -r /你本地电脑路径/appservice/ root@ip:/home/project
```
会让你输入密码，输入就好了，注意：sudo scp -r /你本地电脑路径/appservice/ root@ip:/home/project 是在你的本地电脑上输入的！

现在你的项目已经来到了服务器上面，

创建uWSGI入口
比较麻烦的正题开始了。创建wsgi入口：这个不要慌张因为刚刚你把项目传上来的时候已经创建了一个appservice的文件夹，下面继续粘贴命令
```
nano /home/project/appservice/wsgi.py
```
让你创建一个wsgi.py内容是

from 你项目入口里面的application import application
```
if __name__ == "__main__":
    application.run()
```
然后我们保存一下！记得保存！！

保存成功后我们继续，先把虚拟环境关掉
```
deactivate
```
现在开始很重要了！虽然可以直接粘贴，但是最好要理解的粘贴，别因为你的文件夹名称不同导致失败。

创建uWSGI配置文件
```
nano /home/project/appservice/myproject.ini
```
粘贴内容
```
[uwsgi]
module = wsgi

master = true
processes = 5

socket = myproject.sock
chmod-socket = 660
vacuum = true

die-on-term = true
```
完成后保存，友情提示 输入 :wq

创建一个Upstart脚本
就是粘贴
```
sudo nano /etc/init/myproject.conf
```
粘贴
```
description "uWSGI server instance configured to serve myproject"

start on runlevel [2345]
stop on runlevel [!2345]

setuid root
setgid www-data

env PATH=/home/project/venv/bin           
chdir /home/myproject/appservice
exec uwsgi --ini myproject.ini
```
env:就是最开始我们创建的虚拟环境路径

chdir：我们的项目路径

这两个根据实际情况改一下子就可以了

完成后保存并关闭文件。 :wq

您可以通过键入以下内容立即开始进程：
```
sudo start myproject
```
此时应该显示的运行成功，之前我运行一次不成功的情况是，setuid那个root没改

将Nginx配置为代理请求
继续粘贴
```
sudo nano /etc/nginx/sites-available/myproject
```
粘
```
server {
    listen 80;
    server_name 你的服务器对外ip;
    access_log /home/project/log/access.log;
    error_log /home/project/log/error.log;
    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/project/appservice/myproject.sock;
    }
}
```
粘贴到这里有的朋友会发现myproject.sock这个东西出现了很多次阿，哪里来的啊，不要怕，此时这个myproject.sock正安静的躺在你的appservice里。

保存 :wq

注意/home/project/log/文件夹得先创建出来



要启用我们刚刚创建的Nginx服务器块配置，请将该文件链接到sites-enabled目录：
```
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```
我们可以键入以下内容来测试语法错误：
```
sudo nginx -t
```
balabalabalabtla%^&$%$* 出现success ok 卧槽，恭喜你。

如果这返回没有指示任何问题，我们可以重新启动Nginx进程来阅读我们的新配置：
```
sudo service nginx restart
```
您现在应该可以在Web浏览器中访问服务器的域名或IP地址

md等待为什么我的不好用？费了一大顿劲为什么还是链接不上。

这里面是因为你的阿里云80端口没给你开，你去添加一个80端口就ok了！！
