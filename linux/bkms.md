# bkms

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