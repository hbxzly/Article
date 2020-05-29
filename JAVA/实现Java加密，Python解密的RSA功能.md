@[toc](实现Java加密，Python解密的非对称RSA功能)

## 摘要
> 因为最近业务需要使用到openssl的rsa非对称加密算法，研究了下它的使用方式，但是特殊在于前端分IOS和android两端，所以前端部门要求使用java给他们做一个加密工具包，但是因为服务端是python做的，所以需要两端的数据能够共通。研究了几天终于搞定了，下面是一些重要的代码以及一些我踩过的坑，分享一下。

> 欢迎访问我的[Github](https://github.com/carolcoral)

> 相关详细Java端代码请查看 [FileEncryptDecrypt](https://github.com/carolcoral/demo4java/blob/master/common/src/main/java/site/cnkj/util/FileEncryptDecrypt.java)

> 相关详细Python端代码请查看 [FileDecryptEncrypt](https://github.com/CodingEverThen/CommonUtilsForPython/blob/master/common/FileDecryptEncrypt.py)
## OpenSSL[官网](https://www.openssl.org/)
#### 一. 编译
1. 下载：https://www.openssl.org/source/
2. 安装 OpenSSL, 请确保你已安装以下组件:

```shell
- make
- Perl 5
- an ANSI C compiler
- a development environment in form of development libraries and C
- header files
- a supported Unix operating system
```

#### 二. 安装
```
$ ./config
$ make
$ make test
$ make install
```

#### 三. 生成密钥
> 生成的密钥的路径是你当前执行命令的路径
> 这里默认生成1024长度密钥
> 公钥是基于私钥来生成的，所以必须先生成私钥

```
# 进入openssl
root@VM-0-15-ubuntu:/home/ubuntu# openssl
# 生成一个1024位的私钥文件rsa_private_key.pem
OpenSSL> genrsa -out rsa_private_key.pem 1024
# 从私钥中提取公钥rsa_public_key.pem
OpenSSL> rsa -in rsa_private_key.pem -out rsa_public_key.pem -outform PEM -pubout
# 将私钥转换成 DER 格式
OpenSSL> rsa -in rsa_private_key.pem -out rsa_private_key.der -outform der
# 将公钥转换成 DER 格式
OpenSSL> rsa -in rsa_public_key.pem -out rsa_public_key.der -pubin -outform der
# 把RSA私钥转换成PKCS8格式
OpenSSL> pkcs8 -topk8 -in rsa_private_key.pem -out pkcs8_rsa_private_key.pem -nocrypt
# 从私钥创建公钥证书请求
OpenSSL> req -new -key rsa_private_key.pem -out rsa_public_key.csr
# 生成证书并签名(有效期10年)
OpenSSL> x509 -req -days 3650 -in rsa_public_key.csr -signkey rsa_private_key.pem -out rsa_public_key.crt
# 把crt证书转换为der格式
OpenSSL> x509 -outform der -in rsa_public_key.crt -out rsa_public_key.der
# 把crt证书生成私钥p12文件
OpenSSL> pkcs12 -export -out rsa_private_key.p12 -inkey rsa_private_key.pem -in rsa_public_key.crt
```

## 1.JAVA端加密
> 其实单一语言的加解密都还是比较简单的，关键在于跨语言的兼容问题上。
> 而且需要特别注意的是我使用的RSA长度是`1024`的，也就是说我单个需要加密的数据的长度不能超过 `1024/8-11=117`，实际测试单条数据最大长度是 `116 bytes`。
> 如果需要扩大需要加密的单条数据的长度，只需要在生成公钥的时候设置对应的长度即可。（(数据长度+12)*8=密钥长度）
> 其中`RSA_ALGORITHM` 设置的是算法的使用模式，因为Python端使用的是`OAEP`所以这里用的是`RSA/ECB/OAEPWithSHA-256AndMGF1Padding`
> 在Java端我们使用的公钥格式是`DER`，如果使用`PEM`则会出现异常

```java
import org.apache.commons.io.FileUtils;
import sun.misc.BASE64Decoder;
import javax.crypto.Cipher;
import javax.crypto.spec.OAEPParameterSpec;
import javax.crypto.spec.PSource;
import java.io.*;
import java.nio.charset.Charset;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.security.*;
import java.security.interfaces.RSAPublicKey;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.MGF1ParameterSpec;
import java.security.spec.X509EncodedKeySpec;
import java.util.ArrayList;
import java.util.Base64;
import java.util.List;

/*
 * @version 1.0 created by Carol on 2019/4/15 15:31
 */
public class FileEncryptDecrypt {

    private static final String RSA_ALGORITHM = "RSA/ECB/OAEPWithSHA-256AndMGF1Padding";
    private static final Charset UTF8 = Charset.forName("UTF-8");

    static {
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }

    /**
     * 加密文件
     * @param keyPath 公钥路径，DER
     * @param input 输入文件地址
     * @param output 输出文件地址
     */
    public static boolean encrypt(String keyPath, String input, String output){
        FileInputStream fileInputStream = null;
        FileOutputStream fileOutputStream = null;
        List<String> list = new ArrayList();
        try {
            byte[] buffer = Files.readAllBytes(Paths.get(keyPath));
            X509EncodedKeySpec keySpec = new X509EncodedKeySpec(buffer);
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            PublicKey publicKey = keyFactory.generatePublic(keySpec);

            File inputFile = new File(input);
            File outputFile = new File(output);
            fileInputStream = new FileInputStream(inputFile);
            fileOutputStream = new FileOutputStream(outputFile);
            byte[] inputByte = new byte[116];
            int len;
            while((len = fileInputStream.read(inputByte)) != -1){
                list.add(new String(inputByte, 0, len));
            }
            for (String s : list) {
                byte [] encrypted = encrypt(publicKey, s);
                fileOutputStream.write(encrypted);
                fileOutputStream.flush();
            }
            fileOutputStream.close();
            fileInputStream.close();
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    private static byte[] encrypt(PublicKey publicKey, String message) throws Exception {
        Cipher cipher = Cipher.getInstance(RSA_ALGORITHM,"BC");
        cipher.init(Cipher.ENCRYPT_MODE, publicKey, new OAEPParameterSpec("SHA-256", "MGF1", MGF1ParameterSpec.SHA256, PSource.PSpecified.DEFAULT));
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        return Base64.getEncoder().encode(cipher.doFinal(message.getBytes(UTF8)));
    }

    /**
     * 从字符串中加载公钥
     *
     */
    private static RSAPublicKey loadPublicKey(String publicKeyStr) throws Exception {
        try {
            byte[] buffer = new BASE64Decoder().decodeBuffer(publicKeyStr);
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            X509EncodedKeySpec keySpec = new X509EncodedKeySpec(buffer);
            return (RSAPublicKey) keyFactory.generatePublic(keySpec);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        } catch (InvalidKeySpecException e) {
            throw new RuntimeException(e);
        }
    }


    public static void main(String [] args) throws Exception {
        encrypt("F:\\dls\\chunk\\config\\rsa_public_key.der", "C:\\Users\\Carol\\Desktop\\file\\868663032830438_migu$user$login$_1554947856915.log","F:\\868663032830438_migu$user$login$_1554947856915.log");
    }

}
```

## 2.Python端解密
> Python端可以使用的模块是很多的，通常可以用`rsa、Crypto、PyCryptodome、PyCrypt`等，需要注意的是其中`PyCrypt`已经停止维护了不建议使用，但是幸运的是`PyCryp`t有一个分支叫`PyCryptodome`.

> 为了便于阅读，Java端对加密后的数据进行了Base64的格式转换，因此在Python端也同样需要对获取的加密后的数据进行Base64的转换后再进行解密.

> 通过观察，发现加密后进行Base64转换后的数据结束符为“`==`”，因此我在这里是直接根据结束符进行拆分数据的，拆分后的数据分别解密后放在同一个字符串集合里面，这里你不需要但是如果出现换行的情况是否还需要自己手动设置换行，当解密后原有的换行符依然可用.

> 但是需要注意的是在windows环境下文件的换行符是"`\r\n`"，加密后依然加密的是"`\r\n`"，但是，但我们解密后写入文件的时候应当替换"`\r\n`"为"`\n`"，否则你会发现多了一行换行，因为当Python重写写入文件的时候认为"`\r\n`"为“`回车换行`”，所以体现在文件中就是两次换行的效果了.

> 在 `_init()`方法中获取私钥的位置只需要把<code>os.path.join(os.path.dirname(os.getcwd()), "config", "rsa_private_key.pem")</code>替换成你自己的私钥路径即可.

> 在Python端我们使用的私钥格式是`PEM`.

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
"""
    @Time    : 2019/4/12
    @Author  : Carol
    @Site    : 
    @File    : FileDecryptEncrypt.py
    @Software: PyCharm
    @Description: 
"""
from Crypto.Random import get_random_bytes
from Crypto.Cipher import AES
from Crypto.Cipher import PKCS1_OAEP
from Crypto.PublicKey import RSA
from Crypto.Hash import SHA256
import base64
import traceback
import os


class FileDecryptEncrypt():
    def __init__(self):
        try:
            self.private_key = RSA.import_key(open(os.path.join(os.path.dirname(os.getcwd()), "config", "rsa_private_key.pem")).read())
        except Exception as e:
            traceback.print_exc()
            print("加载密钥出现异常")

    def decrypt(self, root_path, res_path):
        """
        RSA的文件解密
        :param root_path: 加密后的文件
        :param res_path: 解密后的文件
        :return: 解密结果
        """
        new_line = b""
        cipher = PKCS1_OAEP.new(self.private_key, hashAlgo=SHA256)
        try:
            with open(root_path, "r") as rootf:
                lines = rootf.read().split("==")
            for line in lines:
                if len(line) > 0:
                    line = line + "=="
                    b64_decoded_message = base64.b64decode(line)
                    cipherContent = cipher.decrypt(b64_decoded_message)
                    new_line = new_line + cipherContent
            if not os.path.exists(os.path.dirname(res_path)):
                os.makedirs(os.path.dirname(res_path))
            with open(res_path, "w") as resf:
                resf.write(str(new_line, encoding="UTF-8").replace("\r\n", "\n"))
            return True
        except Exception as e:
            traceback.print_exc()
        return False
        
if __name__ == '__main__':
     FileDecryptEncrypt().decrypt("F:\\868663032830438_1554947856915.log","D:\\\\868663032830438_1554947856915.log")
```

