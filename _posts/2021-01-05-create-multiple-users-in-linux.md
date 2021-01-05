---
layout: post
title: "Create Multiple Users in Linux"
date: 2021-01-05
categories: blog
permalink: /:categories/:year/:month/:title.html
---

## Linux下批量创建用户及修改密码[[github](https://github.com/zhchuu/mia-sharing/blob/master/Add_multiple_users_in_linux.md)]

### 1. 前言

服务器管理员创建用户、修改密码是常有之事，一般来说有三种命令：`useradd` / `adduser` / `newusers`。
1. `useradd`命令是一个更加底层的命令，创建用户后仍然有许多东西需要配置，经过查阅大量教程资料和亲身实践，我强烈不建议使用；
2. `adduser`相当友好，有一定的交互性，帮助更好地创建用户和设置密码，该命令底层调用的是`useradd`，推荐使用；
3. `newusers`命令读取文件，用文件内的信息自动创建新用户，但我觉得这个命令也不够友好，但后面仍会介绍它的用法。

下面我会：
1. 使用`adduser`配合bash**批量**创建用户和修改密码；
2. 使用`newusers`**批量**创建用户。

### 2. 预备知识

一般来说创建用户有几个重要的属性需要设置：

- [ ] 用户名（uid）
- [ ] Group（gid）
- [ ] 密码
- [ ] home目录
- [ ] home目录权限

> 如何查看目前已有的用户？

查看`/etc/passwd`文件，阅读各式为：

```bash
pw_name:pw_passwd:pw_uid:pw_gid:pw_gecos:pw_dir:pw_shell
```

其中：
- pw_name: 用户名
- pw_passwd: 用户密码（在这里被隐藏了，全部都为x）
- pw_uid: 用户的ID
- pw_gid: 用户所在Group的ID
- pw_gecos: 定义一些comments，对该用户的描述等等
- pw_dir: 定义该用户的home目录
- pw_shell: 定义用户的默认shell（一般是/bin/bash）

例如：

```bash
user1:x:1001:1001::/share/user1:/bin/bash
```

表示用户名为user1，用户的ID为1001，所属Group的ID为1001，comments无内容，home目录为/share/user1，shell为/bin/bash。

> 如何查看目前已有的Group以及创建Group？

查看`/etc/group`文件。使用`groupadd`命令创建Group（具体使用方法请查阅文档）。

例如：

```bash
lab:x:1001:
```

表示Group名为lab，编号为1001。


### 3. 方法一：adduser命令

**步骤一**：把要创建的用户名写到文件中

例如：
```bash
user_aaa
user_bbb
user_ccc
user_ddd
```

写入文件`users.txt`中。

**步骤二**：编写bash脚本

此时根据系统的不同，`adduser`命令有不同的执行方法。

> Ubuntu 18.04：

修改`/etc/adduser.conf`文件（修改前请备份），这里面是执行adduser时参数的默认配置，里面每个字段都有详细的介绍，这里只挑出几个需要修改的讲讲：

- `DHOME`: 该字段表示要创建的user的目录是哪里。这里我改成：`DHOME=/share`，表示要根目录要创建在`/share`下
- `USERGROUPS`: 该字段表示，创建user时，是否同时创建一个与user同名的Group。这里我改成：`USERGROUPS=no`，表示不需要创建
- `USERS_GID`: 该字段表示，如果上面的`USERGROUPS`为no，那么user默认属于哪个Group。这里我改成：`USERS_GID=1001`，表示我要让创建的user默认进入1001这个Group（这里可以打开`/etc/group`文件看看自己创建的Group的ID是多少）
- `DIR_MODE`: 该字段表示，为用户创建的根目录的权限设置成什么。这里我改成：`DIR_MODE=0700`，表示我不希望user之间能互相访问对方的根目录

创建`add_users.sh`脚本：

```bash
#!/bin/bash
for user in `cat users.txt`
do
	sudo adduser $user --gecos "First Last,RoomNumber,WorkPhone,HomePhone" --disabled-password
        echo "$user:123456" | sudo chpasswd
done
```

其中：
- `--gecos`表示要把用户的comments字段设置为以下内容
- `--disabled-password`表示不设置密码
- `chpasswd`命令可以修改密码，123456可以修改成自己想要的密码

执行bash：

```bash
$ sudo bash ./add_users.sh 
```

以上修改`/etc/adduser.conf`文件的方法不是必须的，用户也可以执行`adduser -h`去查看每个参数对应的设置方法，然后一个一个写在bash中。

> Fedora 30：

创建`add_users.sh`脚本：

```bash
#!/bin/bash
for user in `cat users.txt`
do
	sudo adduser $user -c "First Last,RoomNumber,WorkPhone,HomePhone" -b /share -g 1001 -m
        echo "$user:123456" | sudo chpasswd
done
```

其中：
- `-c`表示设置comments为以下内容
- `-b`表示Base dir为/share，创建的用户根目录都在此
- `-g`表示Group ID为1001
- `-m`表示需要自动创建根目录
- `chpasswd`命令可以修改密码，123456可以修改成自己想要的密码

执行bash：

```bash
$ sudo bash ./add_users.sh 
```

至此创建完毕，下面验证一下是否正确创建：

运行：
```bash
$ ls -l /share/
```
```bash
total 24
drwx------. 3 user1    lab 4096 Jan  5 15:52 user1
drwx------. 3 user2    lab 4096 Jan  5 16:08 user2
drwx------. 3 user_aaa lab 4096 Jan  5 17:35 user_aaa
drwx------. 3 user_bbb lab 4096 Jan  5 17:35 user_bbb
drwx------. 3 user_ccc lab 4096 Jan  5 17:35 user_ccc
drwx------. 3 user_ddd lab 4096 Jan  5 17:35 user_ddd
```
可以看到用户的根目录都在正确的位置。

运行：
```bash
$ su user_aaa
```
并输入密码，即可正确登陆user_aaa。

运行：
```bash
$ tail /etc/passwd
```
```bash
user_aaa:x:1003:1001:First Last,RoomNumber,WorkPhone,HomePhone:/share/user_aaa:/bin/bash
user_bbb:x:1004:1001:First Last,RoomNumber,WorkPhone,HomePhone:/share/user_bbb:/bin/bash
user_ccc:x:1005:1001:First Last,RoomNumber,WorkPhone,HomePhone:/share/user_ccc:/bin/bash
user_ddd:x:1006:1001:First Last,RoomNumber,WorkPhone,HomePhone:/share/user_ddd:/bin/bash
```
可以看到用户信息都正确写入`/etc/passwd`中。

至此，第一种方法介绍完毕。以上操作批量创建了用户，并且设置了统一的密码。如果有其他个性化的需求，例如每个用户不一样的密码，可以自行修改bash脚本，添加对应的参数即可。

### 4. 方法二：newusers命令

**步骤一**：把要创建的用户以及相关信息按照格式写入到文件中

例如：
```bash
user_eee:123456:1007:1001:,,,:/share/user_eee:/bin/bash
user_fff:123456:1008:1001:,,,:/share/user_fff:/bin/bash
user_ggg:123456:1009:1001:,,,:/share/user_ggg:/bin/bash
```
写入文件`users_details.txt`中。其中，必须按照第2节中介绍的`/etc/passwd`文件里的格式编写。

**步骤二**：执行命令

运行：
```bash
$ sudo newusers users_details.txt
```

至此创建完毕，下面验证一下是否正确创建：

运行：
```bash
$ tail /etc/passwd
```
```bash
user_aaa:x:1003:1001:First Last,RoomNumber,WorkPhone,HomePhone:/share/user_aaa:/bin/bash
user_bbb:x:1004:1001:First Last,RoomNumber,WorkPhone,HomePhone:/share/user_bbb:/bin/bash
user_ccc:x:1005:1001:First Last,RoomNumber,WorkPhone,HomePhone:/share/user_ccc:/bin/bash
user_ddd:x:1006:1001:First Last,RoomNumber,WorkPhone,HomePhone:/share/user_ddd:/bin/bash
user_eee:x:1007:1001:,,,:/share/user_eee:/bin/bash
user_fff:x:1008:1001:,,,:/share/user_fff:/bin/bash
user_ggg:x:1009:1001:,,,:/share/user_ggg:/bin/bash
```
可以看到用户信息都正确写入`/etc/passwd`中。

至此，第二种方法介绍完毕。

## 参考

[Understanding /etc/group File](https://www.cyberciti.biz/faq/understanding-etcgroup-file/)  \\
[How to add a unix/linux user in a bash script](https://unix.stackexchange.com/questions/79909/how-to-add-a-unix-linux-user-in-a-bash-script)  \\
[adduser(8) - Linux man page](https://linux.die.net/man/8/adduser)  \\
[newusers(8) - Linux man page](https://linux.die.net/man/8/newusers)
