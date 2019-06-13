
# 什么是PID 1

> 在Linux操作系统中，当内核初始化完毕之后，会启动一个init进程，这个进程是整个操作系统的第一个用户进程，所以它的进程ID为1，也就是我们常说的PID1进程。在这之后，所有的用户态进程都是该进程的后代进程，由此我们可以看出，整个系统的用户进程，是一棵由init进程作为根的进程树。

> init进程有一个非常厉害的地方，就是SIGKILL信号对它无效。很显然，如果我们将一棵树的树根砍了，那么这棵树就会分解成很多棵子树，这样的最终结果是导致整个操作系统进程杂乱无章，无法管理。

> PID 1进程的发展也是一段非常有趣的过程，从最早的sysvinit,到upstart,再到systemd。其中systemd还在linux社区引起了不小的争议，systemd作者Lennart还在 Google Plus 上发了贴子，喜欢八卦的同学可以前往一读。

> 那么这个PID 1进程在操作系统的整个生命周期中，到底起了什么重要的作用呢？首先我们先来了解以下几个概念：

## 进程表项

> linux内核程序通过进程表对进程进行管理, 每个进程在进程表中占有一项，称为进程表项，它记录了进程的状态，打开的文件描述符等等一系统信息。当一个进程结束了运行或在半途中终止了运行，那么内核就需要释放该进程所占用的系统资源。这包括进程运行时打开的文件，申请的内存等。但是，这里要注意的是，进程表项并没有随着进程的退出而被清除，它会一直占用内核的内存。为什么会有这么奇怪的行为呢？这是因为在某些程序中，我们必须明确地知道进程的退出状态等信息，而这些信息的获取是由父进程调用wait/waitpid而获取的。设想这样一种场景，如果子进程在退出的时候直接清除文件表项的话，那么父进程就很可能没有地方获取进程的退出状态了，因此操作系统就会将文件表项一直保留至wait/waitpid系统调用结束。

## 僵尸进程

> 僵尸进程指的是：进程退出后，到其父进程还未对其调用wait/waitpid之间的这段时间所处的状态。一般来说，这种状态持续的时间很短，所以我们一般很难在系统中捕捉到。但是，一些粗心的程序员可能会忘记调用wait/waitpid，或者由于某种原因未执行该调用等等，那么这个时候就会出现长期驻留的僵尸进程了。如果大量的产生僵尸进程，其进程号就会一直被占用，可能导致系统不能产生新的进程。

> 聪明的读者可能立马会想到一种情况，就是如果父进程先于子进结束，那么是不是就没有人负责这个子进程的资源清理工作了，那我们的系统岂不是到处都是僵尸进程?事实上操作系统设计人员早就想到了这个问题，这也是我们的PID 1进程最重要的职责。

# 容器中的孤儿进程

## 容器中的PID 1

> 熟悉Docker同学可能知道，容器并不是一个完整的操作系统，它也没有什么内核初始化过程，更没有像init(1)这样的初始化过程。在容器中被标志为PID 1的进程实际上就是一个普普通通的用户进程，也就是我们制作镜像时在Dockerfile中指定的ENTRYPOINT的那个进程。而这个进程在宿主机上有一个普普通通的进程ID，而在容器中之所以变成PID 1，是因为linux内核提供的PID namespaces功能，如果宿主机的所有用户进程构成了一个完整的树型结构，那么PID namespaces实际上就是将这个ENTRYPOINT进程（包括它的后代进程）从这棵大树剪下来，很显然，剪下来的这部分东西本身也是一个树型结构，它完全可以自己长成一棵苍天大树（不断地fork）,当然，子树里面是看不到整棵树的原貌的，但是在子树外面确可以看到完整的子树。
比如我们在宿主机查看某个tomcat容器：
```c
$ docker top 7bb975e9a7cb
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
8080                56128               56100               0                   15:52               ?                   00:00:00            /bin/bash /home/tomcat/start.sh
8080                56178               56128               2                   15:52               ?                   00:02:26            /usr/bin/java ......
```
```c
$ cat /proc/56128/status | grep NSpid
NSpid:  56128   1
```
```c
$ cat /proc/56178/status | grep NSpid
NSpid:  56178   17
```

> 可以看到同一个进程在容器内外的进程号是不同的。我们如果在容器外部kiss -9 56128,那整个容器便会处于退出状态。
我们现在进入容器：
```c
$ docker exec -it 7bb975e9a7cb bash
$  ps -ef
UID         PID   PPID  C STIME TTY          TIME CMD
tomcat        1      0  0 15:52 ?        00:00:00 /bin/bash /home/tomcat/start.sh
tomcat       17      1  2 15:52 ?        00:02:30 /usr/bin/java ....
tomcat      119      0  0 17:24 pts/0    00:00:00 bash
tomcat      126    119  0 17:25 pts/0    00:00:00 ps -ef
```

> 可以看到bash进程的父进程号是0，和PID 1进程处于同一层级上。接下来我们打算在容器当中造个孤儿进程出来。
```c
#parent.sh
bash ./child.sh
```
```c
#child.sh
while true
do
    sleep 10
done
```
```c
bash ./parent.sh
```

> 在另一个终端中运行
```c
$ ddocker exec -it 7bb975e9a7cb ps xf -o pid,ppid,stat,args
PID   PPID STAT COMMAND
201      0 Rs+  ps xf -o pid,ppid,stat,args
119      0 Ss   bash
198    119 S+    \_ bash ./parent.sh
199    198 S+        \_ bash ./child.sh
200    199 S+            \_ sleep 10
  1      0 Ss   /bin/bash /home/tomcat/start.sh
17      1 Sl   /usr/bin/java ......
```

> 接下来用kill -9杀死parent进程
```c
$ docker exec -it 7bb975e9a7cb kill -9 198
```
```c
$ docker exec -it 7bb975e9a7cb ps xf -o pid,ppid,stat,args
PID   PPID STAT COMMAND
222      0 Rs+  ps xf -o pid,ppid,stat,args
199      0 S    bash ./child.sh
214    199 S     \_ sleep 10
119      0 Ss+  bash
  1      0 Ss   /bin/bash /home/tomcat/start.sh
17      1 Sl   /usr/bin/java ......
```

> 可以看到，child进程的父进程变成了PID 0，那么这个PID 0又是何方神圣，为什么它可以接管孤儿进程，又为何ENTRYPOINT进程的父进程也是它。

## 容器中的PID 0

> 我们在前面提到过，容器中的进程树实际上是宿主机进程树的一棵子树，那么我们在宿主机上是否就可以找到这棵子树的父进程呢？我们在宿主机上执行以下命令
```c
$ ps -axf | grep -C 5 56128
......
56100 ?        Sl     0:00      \_ docker-containerd-shim -namespace moby -workdir /data/var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/7bb975e9a7cbf84a17b584c0594c854283e47116cb4fd7eaecec8e4c706e363f -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc -systemd-cgroup
56128 ?        Ss     0:00          \_ /bin/bash /home/tomcat/start.sh
56178 ?        Sl     2:47          |   \_ /usr/bin/java ......
```

> 至此，我们可以大胆的猜想，这个PID 0应该就是这个docker-containerd-shim

## Docker 1.11版本后的架构

![Docker](http://shareinto.k8s.101.com/image/docker-arch.jpg)
