##lesson5 多线程

###线程理论基础

* 与进程相比, 线程的优点:
	1. 和进程相比, 线程是一种非常"节俭"的多任务操作方式. 一个进程中的多个线程, 使用相同的地址空间, 线程间切换时间也远小于进程间切换.
	2. 线程间的通讯机制更快捷方便. 同一进程下的线程之间共享数据空间,所以一个线程的数据可直接为其他线程所用.
	3. 使多CPU系统更有效率. OS能保证当线程数不大于CPU数目时,不同的线程运行于不同的CPU.
	4. 改善程序结构. 一个长而复杂的进程可分为多个线程, 成为几个独立或半独立的运行部分(模块化).

* Linux多线程

> linux系统下的多线程遵循POSIX线程接口, 称为pthread. 编写Linux下的多线程程序, 需要使用头文件pthread.h, 连接时需要使用库libpthread.a。

###多线程程序设计
* 创建线程

```
#include <pthread.h>
int pthread_creat(
  // tidp: 线程id
  pthread_t* tidp,
  
  // attr: 线程属性(通常为空)
  const pthread_attr_t* attr,
  
  // start_rtn: 线程要执行的函数
  void *(*start_rtn)(void),
  
  // arg: start_rtn的参数
  void* arg)
```

* 编译

> 因为pthread的库不是linux系统的库, 所以编译时加上`-lpthread`

exp:`gcc filename -lpthread -o outputfile`

```
// --- thread_create.c ---
// 实例: 线程创建
#include <stdio.h>
#include <pthread.h>

void *myThread1(void)
{
  for (int i=0; i<100; i++)
  {
    printf("This is the 1st pthread, created by jazy.\n");
    sleep(1);
  }
}

void *myThread2(void)
{
  for (int i=0; i<100; i++)
  {
    printf("This is the 2st pthread, created by jazy.\n");
    sleep(1);
  }
}

int main()
{
  int i=0, ret=0;
  pthread_t id1, id2;
  
  ret = pthread_create(&id1, NULL, (void*)myThread1, NULL);
  if (ret)
  {
    printf("Create pthread1 error!\n");
    return 1;
  }
  
  ret = pthread_create(&id2, NULL, (void*)myThread2, NULL);
  if (ret)
  {
    printf("Create pthread2 error!\n");
    return 1;
  }
  
  pthread_join(id1, NULL);
  pthread_join(id2, NULL);
  
  return 0;
}
// --- thread_create.c ---
```
```
// --- thread_int.c ---
// 实例: 线程函数的参数传递
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void *create(void* arg)
{
  int* num = (int*)arg;
  printf("Create parameter is %d.\n", *num);
  return (void*)0;
}

int main()
{
  int ret = 0;
  
  pthread_t tidp;
  int test = 4;
  int* attr = &test;
  
  ret = pthread_create(&tidp, NULL, (void*) create, (void*)attr);
  if (ret)
  {
    printf("Create pthread error ...\n");
    return -1;
  }
  
  sleep(1);
  printf("pthread is created ...\n");

  return 0;
}
// --- thread_int.c ---
```
```
// --- thread_share.c ---
// 实例: 线程内存共享
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int a = 1;
void *create(void* arg)
{
  printf("new pthread ...  \n");
  printf("[pthread] a = %d.\n", a++);
  return (void*)0;
}

int main(int argc, char* argv[])
{
  int ret = 0;
  
  pthread_t tidp;
  
  printf("[main] a = %d.\n", a);
  
  ret = pthread_create(&tidp, NULL, (void*) create, NULL);
  if (ret != 0)
  {
    printf("Create pthread error ...\n");
    return -1;
  }
  
  sleep(1);
  printf("new pthread is created ...\n");
  printf("[main] a = %d.\n", a);

  return 0;
}
// --- thread_share.c ---
```

* 线程终止 - 线程的正常退出方式有:
	1. 线程从启动例程中返回.
	2. 线程可以被另一个进程终止.
	3. 线程自己调用pthread_exit函数.

> 进程中任何一个线程中调用exit/_exit, 那么整个进程都会终止, (而不是只是该线程终止).

```
// 功能: 终止调用线程
// rval_ptr: 线程退出返回值的指针.
#include <phtread.h>
void pthread_exit(void* rval_ptr)
```
```
// --- thread_exit.c ---
// 实例: 线程退出
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void *create(void* arg)
{
  printf("[thread]new pthread is created ...  \n");
  return (void*)8;
}

int main(int argc, char* argv[])
{
  int ret = 0;
  
  pthread_t tid;
  void* temp;
  printf("[main] init ...\n");
  
  ret = pthread_create(&tid, NULL, create, NULL);
  if (ret != 0)
  {
    printf("[main] pthread creat error ...\n");
    return -1;
  }
  
  ret = pthread_join(tid, &temp);
  if (ret)
  {
    printf("[main] thread is not exit ...\n");
    return -2;
  }
  printf("[main] pthread exit code is %d.\n", (int)temp);

  return 0;
}
// --- thread_exit.c ---
```

* 线程等待

```
// 功能: 阻塞调用线程, 直到指定的线程终止.
// tid: 等待退出的线程id
// rval_ptr: 线程退出的返回值的指针
#include <pthread.h>
int pthread_join(pthread_t tid, void **rval_ptr)
```

* 线程标识

```
// 功能: 获取调用线程的thread identifier
#include <pthread.h>
pthread_t pthread_self(void)
```

* 线程清除 
	- 线程终止有2种情况:
		1. 正常终止(可预见): pthread_ext/return
		2. 非正常终止(不可预见): 在其他线程的干预下/由于自身运行出错(如访问了非法地址)而退出

> 无论线程是正常终止还是非正常终止, 都必须保证线程终止时能顺利释放掉自己所占用的资源.
> 
> 从`pthread_cleanup_push`的调用点到`pthread_cleanup_pop`之间的程序段中的终止动作(包括调用`pthread_exit()`和异常终止, <b>不包括return</b>)都将执行`pthread_cleanup_push`所指定的清理函数.

```
// 功能: 将清除函数压入清除栈
// rtn: 清除函数
// arg: 清除函数的参数
#include <pthread.h>
void pthread_cleanup_push(void (*rtn)(void*), void* arg)
```
```
// 功能: 将清除函数弹出清除栈
// execute: 执行到pthread_cleanup_pop()时, 是否在弹出清理函数的同时执行该函数, 非0=执行; 0=不执行;
#include <pthread.h>
void pthread_cleanup_pop(int execute)
```
```
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void* clean(void* arg)
{
  printf("cleanup:%s ...\n", (char*)arg);
  return (void*)0;
}

void* thr_fn1(void* arg)
{
  printf("thread 1 start ...\n");
  
  pthread_cleanup_push((void*)clean, "thread 1 first handler");
  pthread_cleanup_push((void*)clean, "thread 1 second handler");
  
  printf("thread 1 push complete ...\n");
  if (arg)
  {
    return ((void*)1);
  }
  
  pthread_cleanup_pop(0);
  pthread_cleanup_pop(0);
  
  return (void*)1;
}

void* thr_fn2(void* arg)
{
  printf("thread 2 start ...\n");
  
  pthread_cleanup_push((void*)clean, "thread 2 first handler");
  pthread_cleanup_push((void*)clean, "thread 2 second handler");
  
  printf("thread 2 push complete ...\n");
  if (arg)
  {
    pthread_exit((void*)2);
  }
  
  pthread_cleanup_pop(0);
  pthread_cleanup_pop(0);
  
  pthread_exit((void*)2);
}

int main(void)
{
  int err;
  pthread_t tid1, tid2;
  void* tret;
  
  err = pthread_creat(&tid1, NULL, thr_fn1, (void*)1);
  if (err != 0)
  {
    printf("error ...\n");
    return -1;
  }
  
  err = pthread_creat(&tid2, NULL, thr_fn2, (void*)1);
  if (err != 0)
  {
    printf("error ...\n");
    return -1;
  }
  
  err = pthread_join(tid1, &tret);
  if (err != 0)
  {
    printf("error ...\n");
    return -1;
  }
  printf("thread 1 exit code %d ...\n", (int)tret);
  
  
  err = pthread_join(tid2, &tret);
  if (err != 0)
  {
    printf("error ...\n");
    return -1;
  }
  printf("thread 2 exit code %d ...\n", (int)tret);
  
}
```

###线程同步

