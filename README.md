# Why-Cronjob-Not-Work

---

作为`linux`中的定时任务工具，`crontab`被广大开发者所热爱和使用。该技术由来已久，相当成熟，但是在真正使用的时候会时不时地发现为什么`crontab`脚本没有按照预期那样执行？本文以本周笔者遇到一个`crontab`不能运行的问题为引子，详细地介绍为什么`crontab`不运行的各种原因。

## 引子

本周遇到一个`crontab`不能执行的问题，发现原因后觉得甚是有趣。

笔者通过一个`python`脚本向`/etc/cron.d`目录下的一个文件写入定时任务命令，每分钟调用一个脚本，调用的这个脚本是个`python`文件，然后发现`cron`并没有按照预期每分钟执行一次。然后笔者就将原定时任务脚本`aaa`拷贝了一份，并重新命名为`bbb`，然后将定时任务中调用脚本改成了执行一个简单的`echo`命令，然后保存退出，发现`bbb`是可以正常定时运行的，这时候，笔者就通过`file`命令想比较一下这两个文件有何不同：

```
[root@tony cron.d]# file *
aaa:     ASCII text, with no line terminators
bbb:     ASCII text
```

这个时候我们可以发现`aaa`文件出现了比较奇怪的标识：

```
with no line terminators
```

显而易见，这是在说`cron`脚本中定时命令没有行终止符，导致这个问题是因为该`cron`脚本由`python`代码生成时没有添加换行符：

```
with open('/etc/cron.d/aaa', 'w') as f:
    f.write('xxx')
```

然后笔者尝试性地在`aaa`文件中在定时命令下新增一行后，发现定时任务可以正常运行了。不得不说，这是一个很有意思的问题，`crontab`居然会因为一个换行符导致定时任务的不运行，后来`google`了一下发现，`crontab`的确存在这个机制，具体解释下面会提到。

在`google`的同时，在`ask unbuntu`上发现了这篇文章：[《Why crontab scripts are not working?》][3]，里面很多开发者罗列了他们遇到`cron`不能正常运行的各种因素，笔者大致浏览了下，发现有遇到过，也有很多并不知道的，所以想把这些因素和解决方案一一罗列下来。

## 因素

### 因素1：环境变量

#### 场景及原因

`cron`中的环境变量和系统的环境变量是不一样的，我们可以通过设置定时脚本将`cron`中的环境变量打印出来：

```
* * * * * env > /tmp/env.output
```

可以看到`cron`中的环境变量：

```
XDG_SESSION_ID=12952
SHELL=/bin/sh
USER=root
PATH=/usr/bin:/bin
PWD=/root
LANG=en_US.UTF-8
SHLVL=1
HOME=/root
LOGNAME=root
XDG_RUNTIME_DIR=/run/user/0
_=/usr/bin/env
```

查看系统的环境变量：

```
[root@tony cron.d]# env
XDG_SESSION_ID=1140
HOSTNAME=tony
TERM=xterm-256color
SHELL=/bin/bash
HISTSIZE=1000
USER=root
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/root/bin
MAIL=/var/spool/mail/root
PWD=/etc/cron.d
LANG=en_US.UTF-8
TMUX_PANE=%18
HISTCONTROL=ignoredups
SHLVL=2
HOME=/root
LOGNAME=root
_=/usr/bin/env
OLDPWD=/root
```

我们可以看到`cron`中的环境变量很多都和系统环境变量不一样（`cron`会忽略`/etc/environment`文件），尤其是`PATH`，只有`/usr/bin:/bin`，也就是说在`cron`中运行`shell`命令，如果不是全路径，只能运行`/usr/bin`或`/bin`这两个目录中的标准命令，而像`/usr/sbin`、`/usr/local/bin`等目录中的非标准命令是不能运行的。

这个问题笔者也遇到很多次，所以很多非标准命令都选择了全路径，但是这个方法也有问题，因为不同环境的命令所存在的目录是不一样的。

#### 解决方案

**方案1：**

在`cron`脚本文件头部声明`PATH`

```
#!/bin/bash
PATH=/opt/someApp/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# rest of script follows
```

**方案2：**

在定时脚本调用的脚本头部声明`PATH`

```
PATH=/opt/someApp/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

15 1 * * * backupscript --incremental /home /root
```


### 因素2：换行符

#### 场景及原因

这个因素就是笔者引子中提到的，官方解释（`man crontab`）如下：

```
Although cron requires that each entry in a crontab end in a newline character, neither the crontab command nor the cron daemon will detect this error. Instead, the crontab will appear to load normally. However, the command will never run. The best choice is to ensure that your crontab has a blank line at the end.

4th Berkeley Distribution      29 December 1993     CRONTAB(1)
```

简单翻译一下就是：

尽管`crontab`要求`cron`中的每个条目都要以换行符结尾，但`crontab`命令和`cron`守护进程都不会检测到这个错误。相反，`crontab`将正常加载。然而，命令永远不会运行。最好的选择是确保您的`crontab`在末尾有一个空白行。

#### 解决方案

给`cron`中每个条目下面添加一个空行

**注意：**

除了没了换行符会导致`cron`中的命令不会运行，即引子中所标识：

```
with no line terminators
```

但是因为非`linux`操作系统导致的非`\n`换行符同样会导致该问题，比如`windows`的`^M`、`mac`的`\r`等

```
with CR line terminators
```

**解决方案：**

`windows`的话就通过`dos2unix`命令转换；而`mac`则可以通过`mac2unix`来转换，`mac2unix`也是`dos2unix`软件中的一部分

#### Refer

* [remove CR line terminators][4]

### 因素3：crond 服务

#### 场景及原因

很多时候`crond`服务未开启，也会导致定时任务不会正常执行。

#### 解决方案

查看服务是否运行，如果未运行，启动`crond`服务即可。

查看方式有两种：

1.通过进程查看

`pgrep`相当于`ps -ef | grep`

```
pgrep cron
```

2.通过`service`查看

```
service crond status
```

启动服务：

```
service crond start
```

### 因素4：shell 解释器

#### 场景及原因

从因素`1`就知道`cron`环境变量中的`SHELL`是`sh`而不是`bash`，我们知道很多`shell`命令是可以在`bash`中正常运行，但是不能在`sh`中运行的，所以这个因素也会影响定时任务的正常运行。

#### 解决方案：

**方案1：**

将`cron`中需要执行的命令在`sh`中执行确认

**方案2：**

将`cron`中需要执行的命令外面加一个`bash shell`的封装：

```
bash -c "mybashcommand"
```

**方案3：**

修改`cron`中的`SHELL`环境变量的值，让所有命令都用`bash`解释器：

```
SHELL=/bin/bash
```

**方案4：**

如果定时任务执行的命令是`shell`脚本，只要在脚本内添加`bash`解释器：

```
#!/bin/bash
```

### 因素5：时区

#### 场景及原因

当修改系统时区后，无论是之前已经存在的`cron`还是之后新创建的`cron`，脚本中设置的定时时间都以旧时区为准，比如原来时区是`Asia/Shanghai`，时间为`10:00`，然后修改时区为`Europe/Paris`，时间变为`3:00`，此时你设置`11:00`的定时时间，`cron`会在`Asia/Shanghai`时区的`11:00`执行。

#### 解决方案：

**方案1：**

重启`crond`服务

```
service crond restart
```

**方案2：**

`kill crond`进程，因为`crond`进程是可重生的

### 因素6：百分号%

#### 场景及原因

当`cron`定时执行命令中，有百分号并且没有转义的时候，`cron`执行会出错，比如执行以下`cron`：

```
0 * * * * echo hello >> ~/cron-logs/hourly/test`date "+%d"`.log
```

会有如下报错：

```
/bin/sh: -c: line 0: unexpected EOF while looking for matching ``'
/bin/sh: -c: line 1: syntax error: unexpected end of file
```

有的日志也会有如下报错：

```
(echo) ERROR (getpwnam() failed)
```

`crontab manpage`中解释：

```
The "sixth" field (the rest of the line) specifies the command to be run. The entire command portion of the line, up to a newline or % character, will be executed by /bin/sh or by the shell specified in the SHELL variable of the cronfile. Percent-signs (%) in the command, unless escaped with backslash (\), will be changed into newline characters, and all data after the first % will be sent to the command as standard input.
```

即`cron`中换行符或`%`前的命令会被`shell`解释器执行，但是`%`会被认为新一行的字符，并且`%`后所有的数据都会以标准输出的形式发送给命令。

#### 解决方案

为百分号做转义，即在`%`前添加反斜杠`\`

#### Refer

* [Cron and Crontab usage and examples][1]
* [How can I execute date inside of a cron tab job?][2]


### 因素7：密码过期

#### 场景及原因

`Linux`下新建用户密码过期时间是从`/etc/login.defs`文件中`PASS_MAX_DAYS`提取的，普通系统默认就是`99999`，而有些安全操作系统是`90`。更改此处，只是让新建的用户默认密码过期时间变化，已有用户密码过期时间仍然不变。

当用户密码过期也会导致`cron`脚本执行失败。

#### 解决方案

将用户密码有效期设置成永久有效期或者延长有效期

**方案1：**

```
chage -M <expire> <username>
```

**方案2：**

```
passwd -x -1 <username>
```

**方案3：**

手动修改`/etc/login.defs`文件中`PASS_MAX_DAYS`的值

### 因素8：权限

#### 场景及原因

很多时候解决方案都是采用`root`用户执行`cron`，但是有时候这并不是一个很好的方式。如果采用非`root`用户执行`cron`，需要注意很多权限问题，比如`cron`用户对操作的文件或目录是否存在权限等。

如果权限不够，`cron`会拒绝执行：

```
sudo service cron restart
grep -i cron /var/log/syslog|tail -2
2013-02-05T03:47:49.283841+01:00 ubuntu cron[49906]: (user) INSECURE MODE (mode 0600 expected) (crontabs/user)
```

#### 解决方案

```
# correct permission
sudo chmod 600 /var/spool/cron/crontabs/user
# signal crond to reload the file
sudo touch /var/spool/cron/crontabs
```

### 因素9：不同平台

#### 场景及原因

一些特殊选项各个平台支持不一样，有的支持，有的不支持，例如`2/3`、`1-5`、`1,3,5`

#### 解决方案

需要针对不同平台做兼容性测试

### 因素10：不同 cron

#### 场景及原因

将之前运行的`Crontab Spec`在从一个`Crontab`文件移动到另一个`Crontab`文件时可能会崩溃。有时候，原因是你已经将`Spec`从系统`crontab`文件转移到用户`crontab`文件，反之亦然。

`cron`分为系统`cron`和用户`cron`，用户`cron`指`/var/spool/cron/username`或`/var/spool/crontabs/crontabs/username`，系统`cron`指
`/etc/crontab`以及`/etc/crontab`，这两者是存在部分差异的。

系统`crontab`在命令行运行之前有一个额外的字段`user`。这会导致一些错误，比如你将`/etc/crontab`中的命令或者`/etc/cron.d`中的文件移动至用户`crontab`会报错如下：
```
george; command not found
```
相反，当发生相反的情况时，`cron`将显示`/usr/bin/restartxyz is not a valid username`之类的错误。

#### 解决方案

当共享系统`cron`或用户`cron`时，注意用户的添加和删除。

### 因素11：crontable 变量

#### 场景及原因

虽然你可以在`crontable`里面声明环境变量，但是在下面这种情况定时任务是不会执行的：

```
SOME_DIR=/var/log
MY_LOG_FILE=${SOME_LOG}/some_file.log

BIN_DIR=/usr/local/bin
MY_EXE=${BIN_DIR}/some_executable_file

0 10 * * * ${MY_EXE} some_param >> ${MY_LOG_FILE}
```

这是因为在`crontable`里面只能声明变量，不能对变量进行操作或者执行其他任何`shell`命令的，所以上述的`shell`字符串拼接是不会成功的，所以只能声明变量，然后在命令中引用变量。

#### 解决方案：

**方案1：** 

直接声明变量

```
SOME_DIR=/var/log
MY_LOG_FILE=/var/log/some_file.log

BIN_DIR=/usr/local/bin
MY_EXE=/usr/local/bin/some_executable_file

0 10 * * * ${MY_EXE} some_param >> ${MY_LOG_FILE}
```

**方案2：**

声明多个变量，在命令中引用拼接

```
SOME_DIR=/var/log
MY_LOG_FILE=some_file.log

BIN_DIR=/usr/local/bin
MY_EXE=some_executable_file

0 10 * * * ${BIN_DIR}/${MY_EXE} some_param >> ${SOME_DIR}/${MY_LOG_FILE}
```

### 因素12：GUI

#### 场景及原因

如果你的`cronjob`调用了相关`GUI`应用时，你需要告诉它们应该使用什么`DISPLAY`环境变量，从因素`1`我们可以知道`cron`中的环境变量是和系统环境变量不一样的，`DISPLAY`同样如此，比如

```
Firefox launch with cron.
```

#### 解决方案

声明`DISPLAY=:0`

```
* * * * * export DISPLAY=:0 && <command>
```

## 总结

目前主要总结了影响`cron`运行的`12`种因素，当然肯定还存在其他影响因素，本文将持续更新，希望这些坑能够被广大开发者所熟知。

大家如果有上述以外导致`cron`不能正常运行的因素可以在博客下方留言，或者在`Github`上面提`pr`，笔者已经将本文在`Github`上面创建了一个仓库，让我们一起不断完善吧 -。-

`Github`仓库地址：https://github.com/tony-yin/Why-Cronjob-Not-Work


[1]: http://www.pantz.org/software/cron/croninfo.html
[2]: https://unix.stackexchange.com/questions/29578/how-can-i-execute-date-inside-of-a-cron-tab-job
[3]: https://askubuntu.com/questions/23009/why-crontab-scripts-are-not-working?page=1&tab=votes#tab-top
[4]: https://stackoverflow.com/questions/14080306/remove-cr-line-terminators
