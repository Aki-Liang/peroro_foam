# BKMs

### Centos 添加用户
添加普通用户

```
    [root@server ~]# useradd admin        //添加一个名为admin的用户
    [root@server ~]# passwd admin         //修改密码
    Changing password for user admin.
    New UNIX password:                              //在这里输入新密码
    Retype new UNIX password:                  //再次输入新密码
    passwd: all authentication tokens updated successfully
```

### Centos新用户 开启SU权限

修改/etc/sudoers文件，找到下面一行，在root下面添加一行，如下所示：

```
    ## Allow root to run any commands anywhere
    root ALL=(ALL) ALL
    admin root ALL=(ALL) ALL
```

### Centos修改用户权限

修改/etc/passwd文件，找到如下行，把用户ID修改为0，如下所示：

admin:x:500:0:admin:/home/admin:/bin/bash

改为

admin:x:0:0:admin:/home/admin:/bin/bash

修改后保存，用新账户登录后，直接获取的就是root帐号的权限。


### Centos挂载exfat硬盘
1.Install the nux repo for CentOS 7
yum install -y http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm

yum install exfat-utils fuse-exfat
装这两包
3.插U盘，cat /proc/partitions, 看看是否挂载成功

4.cd /media
mkdir udisk
mount -t exfat /dev/sdbX(X是你的盘号） udisk