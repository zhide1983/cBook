# Mentor Crack

## Mentor License特征

Mentor的license有如下特征：

* 通常而言，需要破解的文件只在mgcld以及mgls_asynch两个可执行文件
* 目前常见工具，只需要mgc_s和mentorall_s两个Feature
* 含ECC，用SIGN2带出
* 含VENDOR_STRING=的保护，其信息和Feature Name，Expire Date和用户数量有关系，和HostID, ISSUER, NOTICE, SN无关
* Flexlm中Feature Name是大小写不区分的

下面是一个典型的新版本license格式
```bash
SERVER lics 000EE8E9445D 27700
DAEMON mgcld

INCREMENT mgc_s mgcld 2020.120 31-dec-2020 99 3EC24A7BFF407BFAA1F6 \
	VENDOR_STRING=5F3A8E92 SN=73690 \
    SIGN2="0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 \
    0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 \
    0000 0000 0000 0000 0000 0000" 
INCREMENT mentorall_s mgcld 2020.120 31-dec-2020 99 6EE2FADBEA901E78B22F \
	VENDOR_STRING=A359314B SN=73690 \
    SIGN2="0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 \
    0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 \
    0000 0000 0000 0000 0000 0000" 
```

## Mentor工具的ECC破解


## Mentor License的制作
网上的MentorKG版本比较多，其核心在于用于计算VENDOR_STRING。常用的MentorKG，可以用下面的命令来看到帮助文档。
```bash
MentorKG.exe -help
```
根据帮助信息，通常可以修改用户数，到期时间的信息，产生floating的license。
```bash
MentorKG.exe -f -h 001122334455 -u 300 -exp 12/31/2030 -none
```
这里的-none是不产生ISSUER/NOTICE/ck等无用信息。此后，会产生一个LICENSE.txt。

此时有两种情况：
* 如果有一个mgcld的lmcrypt，则可以保留VENDOR_STRING/用户数/exp date/version不变的前提下，进行一定的修改，比如加入ISSUER等域，修改MAC地址，做成一个SRC，产生适当的license。
* 如果没有，则可以直接补上SIGN2=“0000 ... 0000"域，其中内容数量要够

```bash
SERVER xxx 001122334455 27000
DAEMON mgcld
INCREMENT mgc_s mgcld 2030.120 31-dec-2030 300 AF81972ABEFA7D7090D3 \
	VENDOR_STRING=224021CD SN=78165
INCREMENT mentorall_s mgcld 2030.120 31-dec-2030 300 2F81778A608DAEA64451 \
	VENDOR_STRING=99CB3729 SN=78165

```
添加修改后，成为：
```bash
SERVER xxx 001122334455 27000
DAEMON mgcld
INCREMENT mgc_s mgcld 2030.120 31-dec-2030 300 5F21571AA0AD6EB69420 \
	VENDOR_STRING=224021CD SUPERSEDE \
    SIGN2="0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 \
    0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 \
    0000 0000 0000 0000 0000 0000" 
INCREMENT mentorall_s mgcld 2030.120 31-dec-2030 300 9F31F7DA3F5C3EC7632B \
	VENDOR_STRING=99CB3729 SUPERSEDE \
    SIGN2="0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 \
    0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 \
    0000 0000 0000 0000 0000 0000" 
```
若保留用户数后的license key，则要求MentorKG.exe输入正确的hostid，这样可以利用license进行一个用户信息修改控制，此时license不允许进行任何修改。

若主要考虑方便，不需要进行控制，则license key可以直接删除，工具并不要求检查旧版本的license key（可以没有，但不能错，错了会启动失败）。

**注意：推荐使用MentorKG2008的版本，能比较方便的修改expire date**