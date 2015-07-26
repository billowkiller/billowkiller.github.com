---
layout: post
title: "PID File Analysis"
date: 2015-07-26 16:57
comments: true
categories: linux
tags: [pid, system]
---

# pid 文件浅析

在Linux系统的目录/var/run下面一般我们都会看到很多的*.pid文件。而且往往新安装的程序在运行后也会在/var/run目录下面产生自己的pid文件。那么这些pid文件有什么作用呢？它的内容又是什么呢？

<!--more-->

###pid文件的内容

pid文件为文本文件，内容只有一行, 记录了该进程的ID。
用cat命令可以看到。

### pid文件的作用

防止进程启动多个副本。只有获得pid文件固定路径固定文件名)写入权限(F_WRLCK)的进程才能正常启动并把自身的PID写入该文件中。其它同一个程序的多余进程则自动退出。

### 编程技巧

调用fcntl设置pid文件的锁定F_SETLK状态，其中锁定的标志位F_WRLCK。
如果成功锁定，则写入进程当前PID，进程继续往下执行。
如果锁定不成功，说明已经有同样的进程在运行了，当前进程结束退出。

```c pid usage
lock.l_type = F_WRLCK;
lock.l_whence = SEEK_SET;
if (fcntl(fd, F_SETLK, &lock) < 0){
    //锁定不成功, 退出......
}
sprintf (buf, "%d\n", (int) pid);
pidsize = strlen(buf);
if ((tmp = write (fd, buf, pidsize)) != (int)pidsize){
    //写入不成功, 退出......
}
```

### 一些注意事项：

1. 如果进程退出，则该进程加的锁自动失效。
2. 如果进程关闭了该文件描述符fd， 则加的锁失效。(整个进程运行期间不能关闭此文件描述符)
3. 锁的状态不会被子进程继承。如果进程关闭则锁失效而不管子进程是否在运行。
	(Locks are associated with processes. A process can only have one kind of lock set for each byte of a given file. When any file descriptor for that file is closed by the process, all of the locks that process holds on that file are released, even if the locks were made using other descripors that remain open. Likewise, locks are released when a process exits, and are not inherited by child processes created using fork.)
	
### 其他

**C的fcntl函数：**

	int fcntl(int fd, int cmd, struct flock *lock);

参数cmd代表欲垄断的号召

* F_DUPFD 复制参数fd的文件描写符，厉行获胜则归来新复制的文件描写符，
* F_GETFD 获得close-on-exec符号，若些符号的FD_CLOEXEC位为0，代表在调用exec()相干函数时文件将不会关闭
* F_SETFD 设置close-on-exec符号，该符号以参数arg的 FD_CLOEXEC位定夺
* F_GETFL 获得open()设置的符号
* F_SETFL 改换open()设置的符号
* F_GETLK 获得文件锁定的事态，依据lock的描写，定夺是否上文件锁
* F_SETLK 设置文件锁定的事态，此刻flcok，构造的l_tpye值定然是F_RDLCK、F_WRLCK或F_UNLCK，
万一无法发生锁定，则归来-1
* F_SETLKW 是F_SETLK的阻塞版本，在无法获得锁时会进去睡眠事态，万一能够获得锁可能捉拿到信号则归来

参数lock指针为flock构造指针定义如下

```c flock structure
struct flock {
	...
	short l_type;
	short l_whence;
	off_t l_start; 锁定区域的开关位置
	off_t l_len; 锁定区域的大小
	pid_t l_pid; 锁定动作的历程
	...
};
```

1_type有三种事态：

* F_RDLCK读取锁（分享锁）
* F_WRLCK写入锁（排斥锁）
* F_UNLCK解锁

l_whence也有三种措施

* SEEK_SET以文件开始为锁定的起始位置
* SEEK_CUR以现在文件读写位置为锁定的起始位置
* SEEK_END以文件尾为锁定的起始位置

**Python用法:**

```python test_fcntl.py
import signal, fcntl, os, time

def signal_handler(signum, frame):
    fcntl.flock(pid_file, fcntl.LOCK_UN)
    pid_file.close()
    os.unlink(pid_path)
    sys.exit(0)

if __name__ == '__main__':
    pid_path = 'test.pid'
    pid_file = open(pid_path, "w")
    fcntl.flock(pid_file, fcntl.LOCK_EX | fcntl.LOCK_NB)
    pid_file.write("%d" % os.getpid())
    signal.signal(signal.SIGTERM, signal_handler)
    while(True):
        print "Hello World"
        time.sleep(1)
```
