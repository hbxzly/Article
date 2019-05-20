#### OpenSSL有两种运行模式：交互模式和批处理模式。直接输入openssl回车进入交互模式，输入带命令选项的openssl进入批处理模式。

#### (1) 配置文件OpenSSL的默认配置文件位置不是很固定，可以用openssl ca命令得知。你也可以指定自己的配置文件。当前只有三个OpenSSL命令会使用这个配置文件：ca, req, x509。有望未来版本会有更多命令使用配置文件。

#### (2)消息摘要算法支持的算法包括：MD2, MD4, MD5, MDC2, SHA1(有时候叫做DSS1), RIPEMD-160。SHA1和RIPEMD-160产生160位哈西值,其他的产生128位。除非出于兼容性考虑，否则推荐使用SHA1或者RIPEMD-160。除了RIPEMD-160需要用rmd160命令外，其他的算法都可用dgst命令来执行。OpenSSL对于SHA1的处理有点奇怪，有时候必须把它称作DSS1来引用。消息摘要算法除了可计算哈西值，还可用于签名和验证签名。签名的时候，对于DSA生成的私匙必须要和DSS1(即SHA1)搭配。而对于RSA生成的私匙，任何消息摘要算法都可使用。

##### 消息摘要算法应用例子
```
# 用SHA1算法计算文件file.txt的哈西值，输出到stdout
$ openssl dgst -sha1 file.txt

# 用SHA1算法计算文件file.txt的哈西值,输出到文件digest.txt
$ openssl sha1 -out digest.txt file.txt

# 用DSS1(SHA1)算法为文件file.txt签名,输出到文件dsasign.bin
# 签名的private key必须为DSA算法产生的，保存在文件dsakey.pem中
$ openssl dgst -dss1 -sign dsakey.pem -out dsasign.bin file.txt

# 用dss1算法验证file.txt的数字签名dsasign.bin，
# 验证的private key为DSA算法产生的文件dsakey.pem
$ openssl dgst -dss1 -prverify dsakey.pem -signature dsasign.bin file.txt

# 用sha1算法为文件file.txt签名,输出到文件rsasign.bin
# 签名的private key为RSA算法产生的文件rsaprivate.pem
$ openssl sha1 -sign rsaprivate.pem -out rsasign.bin file.txt

# 用sha1算法验证file.txt的数字签名rsasign.bin，
# 验证的public key为RSA算法生成的rsapublic.pem
$ openssl sha1 -verify rsapublic.pem -signature rsasign.bin file.txt
```

#### (3) 对称密码OpenSSL支持的对称密码包括Blowfish, CAST5, DES, 3DES(Triple DES), IDEA, RC2, RC4以及RC5。OpenSSL 0.9.7还新增了AES的支持。很多对称密码支持不同的模式，包括CBC, CFB, ECB以及OFB。对于每一种密码，默认的模式总是CBC。需要特别指出的是，尽量避免使用ECB模式，要想安全地使用它难以置信地困难。enc命令用来访问对称密码，此外还可以用密码的名字作为命令来访问。除了加解密，base64可作为命令或者enc命令选项对数据进行base64编码/解码。当你指定口令后，命令行工具会把口令和一个8字节的salt(随机生成的)进行组合，然后计算MD5 hash值。这个hash值被切分成两部分：加密钥匙(key)和初始化向量(initialization vector)。当然加密钥匙和初始化向量也可以手工指定，但是不推荐那样，因为容易出错。

##### 对称加密应用例子
```
# 用DES3算法的CBC模式加密文件plaintext.doc，
# 加密结果输出到文件ciphertext.bin
$ openssl enc -des3 -salt -in plaintext.doc -out ciphertext.bin

# 用DES3算法的OFB模式解密文件ciphertext.bin，
# 提供的口令为trousers，输出到文件plaintext.doc
#注意：因为模式不同，该命令不能对以上的文件进行解密
$ openssl enc -des-ede3-ofb -d -in ciphertext.bin -out plaintext.doc -pass pass:trousers

# 用Blowfish的CFB模式加密plaintext.doc，口令从环境变量PASSWORD中取
# 输出到文件ciphertext.bin
$ openssl bf-cfb -salt -in plaintext.doc -out ciphertext.bin -pass env:PASSWORD

# 给文件ciphertext.bin用base64编码，输出到文件base64.txt
$ openssl base64 -in ciphertext.bin -out base64.txt

# 用RC5算法的CBC模式加密文件plaintext.doc
# 输出到文件ciphertext.bin，
# salt、key和初始化向量(iv)在命令行指定
$ openssl rc5 -in plaintext.doc -out ciphertext.bin -S C62CB1D49F158ADC -iv E9EDACA1BD7090C6 -K89D4B1678D604FAA3DBFFD030A314B29
```

#### (4)公匙密码
##### 4.1 Diffie-Hellman被用来做钥匙协商(key agreement)，具有保密(secrecy)功能，但是不具有加密(encryption)或者认证(authentication)功能，因此在进行协商前需用别的方式对另一方进行认证。首先，Diffie-Hellman创建一套双方都认可的参数集，包括一个随机的素数和生成因子(generator value，通常是2或者5)。基于这个参数集，双方都计算出一个公钥匙和私钥匙，公钥匙交给对方，对方的公钥匙和自己的私钥匙用来计算共享的钥匙。OpenSSL 0.9.5 提供了dhparam命令用来生成参数集，但是生成公钥匙和私钥匙的命令dh和gendh已不推荐使用。未来版本可能会加上这个功能。
        
 ##### Diffie-Hellman应用例子
 ```
 # 使用生成因子2和随机的1024-bit的素数产生D0ffie-Hellman参数
 # 输出保存到文件dhparam.pem
 $ openssl dhparam -out dhparam.pem -2 1024
 
 # 从dhparam.pem中读取Diffie-Hell参数，以C代码的形式
 # 输出到stdout
 $ openssl dhparam -in dhparam.pem -noout -C
```
 
##### 4.2 数字签名算法(Digital Signature Algorithm, DSA)主要用来做认证，不能用来加密(encryption)或者保密(secrecy)，因此它通常和Diffie-Hellman配合使用。在进行钥匙协商前先用DSA进行认证(authentication)。有三个命令可用来完成DSA算法提供的功能。dsaparam命令生成和检查DSA参数，还可生成DSA私钥匙。gendsa命令用来为一套DSA参数生成私钥匙，这把私钥匙可明文保存，也可指定加密选项加密保存。可采用DES，3DES，或者IDEA进行加密。dsa命令用来从DSA的私钥匙中生成公钥匙，还可以为私钥匙加解密，或者改变私钥匙加密的口令。
 
 ##### DSA应用例子
 ```
 # 生成1024位DSA参数集，并输出到文件dsaparam.pem$ openssl dsaparam -out dsaparam.pem 1024# 使用参数文件dsaparam.pem生成DSA私钥匙，# 采用3DES加密后输出到文件dsaprivatekey.pem$ openssl gendsa -out dsaprivatekey.pem -des3 dsaparam.pem# 使用私钥匙dsaprivatekey.pem生成公钥匙，# 输出到dsapublickey.pem$ openssl dsa -in dsaprivatekey.pem -pubout -out dsapublickey.pem# 从dsaprivatekey.pem中读取私钥匙，解密并输入新口令进行加密，# 然后写回文件dsaprivatekey.pem$ openssl dsa -in dsaprivatekey.pem -out dsaprivatekey.pem -des3 -passin
```
 
##### 4.3 RSARSA得名于它的三位创建者：Ron Rivest, Adi Shamir, Leonard Adleman。目前之所以如此流行，是因为它集保密、认证、加密的功能于一体。不像Diffie-Hellman和DSA，RSA算法不需要生成参数文件，这在很大程度上简化了操作。有三个命令可用来完成RSA提供的功能。genrsa命令生成新的RSA私匙，推荐的私匙长度为1024位，不建议低于该值或者高于2048位。缺省情况下私匙不被加密，但是可用DES、3DES或者IDEA加密。rsa命令可用来添加、修改、删除私匙的加密保护，也可用来从私匙中生成RSA公匙，或者用来显示私匙或公匙信息。rsautl命令提供RSA加密和签名功能。但是不推荐用它来加密大块数据，或者给大块数据签名，因为这种算法的速度较来慢。通常用它给对称密匙加密，然后通过enc命令用对称密匙对大块数据加密。
 
 ##### RSA应用例子
 ```
 # 产生1024位RSA私匙，用3DES加密它，口令为trousers，
 # 输出到文件rsaprivatekey.pem
 $ openssl genrsa -out rsaprivatekey.pem -passout pass:trousers -des3 1024
 
 # 从文件rsaprivatekey.pem读取私匙，用口令trousers解密，
 # 生成的公钥匙输出到文件rsapublickey.pem
 $ openssl rsa -in rsaprivatekey.pem -passin pass:trousers -pubout -out rsapubckey.pem
 
 # 用公钥匙rsapublickey.pem加密文件plain.txt，
 # 输出到文件cipher.txt
 $ openssl rsautl -encrypt -pubin -inkey rsapublickey.pem -in plain.txt -out cipher.txt
 
 # 使用私钥匙rsaprivatekey.pem解密密文cipher.txt，
 # 输出到文件plain.txt
 $ openssl rsautl -decrypt -inkey rsaprivatekey.pem -in cipher.txt -out plain.txt
 
 # 用私钥匙rsaprivatekey.pem给文件plain.txt签名，
 # 输出到文件signature.bin
 $ openssl rsautl -sign -inkey rsaprivatekey.pem -in plain.txt -out signature.bin
 
 # 用公钥匙rsapublickey.pem验证签名signature.bin，
 # 输出到文件plain.txt
 $ openssl rsautl -verify -pubin -inkey rsapublickey.pem -in signature.bin -out plain
```
 
#### (5) S/MIME[Secure Multipurpose Internet Mail Exchange]S/MIME应用于安全邮件交换，可用来认证和加密，是PGP的竞争对手。与PGP不同的是，它需要一套公匙体系建立信任关系，而PGP只需直接从某个地方获取对方的公匙就可以。然而正因为这样，它的扩展性比PGP要好。另一方面，S/MIME可以对多人群发安全消息，而PGP则不能。命令smime可用来加解密、签名、验证S/MIME v2消息(对S/MIME v3的支持有限而且很可能不工作)。对于没有内置S/MIME支持的应用来说，可通过smime来处理进来(incoming)和出去(outgoing)的消息。
 
##### RSA应用例子
```
# 从X.509证书文件cert.pem中获取公钥匙，
# 用3DES加密mail.txt# 输出到文件mail.enc
$ openssl smime -encrypt -in mail.txt -des3 -out mail.enc cert.pem

# 从X.509证书文件cert.pem中获取接收人的公钥匙，
# 用私钥匙key.pem解密S/MIME消息mail.enc，
# 结果输出到文件mail.txt
$ openssl smime -decrypt -in mail.enc -recip cert.pem -inkey key.pem -out mail.txt

# cert.pem为X.509证书文件，用私匙key,pem为mail.txt签名，
# 证书被包含在S/MIME消息中，输出到文件mail.sgn
$ openssl smime -sign -in mail.txt -signer cert.pem -inkey key.pem -out mail.sgn

# 验证S/MIME消息mail.sgn，输出到文件mail.txt
# 签名者的证书应该作为S/MIME消息的一部分包含在mail.sgn中
$ openssl smime -verify -in mail.sgn -out mail.txt
```

#### (6) 口令和口令输入(passphase)OpenSSL口令选项名称不是很一致，通常为passin和passout。可以指定各种各样的口令输入来源，不同的来源所承担的风险取决于你的接受能力。stdin这种方式不同于缺省方式，它允许重定向标准输入，而缺省方式下是直接从真实的终端设备(TTY)读入口令的。pass:直接在命令行指定口令为password。不推荐这样使用。env:从环境变量中获取口令，比pass方式安全了些，但是进程环境仍可能被别有用心的进程读到。file:从文件中获取，注意保护好文件的安全性。fd:从文件描述符中读取。通常情况是父进程启动OpenSSL命令行工具，由于OpenSSL继承了父进程的文件描述符，因此可以从文件描述符中读取口令。

#### (7) 重置伪随机数生成器(Seeding the Pseudorandom Number Generator)对于OpenSSL，正确地重置PRNG(Pseudo Random Number Generator)很重要。命令行工具会试图重置PRNG，当然这不是万无一失的。如果错误发生，命令行工具会生成一条警告，这意味着生成的随机数是可预料的，这时就应该采用一种更可靠的重置机制而不能是默认的。在Windows系统，重置PRNG的来源很多，比如屏幕内容。在Unix系统，通常通过设备/dev/urandom来重置PRNG。从0.9.7开始，OpenSSL还试图通过连接EGD套接字来重置PRNG。除了基本的重置来源，命令行工具还会查找包含随机数据的文件。假如环境变量RANDFILE被设置，它的值就可以用来重置PRNG。如果没有设置，则HOME目录下的.rnd文件将会使用。OpenSSL还提供了一个命令rand用来指定重置来源文件。来源文件之间以操作系统的文件分割字符隔开。对于Unix系统，如果来源文件是EGD套接字，则会从EGD服务器获取随机数。EGD服务器是用Perl写成的收集重置来源的daemon，可运行在装了Perl的基于Unix的系统，见http://egd.sourceforge.net。如果没有/dev/random或者/dev/urandom，EGD是一个不错的候选。EGD不能运行在Windows系统中。对于Windows环境，推荐使用EGADS(Entropy Gathering And Distribution System)。它可运行在Unix和Windows系统中，见http://www.securesw.com/egads。(本文参考O’Reilly-Network Security with OpenSSL)附加测试实例：

##### 消息摘要算法应用例子
```
# 用SHA1算法计算文件file.txt的哈西值，输出到stdout
$ openssl dgst -sha1 file.txt
[root@server02 ~]# echo zhaohang > file.txt[root@server02 ~]
# openssl dgst -sha1 file.txt
SHA1(file.txt)= cf017022db32f04cb57d2ec1ae6b39751a6155e4

# 用SHA1算法计算文件file.txt的哈西值,输出到文件digest.txt
$ openssl sha1 -out digest.txt file.txt
[root@server02 ~]# openssl sha1 -out digest.txt file.txt
[root@server02 ~]# cat digest.txt 
SHA1(file.txt)= cf017022db32f04cb57d2ec1ae6b39751a6155e4

# 用DSS1(SHA1)算法为文件file.txt签名,输出到文件dsasign.bin
# 签名的private key必须为DSA算法产生的，保存在文件dsakey.pem中
$ openssl dgst -dss1 -sign dsakey.pem -out dsasign.bin file.txt

# 用dss1算法验证file.txt的数字签名dsasign.bin，
# 验证的private key为DSA算法产生的文件dsakey.pem
$ openssl dgst -dss1 -prverify dsakey.pem -signature dsasign.bin file.txt

# 用sha1算法为文件file.txt签名,输出到文件rsasign.bin
# 签名的private key为RSA算法产生的文件rsaprivate.pem
$ openssl sha1 -sign rsaprivate.pem -out rsasign.bin file.txt

# 用sha1算法验证file.txt的数字签名rsasign.bin，
# 验证的public key为RSA算法生成的rsapublic.pem
$ openssl sha1 -verify rsapublic.pem -signature rsasign.bin file.txt
```

##### 对称加密应用例子
```
# 用DES3算法的CBC模式加密文件 file.txt
# 加密结果输出到文件 ciphertext.bin
$ openssl enc -des3 -salt -in file.txt -out ciphertext.bin -pass pass:123456

# 用DES3算法的CBC模式解密文件 ciphertext.bin
$ openssl enc -des3 -d -in ciphertext.bin -pass pass:123456

# 用DES3算法的OFB模式解密文件ciphertext.bin：
# 提供的口令为trousers，输出到文件plaintext.doc
# 注意：因为模式不同，该命令不能对以上的文件进行解密
$ openssl enc -des-ede3-ofb -d -in ciphertext.bin -out file.txt -pass pass:123456

# 用Blowfish的CFB模式加/解密 file.txt，口令从环境变量 HOSTNMAE 中取
$ openssl bf-cfb -salt -in file.txt -out ciphertext.bin -pass env:HOSTNAME$ openssl bf-cfb -d -in ciphertext.bin -pass env:HOSTNAME
```

##### DSA应用例子 - 数字签名算法
```
# 给文件ciphertext.bin用base64编码，输出到文件base64.txt
$ openssl base64 -in ciphertext.bin -out base64.txt

# 使用参数文件dsaparam.pem生成DSA私钥匙，
# 采用3DES加密后输出到文件dsaprivatekey.pem
$ openssl gendsa -out dsaprivatekey.pem -des3 dsaparam.pem

# 使用私钥匙dsaprivatekey.pem生成公钥匙，
# 输出到dsapublickey.pem
$ openssl dsa -in dsaprivatekey.pem -pubout -out dsapublickey.pem

# 从dsaprivatekey.pem中读取私钥匙，解密并输入新口令进行加密，
# 然后写回文件dsaprivatekey.pem
$ openssl dsa -in dsaprivatekey.pem -out dsaprivatekey.pem -des3 -passin
```

##### RSA应用例子
```
# 产生1024位RSA私匙，用3DES加密它，口令为123456
# 输出到文件rsaprivatekey.pem
$ openssl genrsa -out rsaprivatekey.pem -passout pass:123456 -des3 1024

# 从文件rsaprivatekey.pem读取私匙，用口令123456解密
# 生成的公钥匙输出到文件rsapublickey.pem
$ openssl rsa -in rsaprivatekey.pem -passin pass:123456 -pubout -out rsapubckey.pem

# 用公钥匙rsapublickey.pem加密文件 file.txt
# 输出到文件cipher.txt
$ openssl rsautl -encrypt -pubin -inkey rsapublickey.pem -in file.txt -out cipher.txt

# 使用私钥匙rsaprivatekey.pem解密密文cipher.txt
# 输出到文件 file2.txt
$ openssl rsautl -decrypt -inkey rsaprivatekey.pem -in cipher.txt -out file2.txt

# 用私钥匙rsaprivatekey.pem给文件file.txt签名
# 输出到文件signature.bin
$ openssl rsautl -sign -inkey rsaprivatekey.pem -in plain.txt -out signature.bin

# 用公钥匙rsapublickey.pem验证签名signature.bin
# 输出到文件plain.txt
$ openssl rsautl -verify -pubin -inkey rsapublickey.pem -in signature.bin -out file.txt
