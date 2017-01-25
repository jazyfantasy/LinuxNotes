##lesson3 进程控制

###进程控制原理
* 定义

> 进程是一个具有一定独立功能的程序的<b>一次运行活动</b>

* 特点

	- 动态性
	- 并发性
	- 独立性
	- 异步性

* 进程三态
	- 就绪
	- 执行
	- 阻塞

* 进程ID

	- 进程ID(PID): 标识进程的唯一数字
	- 父进程的ID(PPID)
	- 启动进程的用户ID(UID)

* 进程互斥

> 当有若干进程使用某一共享资源时, 任何时刻最多允许一个进程使用, 其他要使用该资源的进程必须等待, 直到占用该资源的进程释放了该资源为止.

> 临界资源 - 操作系统中一次只允许一个进程访问的资源

> 临界区 - 进程中访问临界资源的程序代码段

> 为实现对临界资源的互斥访问 应保证诸进程互斥的进入各自的临界区

* 进程同步

> 一组并发进程按一定的顺序执行的过程称为进程间的同步. 具有同步关系的一组并发进程称为合作进程, 合作进程间互相发送的信号称为消息或事件.

* 进程调度

> 按一定算法, 从一组待运行的进程中选出一个来占有CPU运行

调度方式: 1.抢占式; 2.非抢占式;

调度算法: 

1. 先来先服务调度算法; 
2. 短进程优先调度算法;
3. 高优先级优先调度算法;
4. 时间片轮转算法;

* 死锁

> 多个进程因竞争资源而形成一种僵局, 若无外力作用, 这些进程都将永远不能再向前推进.


###进程控制编程
* 获取ID

```
#include <sys/types.h>
#include <unistd.h>

//获取本进程ID
pid_t getpid(void)

//获取父进程ID
pid_t getppid(void)
```

* 进程创建 - fork

```
// 创建子进程
#include <unistd.h>
pid_t fork(void)

return:
1. 在父进程中, fork返回新创建的子进程的PID
2. 在子进程中, fork返回0
3. 出现错误是, fork返回一个负值
```
```
#include <sys/types.h>
#include <unistd.h>
main()
{
  pid_t pid;
  int count = 0;
  // 1 process running.
  pid = fork;
  
  // 子进程的数据空间、堆栈空间都会从父进程得到一个拷贝,而不是贡献。
  count++;
  printf("count = %d", count);
  
  // 2 process running next codes.
  if (pid < 0)
  {
    printf("error in fork!\n");
  }
  else if (pid == 0)
  {
    printf("I'm the chlid process, my ID is %d\n", getpid());
  }
  else
  {
    printf("I'm the parent process, my ID is %d\n", getpid());
  }
}
```

* 进程创建 - vfork
`pid_t vfork(void)`

fork & vfork 区别:

1. fork: 子进程拷贝父进程的数据段,不共享
	vfork: 子进程与父进程共享数据段
2. fork: 父、子进程的执行次序不确定
	vfork: 子进程先运行,父进程后运行

* exec函数族

> 用被执行的程序替换调用它的程序

区别:
fork 创建一个新的进程, 产生一个新的PID
exec 启动一个新的进程, 替换原有的进程, 因此进程的PID不会改变。

```
#include <unistd.h>
int execl(const char* path, const char* arg1,...)
参数:
  path: 被执行程序名(含完整路径)
  arg1 - argn: 程序名及所需的命令行参数, 以空指针结束。
实例:
  execl("/bin/ls", "ls", "-al", "/etc/passwd", NULL);
```
```
#include <unistd.h>
int execlp(const char* path const char* arg1,...)
  path: 被执行程序名(不含路径,将从path环境变量中查找该程序)
```
```
#include <unistd.h>
int execv(const char* path, char* const argv[ ])
	path: 被执行程序名(含完整路径)
	argv[ ]: 被执行程序所需的命令行参数数组
实例:
  char* argv[] = {"ls", "-al", "/etc/passwd", NULL, };
  execv("/bin/ls", argv);
```
```
#include <stdlib.h>
int system(const char* string)
功能:
  调用fork产生子进程, 由子进程来调用/bin/sh -c string 来执行参数string所代表的命令。
实例:
  system("ls -al /etc/passwd");
```

* 进程等待

```
#include <sys/types.h>
#include <sys/wait.h>
// 阻塞该进程, 等待直到其某个子进程退出
pid_t wait (int * status)
```
```
// 进程等待实例
#include <sys/types.h>
#include <sys/wait.h>

#include <unistd.h>
#include <stdlib.h>
void main()
{
  pid_t pc, pr;
  pc = fork();
  
  if (pc == 0)
  {
    printf("This is child process:%d\n", getpid());
    sleep(10);
  }
  else if (pc > 0)
  {
    pr = wait(NULL);
    printf("I catched a child process:%d\n", pr);
  }
  exit(0);
}
```





