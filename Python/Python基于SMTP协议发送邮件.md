```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
"""
    @Time    : 2018/5/31
    @Author  : LXW
    @Site    : 
    @File    : sendEmail.py
    @Software: PyCharm
    @Description: 使用SMTP协议发送邮件，支持同时发送给多个地址，支持同时发送文本信息、超文本信息和多附件
"""
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.header import Header
import os

class properties():
    # 设置服务器, "smtp.xx.com"
    mail_host = "smtp.qq.com"
    # 用户名
    mail_user = "@qq.com"
    # 口令
    mail_pass = ""
    # smtp服务器端口，每个服务商提供的邮件服务端口可能不一致，465是腾讯的端口
    mail_port = 465
    # 发送邮件的地址
    sender = "@vip.qq.com"
    # 接收邮件，可设置为你的QQ邮箱或者其他邮箱，list类型，可同时填写多个地址并以,分割
    receivers = "@qq.com","@qq.com"
    # 邮件发送的内容
    messageText = "测试使用\n"
    # 邮件发送的超文本内容
    messageHTML = """
                    <!DOCTYPE html>
                    <html lang="en">
                    <head>
                        <meta charset="UTF-8">
                        <title>test</title>
                    </head>
                    <body>
                        <img src="http://a.hiphotos.baidu.com/image/pic/item/730e0cf3d7ca7bcb6a172486b2096b63f624a82f.jpg" alt="test" width="200px" height="200px">
                    </body>
                    </html>
                """
    # 发送邮件方的别名展示（类似昵称）,为空则显示发件方地址
    messageFromHeader = ""
    # 接收邮件方的展示信息
    messageToHeader = "test python"
    # 邮件主题
    messageSubject = "ceshiceshi123"
    # 需要发送的附件的详细地址，支持多附件发送，附件之间以，分割
    filePaths = '1.txt','2.txt','3.txt'


def sendMail():
    # 下面所有参数均可通过配置文件配置获取
    """
        :param mail_host: 设置服务器,"smtp.xx.com"
        :param mail_user: 用户名
        :param mail_pass: 口令
        :param sender: 发送邮件的地址
        :param receivers: 接收邮件，可设置为你的QQ邮箱或者其他邮箱
        :param messageText: 邮件发送的文本内容
        :param messageHTML: 邮件发送的超文本内容
        :param messageFromHeader: 发送邮件方的别名展示（类似昵称）
        :param messageToHeader: 接收邮件方的展示信息
        :param messageSubject: 邮件主题
        :param filePath: 附件详细地址
        :return:
    """
    # 需要获取的参数列
    mail_host = properties.mail_host
    mail_user = properties.mail_user
    mail_pass = properties.mail_pass
    mail_port = properties.mail_port
    sender = properties.sender
    receivers = properties.receivers
    messageText = properties.messageText
    messageHTML = properties.messageHTML
    messageFromHeader = properties.messageFromHeader
    # 如果发件人昵称未填写则直接使用发件人地址作为名称
    if messageFromHeader == "":
        messageFromHeader = sender
    messageToHeader = properties.messageToHeader
    messageSubject = properties.messageSubject
    filePaths = properties.filePaths

    # 邮件类型为"multipart/alternative"的邮件包括纯文本正文（text / plain）和超文本正文（text / html）。
    # 邮件类型为"multipart/related"的邮件正文中包括图片，声音等内嵌资源。
    # 邮件类型为"multipart/mixed"的邮件包含附件。向上兼容，如果一个邮件有纯文本正文，超文本正文，内嵌资源，附件，则选择mixed类型。
    message = MIMEMultipart('mixed')

    # 邮件显示信息内容
    # 发送邮件方的头部展示信息
    message['From'] = Header(messageFromHeader, 'utf-8')
    # 接收邮件方的展示信息
    message['To'] = Header(messageToHeader, 'utf-8')
    # 邮件主题
    message['Subject'] = Header(messageSubject, 'utf-8')

    try:
        # 发送邮件附件，支持多附件发送
        for filePath in filePaths:
            messageFile = open(filePath, 'rb').read()
            message_file = MIMEText(messageFile, 'base64', 'utf-8')
            message["Content-Type"] = 'application/octet-stream'
            # 目前发送附件不能使用message_file["Content-Disposition"] = 'attachment; filename="aaa.txt"'方式发送信息
            message_file.add_header('Content-Disposition', 'attachment', filename=os.path.basename(filePath))
            # 附件内容
            message.attach(message_file)
    except Exception as e:
        print "附件发送失败：" + str(e)

    # 一共三个参数，第一个为发送文本信息，第二个发送类型，第三个发送信息的编码。如果想要发送html类型的信息，仅需要修改第二个参数'plain'为'html'即可
    # 文本信息,使用‘plain’属性不能正常显示
    message_text = MIMEText(messageText, 'html', 'utf-8')
    message.attach(message_text)

    # 超文本信息
    message_html = MIMEText(messageHTML, 'html', 'utf-8')
    message.attach(message_html)


    try:
        # 因为现在很多服务商做了安全验证，所有在发送邮件的时候需要把原来的smtplib.SMTP()改成现在的smtplib.SMTP_SSL()方式
        smtpObj = smtplib.SMTP_SSL()
        # 链接邮件服务器
        smtpObj.connect(mail_host, mail_port)
        # 登录邮件系统
        smtpObj.login(mail_user, mail_pass)
        # 发送邮件信息
        smtpObj.sendmail(sender,receivers,message.as_string())
        print "邮件发送成功"
    except Exception as e:
        print("邮件发送失败，错误信息：" + str(e))


if __name__ == '__main__':
    sendMail()

```
