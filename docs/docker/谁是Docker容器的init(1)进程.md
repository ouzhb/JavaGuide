

<!-- MarkdownTOC -->

- [什么是PID 1](#什么是PID1)
  - [进程表项](#进程表项)
  - [僵尸进程](#僵尸进程)
- [容器中的孤儿进程](#容器中的孤儿进程)
  - [容器中的PID 1](#容器中的PID1)
  - [容器中的PID 0](#容器中的PID0)
  - [Docker 1.11版本后的架构](#Docker1.11版本后的架构)
  - [runC](#runC)
  - [PR_SET_CHILD_SUBREAPER](#PR_SET_CHILD_SUBREAPER)
  - [1.11以前的Docker](#1.11以前的Docker)

<!-- /MarkdownTOC -->

# 什么是PID1

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

## 容器中的PID1

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

## 容器中的PID0

> 我们在前面提到过，容器中的进程树实际上是宿主机进程树的一棵子树，那么我们在宿主机上是否就可以找到这棵子树的父进程呢？我们在宿主机上执行以下命令
```c
$ ps -axf | grep -C 5 56128
......
56100 ?        Sl     0:00      \_ docker-containerd-shim -namespace moby -workdir /data/var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/7bb975e9a7cbf84a17b584c0594c854283e47116cb4fd7eaecec8e4c706e363f -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc -systemd-cgroup
56128 ?        Ss     0:00          \_ /bin/bash /home/tomcat/start.sh
56178 ?        Sl     2:47          |   \_ /usr/bin/java ......
```

> 至此，我们可以大胆的猜想，这个PID 0应该就是这个docker-containerd-shim

## Docker1.11版本后的架构

![Docker 1.11版本后的架构](http://shareinto.k8s.101.com/image/docker-arch.jpg)

> 从架构图中我们可以看到shim进程下还有一个runC进程，但我们在进程树中并没有发现runC这个进程。

## runC

> runC是OCI标准的一个参考实现，而OCI Open Container Initiative，是由多家公司共同成立的项目，并由linux基金会进行管理，致力于container runtime的标准的制定和runc的开发等工作。runc，是对于OCI标准的一个参考实现，是一个可以用于创建和运行容器的CLI(command-line interface)工具。runc直接与容器所依赖的cgroup/linux kernel等进行交互，负责为容器配置cgroup/namespace等启动容器所需的环境，创建启动容器的相关进程。

> 事实上，Docker容器的创建过程是这样子的 docker-containerd-shim –> runC –> entrypoint，而我们看到的最终状态是 docker-containerd-shim –> entrypoint，聪明的你可能已经猜到，runc进程创建完容器之后，自己就先退出去了。但是这里面其实暗藏了一个问题，按照前面提到的孤儿进程理论，entrypint进程应该由操作系统的PID 1进程接管，但为什么会被shim接管呢？

## PR_SET_CHILD_SUBREAPER

> linux在内核3.14以后版本支持该系统调用，它可以将调用进程标记“child subreaper”属性，而拥有该属性的进程则可以充当init(1)进程的功能，收养其后代进程中所产生的孤儿进程。我们可以从shim的源码中找到答案
```c
func start(log *os.File) error {
     // start handling signals as soon as possible so that things are properly reaped
     // or if runtime exits before we hit the handler
     signals := make(chan os.Signal, 2048)
     signal.Notify(signals)
     // set the shim as the subreaper for all orphaned processes created by the container
     if err := osutils.SetSubreaper(1); err != nil {
         return err
     }
     ...
 }
```

> 既然充当了reaper的角色，那么就应该尽到回收资源的责任：
```c
func start(log *os.File) error {
    ...
    switch s {
        case syscall.SIGCHLD:
            exits, _ := osutils.Reap(false)
            ...
    }
    ...
}
```

```c
func Reap(wait bool) (exits []Exit, err error) {
    ...
    
    for {
        pid, err := syscall.Wait4(-1, &ws, flag, &rus)
        if err != nil {
            if err == syscall.ECHILD {
                return exits, nil
            }
            return exits, err
        }
        
        ...
    }
}
```
> 从这里我们可以看到shim的wait/waitpid系统调用。

## 1.11以前的Docker

> 实际上在早期的Docker中，并没有reaper的设置，那么内核此时会如何处理孤儿进程呢？
```c
/*
 * When we die, we re-parent all our children, and try to:
 * 1. give them to another thread in our thread group, if such a member exists
 * 2. give it to the first ancestor process which prctl'd itself as a
 *    child_subreaper for its children (like a service manager)
 * 3. give it to the init process (PID 1) in our pid namespace
 */
static struct task_struct *find_new_reaper(struct task_struct *father,
                       struct task_struct *child_reaper)
{
    struct task_struct *thread, *reaper;

    thread = find_alive_thread(father);
    if (thread)
        return thread;

    if (father->signal->has_child_subreaper) {
        /*
         * Find the first ->is_child_subreaper ancestor in our pid_ns.
         * We start from father to ensure we can not look into another
         * namespace, this is safe because all its threads are dead.
         */
        for (reaper = father;
             !same_thread_group(reaper, child_reaper);
             reaper = reaper->real_parent) {
            /* call_usermodehelper() descendants need this check */
            if (reaper == &init_task)
                break;
            if (!reaper->signal->is_child_subreaper)
                continue;
            thread = find_alive_thread(reaper);
            if (thread)
                return thread;
        }
    }

    return child_reaper;
}
```

> * 找到相同线程组里其它可用线程
> * 沿着它的进程树向祖先进程找一个最近的child_subreaper并且运行着的进程
> * 该namespace下进程号为1的进程

> 很显然，旧版本的Docker在容器内所产生的孤儿进程，会被进程号为1（也就是entrypoint进程）所接管，而我们知道，该进程一般是一个普通的应用程序，一般不会特意去实现对孤儿进程的处理，所以，在使用早期版本的Docker时，我们会发现操作系统会经常出现僵尸进程。所以，为了你的系统稳定，早点升级你的内核和Docker的版本吧。
