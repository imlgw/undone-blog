# Linux 的帐号管理
## /etc/passwd
每一行都代表一个帐号，有几行就代表有几个帐号在你的系统中！ 不过需要特别留意的是，里头很多帐号本
来就是系统正常运行所必须要的，我们可以简称他为系统帐号， 例如 bin, daemon, adm, nobody 等等，这些帐号请不要随意的杀掉
```c
$ head -10 /etc/passwd
root:x:0:0:root:/root:/bin/zsh
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
```
分为7个部分，冒号：分割

> 账号名称 | 密码 | UID | GID | 使用者信息说明 | 主文件夹 | Shell
1. 账户名称：不用多说
1. 密码：早期Unix密码是在第二个字段上，后续为了安全将其移动到了`/etc/shadow`中
3. UID：0: 系统管理员（可以有多个UID=0的用户，都可以作为系统管理员）1~999：系统账号，保留给操作系统用的账户ID（一般是不可登录的）
1000~60000：可登录账号，一般使用者使用
4. GID：群组ID，与 /etc/group 有关
5. 使用者信息说明栏：解释账号的意义，没有重要的用途
6. 主文件夹：使用者的主文件夹，默认的使用者主文件夹在：/home/username
7. Shell：使用者登陆系统后就会取得一个`Shell`来与系统的核心沟通以进行使用者的操作任务，这个Shell就是在这里指定的，有一个 shell 可以用来替代成让帐号无法取得`Shell`环境的登录，那就是 `/sbin/nologin`（指无法通过Shell登录系统，但是是可以使用系统资源的，像一些服务类似MySQL，Redis都是nologin的，这也是为了安全）
## /etc/shadow
```c
$ tail -10 /etc/shadow
colord:!!:17827::::::
pulse:!!:17827::::::
gdm:!!:17827::::::
radvd:!!:17827::::::
sssd:!!:17827::::::
gnome-initial-setup:!!:17827::::::
avahi:!!:17827::::::
saned:!!:18532::::::
redis:!!:18532::::::
zmj:$6$sMnrwwe2$5LumOHcnAAHd3FRj39yI7iwOyVJlbWUeDTsZWk8NkVG9OXePxAnaO5H0daFiDA7xnmdte7eOQtCpIp/Q8I9d5/:18562:0:99999:7:::
```
用户密码相关的信息，冒号：分为9个部分
1. **账号名称：** 密码也需要与帐号对应
2. **密码：** 经过加密的密码
3. **最近更改密码的日期：** 这里是指的从1970-1-1年到今天的累积天数
4. **密码不可被更动的天数：** 这个帐号的密码在最近一次被更改后需要经过几天才可以再被变更
5. **密码需要重新变更的天数：** 强制要求使用者变更密码，这个字段可以指定在最近一次更改密码后， 在多少天数内需要再次的变更密
码才行。你必须要在这个天数内重新设置你的密码，否则这个帐号的密码将会 “ 变为过期特性 ” 。 而如果像上面的 99999 （计算为 273
年） 的话，那就表示密码的变更没有强制性之意
6. **密码需要变更期限前的警告天数**：当帐号的密码有效期限快要到的时候 （第 5 字段），系统会依据这个字段的设置，发出 “ 警告 ” 言论给这个帐号
7. **密码过期后的帐号宽限时间（密码失效日）**：是在密码过期几天后，如果使用者还是没有登陆更改密码，那么这个帐号的密码将会 “ 失效
8. **帐号失效日期**：都是使用 1970 年以来的总日数设置。这个字段表示： 这个帐号在此字段规定的日期之后，将无法再使用。 就是所谓的 “ 帐号失效 ” ，此时不论你的密码是否有过期，这个 “ 帐号 ” 都不能再被使用！ 这个字段会被使用通常应该是在 “ 收费服务 ” 的系统中，你可以规定一个日期让该帐号不能再使用
9. **保留**

## /etc/group
```c
$ tail -10 group
stapusr:x:156:
stapsys:x:157:
stapdev:x:158:
gnome-initial-setup:x:977:
slocate:x:21:
avahi:x:70:
saned:x:976:
redis:x:975:
docker:x:974:
zmj:x:1001:
```
1. **群组名称**: 与GID对应
2. **群组密码**：通常不需要设置，这个设置通常是给**群组管理员**使用的
3. **GID**：群组的ID
4. **此群组支持的帐号名称**：我们知道一个帐号可以加入多个群组，那某个帐号想要加入此群组时，将该帐号填入这个字段即可，逗号分割，不要空格 如：root:x:0:dmtsai,alex

> newgrp: 切换有效群组（Group第一个输出的群组即为有效群组，新建文件时所属的Group），切换的群组必须是你已经有支持的群组

# 账户管理
## 增加用户
```c
[root@study ~]# useradd [-u UID] [-g 初始群组 ] [-G 次要群组 ] [-mM]\
> [-c 说明栏 ] [-d 主文件夹绝对路径 ] [-s shell] 使用者帐号名
选项与参数：
-u ：后面接的是 UID ，是一组数字。直接指定一个特定的 UID 给这个帐号；
-g ：后面接的那个群组名称就是我们上面提到的 initial group 啦～
该群组的 GID 会被放置到 /etc/passwd 的第四个字段内。
-G ：后面接的群组名称则是这个帐号还可以加入的群组。
这个选项与参数会修改 /etc/group 内的相关数据喔！
-M ：强制！不要创建使用者主文件夹！（系统帐号默认值）
-m ：强制！要创建使用者主文件夹！（一般帐号默认值）
-c ：这个就是 /etc/passwd 的第五栏的说明内容啦～可以随便我们设置的啦～
-d ：指定某个目录成为主文件夹，而不要使用默认值。务必使用绝对路径！
-r ：创建一个系统的帐号，这个帐号的 UID 会有限制 （参考 /etc/login.defs）
-s ：后面接一个 shell ，若没有指定则默认是 /bin/bash 的啦～
-e ：后面接一个日期，格式为“YYYY-MM-DD”此项目可写入 shadow 第八字段，
亦即帐号失效日的设置项目啰；
-f ：后面接 shadow 的第七字段项目，指定密码是否会失效。0为立刻失效，
-1 为永远不失效（密码只会过期而强制于登陆时重新设置而已。）
```
## 设置密码
```c
[root@study ~]# passwd [--stdin] [ 帐号名称 ] <==所有人均可使用来改自己的密码
[root@study ~]# passwd [-l] [-u] [--stdin] [-S] \
> [-n 日数 ] [-x 日数 ] [-w 日数 ] [-i 日期 ] 帐号 <==root 功能
选项与参数：
--stdin ：可以通过来自前一个管线的数据，作为密码输入，对 shell script 有帮助！
-l ：是 Lock 的意思，会将 /etc/shadow 第二栏最前面加上 ! 使密码失效；
-u ：与 -l 相对，是 Unlock 的意思！
-S ：列出密码相关参数，亦即 shadow 文件内的大部分信息。
-n ：后面接天数，shadow 的第 4 字段，多久不可修改密码天数
-x ：后面接天数，shadow 的第 5 字段，多久内必须要更动密码
-w ：后面接天数，shadow 的第 6 字段，密码过期前的警告天数
-i ：后面接“日期”，shadow 的第 7 字段，密码失效日期
```
# ACL权限控制
ACL 是 Access Control List 的缩写，主要的目的是在提供传统的 owner,group,others 的 read,write,execute 权限之外的细部权限设置。

ACL 可以针对单一使用者，单一文件或目录来进行 r,w,x 的权限规范，对于需要特殊权限的使用状况非常有帮助。
- 使用者 （ user ）：可以针对使用者来设置权限；
- 群组 （ group ）：针对群组为对象来设置其权限；
- 默认属性 （ mask ）：还可以针对在该目录下在创建新文件 / 目录时，规范新数据的默认权限
## setfacl
```c
[root@study ~]# setfacl [-bkRd] [{-m|-x} acl 参数 ] 目标文件名
选项与参数：
-m ：设置后续的 acl 参数给文件使用，不可与 -x 合用；
-x ：删除后续的 acl 参数，不可与 -m 合用；
-b ：移除“所有的” ACL 设置参数；
-k ：移除“默认的” ACL 参数，关于所谓的“默认”参数于后续范例中介绍；
-R ：递回设置 acl ，亦即包括次目录都会被设置起来；
-d ：设置“默认 acl 参数”的意思！只对目录有效，在该目录新建的数据会引用此默认值
//设置值中的 u 后面无使用者列表，代表设置该文件拥有者
```
设置某个`文件/目录`的ACL设置项目，`u:xx:rwx`：针对某个用户设置ACL权限，`g:xx:rwx`：针对某个群组设置ACL权限

`$ setfacl -m u:zmj:rwx zmjShell.sh` 

## getfacl
设置某个`文件/目录`的ACL设置项目
```c
$ getfacl zmjShell.sh
# file: zmjShell.sh
# owner: root
# group: root
user::rwx
user:zmj:rwx
group::---
mask::rwx
other::---
```