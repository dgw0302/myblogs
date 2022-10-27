---
abbrlink: 0
---


# fork命令

**fork调用的一个奇妙之处就是它仅仅被调用一次，却能够返回两次，它可能有三种不同的返回值：**
  1）在父进程中，fork返回新创建子进程的进程ID；
  2）在子进程中，fork返回0；
  3）如果出现错误，fork返回一个负值

![image-20220309160809634](../../images/Linux/image-20220309160809634.png)





![image-20220309160842092](../../images/Linux/image-20220309160842092.png)1、[fork](https://so.csdn.net/so/search?q=fork&spm=1001.2101.3001.7020)()函数：pid_t 

# fork与kill

fork(void);

返回值：fork仅仅被调用一次，却能够返回两次，它可能有三种不同的返回值
      (1)在父进程中，fork返回新创建子进程的进程ID；
      (2)在子进程中，fork返回0；
      (3)如果出现错误，fork返回一个负值；
      在fork[函数](https://so.csdn.net/so/search?q=函数&spm=1001.2101.3001.7020)执行完毕后，如果创建新进程成功，则出现两个进程，一个是子进程，一个是父进程。在子进程中，fork函数返回0，在父进程中，fork返回新创建子进程的进程ID。

2、kill()函数：int kill(pid_t pid, int sig);

函数参数：①pid：指定进程的进程ID，注意用户的权限，比如普通用户不可以杀死1号进程（init）。

​        pid>0：发送信号给指定进程

​        pid=0：发送信号给与调用kill函数进程属于同一进程组的所有进程

​        pid<0：发送信号给pid绝对值对应的进程组

​        pid=-1：发送给进程有权限发送的系统中的所有进程

​         ②信号量：本实验用SIGTERAM。程序结束(terminate)信号，和SIGKILL不同的是该信号可以被阻塞和处理，通常用来要求程序自己退出。如果终止不了，我们才会尝试SIGKILL。







vim



linux创建文件的命令





linux创建文件目录的命令

























