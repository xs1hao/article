---
name: 《Linux命令行》学习笔记（十四）
title: 《Linux命令行》学习笔记（十四）
tags: ["技术","学习笔记","Linux命令行与shell脚本编程大全"]
categories: 学习笔记
info: "第16章 控制脚本"
time: 2020/6/16
desc: 'Linux命令行与shell脚本编程大全, 资料下载, 学习笔记, 第16章 控制脚本'
keywords: ['Linux命令行与shell脚本编程大全', 'shell学习', '学习笔记', '第16章 控制脚本']
---

# 《Linux命令行》学习笔记（十四）

## 第16章 控制脚本

> 本章内容：
>
> - 处理信号
> - 以后台模式运行脚本
> - 禁止挂起
> - 作业控制
> - 修改脚本优先级
> - 脚本执行自动

### 16.1 处理信号

Linux 可以利用信号与运行在系统中的进程进行通信。可以通过对脚本进行编程，使其在收到特定信号时执行某些命令从而控制 shell 脚本的操作。

#### 16.1.1 重温 Linux 信号

Linux 有以下信号。在脚本中捕获时要在其名称前加上 SIG 前缀。

![linux-signal-1.jpg](./images/linux-signal-1.jpg)

默认情况下，bash shell 会忽略收到的 SIGQUIT（3）信号和 SIGTERM（15）信号，而会处理收到的 SIGHUP（1）信号和 SIGINT（2）信号。

> 通过SIGINT信号，可以中断shell。Linux内核会停止为shell分配CPU处理时间。这种情况发生时，shell会将SIGINT信号传给所有由它所启动的进程，以此告知出现的状况。
>
> 而shell脚本的默认行为 是忽略这些信号。它们可能会不利于脚本的运行。要避免这种情况，你可以脚本中加入识别信号的代码，并执行命令来处理信号

#### 16.1.2 生成信号

bash shell 可以使用 Ctrl 键和键盘上的某些特定按键生成两种基本的 Linux 信号。

**1. 中断进程**

Ctrl+C 组合键会发送 INT（2） 信号，停止 shell 中当前运行的进程。比如`sleep`命令会使得 shell 暂停指定的秒数，而在此之前按下 Ctrl+C 组合键，就可以提前终止进程。

**2. 暂停进程**

你可以使用 Ctrl+Z 组合键发送一个 TSTP（18） 信号，使其在进程运行期间暂停进程而无需终止它。尽管有时这样会比较危险，但它可以让你在不终止进程的情况下使你能够深入脚本内部一窥究竟。

输入 Ctrl+Z 之后，shell 会通知进程已停止。

```shell
sleep 100
# ^Z
# [1]+  已停止               sleep 100
```

方括号中的数字是 shell 分配的作业号，它会将第一个作业分配给作业号 1，第二个作业号 2，以此类推。

如果你的 shell 会话中已有一个已停止的作业，在退出 shell 时，bash 会提醒你有登出的任务。

```shell
exit
# 登出
# 有停止的任务。
```

当然如果你已经知道了已停止作业的 PID，就可以用`kill`命令来发送一个 KILL 信号来终止它。

```shell
kill -9 PID
# [1]+  Killed                  sleep 100
```

shell 会显示一条消息，说明作业在运行时被终止了。

此外，还可以配合管道符和后面学习到的字符处理工具`awk`来组合成方便的批量搜索并删除进程小脚本。

```shell
# 批量删除任意指定的进程
ps aux | grep 进程名称 | grep -v grep | awk '{print $2}' | xargs kill -9 &> /dev/null
```

#### 16.1.3 捕获信号

> 也可以不忽略信号，在信号出现时捕获它们并执行其他命令。

`trap`命令允许你来指定 shell 脚本并从 shell 中拦截的 Linux 信号。该命令的格式是：`trap commands signals`

```shell
#!/bin/bash
trap "echo '拦截 Ctrl+C 操作'" SIGINT
echo 测试脚本
count=1
while [ $count -le 10 ]
do
	echo "Loop #$count"
	sleep 1
	count=$[ $count + 1 ]
done
echo "脚本结束"
```

如上面的脚本，`trap`命令会在每次检测到信号时显示一行文本消息，同时阻止用户停止程序，而不是处理该信号并允许shell 停止该脚本。

#### 16.1.4 捕获脚本退出

shell 脚本中不仅可以捕获系统信号，还可以捕获 EXIT 信号。当脚本运行到正常的退出位置时，就会触发 EXIT 信号，而如果提前退出也同样能够捕获到。

```shell
#!/bin/bash
trap "echo byebye" EXIT
echo "执行"
```

> 当按下 Ctrl+C 组合键发送 SIGINT 信号时，脚本也会退出。但在脚本退出前捕获到了EXIT，于是 shell 也会执行 trap 命令

#### 16.1.5 修改或移除捕获

要想在脚本中的不同位置进行不同的捕获处理，只需重新使用带有新选项的 trap 命令。

也就是说，当触发信号时，程序会在已经执行过的程序中自动寻找最后一次出现的 trap 命令以触发。

```shell
trap "echo 无效trap" EXIT
trap "echo 有效trap" EXIT
echo "hi~"
```

也可以删除已设置好的捕获，只需要在`trap`命令与希望恢复默认行为的信号列表之间加个破折号就行了。

```shell
trap "echo 'wow, you can really dance'" EXIT
# 删除捕获
trap - EXIT
echo "测试~"
```

### 16.2 以后台模式运行脚本

> 以后台模式运行shell脚本非常简单。只要在命令后加个&符就行了

```shell
./test.sh &
```

当 & 符放到命令后时，它会将命令和 bash shell 分离开，并将命令作为系统中的一个独立后台进程运行。

注意当后台进程运行时，它仍然会使用显示器来显示 STDERR 和 STDOUT 的消息。

要避免后台的命令和自己要做的事情输出混在一起这种情况，最好还是在脚本中做好进行重定向。

#### 16.2.3 在非控制台下运行脚本

使用`nohup`命令可以让脚本一直以后台模式运行到结束。

`nohup`命令会阻断所有发送给该进程的 SIGHUP 信号，也能够在退出终端时阻止进程退出。

```shell
nohup ./test.sh &
```

> 由于 nohup 命令会解除终端与进程的关联，进程也就不再同 STDOUT 和 STDERR 联系在一起。 
>
> 如果使用 nohup 运行了另一个命令，该命令的输出会被追加到已有的 nohup.out 文件中。当 运行位于同一个目录中的多个命令时一定要当心，因为所有的输出都会被发送到同一个 nohup.out 文件中，结果会让人摸不清头脑。

nohup.out 文件通常包含了所有发送到终端显示器上的输出。

### 16.4 作业控制

> 启动、停止、终止以及恢复作业的这些功能统称为作业控制。通过作业控制，就能完全控制 shell 环境中所有进程的运行方式了

使用 CTRL+Z 停止正在运行的作业后，你可以选择用`kill`命令终止该进程，如果要重启则需要向其发送一个 SIGCONT 信号。

#### 16.4.1 查看作业

`jobs`命令是查看作业的关键命令，允许查看 shell 当前正在处理的作业。

```shell
#!/bin/bash
echo "程序 PID 为：$$"
count=1
while [ $count -le 10 ]
do
	echo "循环数$count"
	sleep 10
	count=$[ $count + 1 ]
done
echo "结束"
```

可以从命令行中启动上面的脚本，然后使用 Ctrl+Z 来停止脚本。

分别执行两次该脚本，然后用`jobs`命令来查看已有的作业，还可以加入下列参数：

![linux-jobs-1.png](./images/linux-jobs-1.png)

```shell
jobs -l
# [1]  + 97366 suspended  sh test.XXXX.sh
# [2]  - 97649 running    sh test.XXXX.sh
```

上面输出中，1 跟 2 代表作业号，其中 + 号的作业会被当成默认作业，带减号的作业将成为下一个默认作业，任何时候都只有一个 + 号和一个 - 号作业。

如果使用`kill`命令提前终止默认作业，那么减号作业就会变成默认作业，减号也会变成加号。

#### 16.4.2 重启停止的作业

使用`bg`命令加上一个指定的作业号可以将后台进程或者前台进程重启。

> 前台进程会接管你 当前工作的终端，所以在使用该功能时要小心了。

如果不加作业号，bg 命令就会重启默认作业。

```shell
sh test.XXXX.sh
# 程序 PID 为：97023
# 循环数: 1
# 循环数: 2
# 循环数: 3
^Z
# [1]  + 99176 suspended  sh test.XXXX.sh
bg 1
# [1]  + 99176 continued  sh test.XXXX.sh
# 循环数: 4
# 循环数: 5
# 循环数: 6
# 循环数: 7
# 循环数: 8
# 循环数: 9
# 循环数: 10
# 结束

# [1]  + 99176 done       sh test.XXXX.sh
```

如果要将一个后台模式的作业重新以前台模式启动，可以使用`fg`命令，其用法与`bg`命令一致。

### 16.5 调整谦让度

在 Linux 等多任务操作系统中，内核负责将 CPU 时间分配给系统上运行的每个进程。而**调整优先级（scheduling priority）**是内核分配给进程的 CPU 时间。在 Linux 中，由 shell 启动的所有进程的调度优先级默认都是相同的。

> 调度优先级是个整数值，从 -20（最高优先级）到 +19（最低优先级）。默认情况下，bash shell 以优先级0来启动所有进程。 
>
> 最低值 -20 是最高优先级，而最高值 19 是最低优先级，这太容易记混了。只要记住那句俗 语“好人难做”就行了。越是“好”或高的值，获得CPU时间的机会越低。 

#### 16.5.1 nice 命令

`nice`命令可以设置命令启动时的优先级，使用`nice -n`的方式即可以指定新的优先级级别，如下：

```shell
nice -n 10 ./test.sh > test.out &
# 也可以不用 n，只要 - 后面加数字就可以
nice -10 ./test.sh > test.out &
```

但要注意的是，**在非 root 模式下**，`nice`命令只能让脚本以更低的优先级运行，而无法提高某个命令的优先级。

```shell
 nice -n -10 ./test4.sh > test4.out &
 # [1] 4985
 # nice: cannot set niceness: Permission denied 
```

#### 16.5.2 renice 命令

`renice`命令可以指定运行进程的 PID 来改变已运行命令的优先级。

```shell
sh test.fJ78.sh &
[1] 10034
jobs -l
# [1]+ 10044 运行中 sh test.fJ78.sh &
renice -n 10 -p 10044
ps -p 10044 -o pid,ppid,ni,cmd
# 10044  4721  10 /bin/bash sh test.fJ78.sh &
```

和`nice`命令一样，`renice`命令也有下列限制：

- 只能对属于自己的进程执行
- 只能降低进程优先级
- 如果是 root 用户，则可以任意调整任意进程的优先级

如果想完全控制运行进程，必须以root账户身份登录或使用 sudo 命令。

### 16.6 定时运行作业

Linux 提供了多个在预选事件运行脚本的方法：`at`命令和`cron`表。

#### 16.6.1 用 at 命令来计划执行作业（定时闹钟）

> at 命令会将作业提交到队列中，指定 shell 何时运 行该作业。at 的守护进程 atd 会以后台模式运行，检查作业队列来运行作业。大多数 Linux 发行版会在启动时运行此守护进程。
>
> atd 守护进程会检查系统上的一个特殊目录（通常位于/var/spool/at）来获取用 at 命令提交的作业。默认情况下，atd 守护进程会每 60 秒检查一下这个目录。有作业时，atd 守护进程会检查作业设置运行的时间。如果时间跟当前时间匹配，atd守护进程就会运行此作业。

**1. at 命令的格式**

`at`命令的基本格式为：`at [-f filename] time`

默认情况下，`at`命令会将 STDIN 的输入放到队列中。你也可以用`-f`来指定脚本文件。

time 参数指定了 Linux 系统什么时候运行作业，`at`命令能识别多种不同的时间格式，如下：

- 标准小时和分钟格式，如 10:15
- 12 小时制指示符，如 10:15 PM
- 特定可命名时间，如：now、noon、midnight 或者 teatime（下午四点）

除此以外还可以指定特定日期：

- 标准日期格式：MMDDYY、MM/DD/YY、DD.MM.YY，完整例子如：2015-07-14 12:48
- 文本日期，如 Jul4 或 Dec25，完整例子如： July14,12:48:48
- 时间增量，如 +25min

使用`at`命令提交的作业会提交到`作业队列`中等待执行，作业队列有 26 个不同的优先级，使用 a~z 26 个英文字母区分，字母排序越高，作业运行的优先值越低，相当于更高的 nice 值。默认情况下的`at`命令会被提交到 a 作业队列，可以使用`-q`参数指定不同的队列字母。

**2. 获取作业输出**

`at`命令默认会将提交该作业的用户的电子邮件地址作为 STDOUT 和 STDERR，将输出通过邮件系统发送给该用户。

> 使用e-mail作为at命令的输出极其不便。at命令利用sendmail应用程序来发送邮件。如 果你的系统中没有安装sendmail，那就无法获得任何输出！因此在使用at命令时，最好在脚本中对 STDOUT 和 STDERR 进行重定向。

如果不想在`at`命令中使用邮件或者重定向，也可以加上`-M`选项来屏蔽产生的输出消息。

**3. 列出等待的作业**

使用`atq`命令可以查看系统中的哪些作业在等待中。

```shell
at -M -f test.sh teatime
atq
```

**3. 删除作业**

使用`atrm [指定的作业号]`命令可以删除指定的等待中的作业。

#### 16.6.2 安排需要定期执行的脚本（循环定时）

> 如果你需要脚本在每天的同一时间运行或是每周一次、每月一次呢？

Linux 系统使用 cron 程序来安排要定期执行的作业。

cron 通过检查一个特殊的时间表（cron 时间表）来获知已安排的作业。

使用`crontab`命令来处理时间表，要列出已有的 cron 时间表，可以用`-l`选项，此外可以用`-e`选项来为时间表添加条目，亦或者直接在执行命令时使用如下格式：

```shell
# 允许用特定值、取值范围（比如1~5）或者是通配符（星号）来指定条目
# 可以用三字符的文本值（mon、tue、wed、thu、fri、sat、sun）或数值（0为周日，6为周六）来指定 dayofweek 表项
# min hour dayofmonth month dayofweek command
15 10 * * * /home/mine/test.sh > testout
# 由于无法设置 dayofmonth 的值来涵盖所有的月份，可能有人会问如何设置一个在每个月的最后一天执行的命令，一般来说可以加一条使用 date 命令的 if-then 语句来检查明天的日期是不是01
00 12 * * * if [`date +%d -d tomorrow` = 01 ] ; then ; command 
```

> 如果你创建的脚本对精确的执行时间要求不高，用预配置的cron脚本目录会更方便。有4个 基本目录：hourly、daily、monthly和weekly。
>
> 如果脚本需要每天运行一次，只要将脚本复制到daily目录，cron就会每天执行它。 
>
> ```shell
> ls /etc/cron.*ly
> #/etc/cron.daily:
> #google-chrome  logrotate  man-db.cron
> 
> #/etc/cron.hourly:
> #0anacron
> 
> #/etc/cron.monthly:
> 
> #/etc/cron.weekly:
> ```

