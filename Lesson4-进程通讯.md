##lesson4 进程通讯
###概述
* 目的 - 为什么进程间需要通讯?
	1. 数据传输 - 一个进程需要将它的数据发送给另一个进程
	2. 资源共享 - 多个进程之间共享同样的资源
	3. 通知事件 - 一个进程需要向另一个或一组进程发送消息, 通知他们发生了某种事件
	4. 进程控制 - 有些进程希望完全控制另一个进程的执行(如Debug进程), 此时控制进程希望能够拦截另一个进程的所有操作,并能够及时知道它的状态改变。

* 发展 - Linux进程间通信(IPC)由以下几部分发展而来:
	1. UNIX进程间通讯
	2. 基于System V(Unix分支之一)进程间通讯
	3. POSIX(Portable OS Interface 可移植操作系统接口)进程间通讯

* 现在Linux使用的进程间通讯方式包括:
	1. 管道(pipe)和有名管道(FIFO)
	2. 信号(signal)
	3. 共享内存
	4. 消息队列
	5. 信号量
	6. 套接字(socket)


###管道通讯
* 什么是管道?

> 管道是<b>单向的、先进先出的</b>, 它把一个进程的输出和另一个进程的输入连接在一起。<b>写进程</b>在管道尾部写入数据, <b>读进程</b>从管道头部读出数据。

* 管道种类

	1. 无名管道 - 用于父子进程间的通讯.
	2. 有名管道 - 用于同一系统中的任意两个进程间的通讯.

* 管道创建

```
// 无名管道
int pipe(int filedis[2])
//文件描述符 - filedis[0]: 用于读管道
//文件描述符 - filedis[1]: 用于写管道

// 有名管道(FIFO文件 本质上就是一个文件)
#include <sys/types.h>
#include <sys/stat.h>
int mkfifo(const char* pathname, mode_t mode)
pathname: FIFO文件名
mode: 文件属性
// 一旦创建了一个FIFO,就可用open打开它,一般的文件访问函数(close、read、write等)都可用于FIFO
```

* 管道关闭
`close(int fd)`

```
// 实例 - 管道创建与关闭
#include <unistd.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

int main()
{
  int pipe_fd[2];
  if (pipe(pipefd) < 0)
  {
    printf("pipe create failure\n");
    return -1;
  }
  else
  {
    printf("pipe create success\n");
  }
  
  close(pipe_fd[0]);
  close(pipe_fd[1]);
}
```

* 无名管道读写

> 通常先创建一个管道[pipe()],再创建一个子进程[fork()],该子进程会继承父进程所创建的管道。

```
#include <unistd.h>
#include <sys/types.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

int main()
{
  int pipe_fd[2];
  pid_t pid;
  char buf_r[100];
  char* p_buf_w;
  int r_num;
  
  memset(buf_r, 0, sizeof(buf_r));
  
  if (pipe(pipefd) < 0)
  {
    printf("pipe create failure\n");
    return -1;
  }
  
  pid = fork();
  if (pid == 0)
  {// child process.
    printf("\n");
    close(pipe_fd[1]);// 子进程不需要写管道
    sleep(2);// 等待父进程将数据写入管道
    
    r_num = read(pipe_fd[0], buf_r, 100);
    if (r_num > 0)
    {
      printf("%d bytes read from pipe: %s\n", r_num, buf_r);
    }
    
    close(pipe_fd[0]);// 结束管道读取
    exit(0);
  }
  else if (pid > 0)
  {// parent process.
    close(pipe_fd[0]);//父进程不需要读管道
    
    if (write(pipe_fd[1], "Hello", 5) != -1)
    {
      printf("parent write Hello!\n");
    }
    
    close(pipe_fd[1]);// 结束管道写入
    sleep(3);
    waitpid(pid, NULL, 0);//等待子进程结束
    exit(0);
  }
}
```

* 有名管道操作

打开FIFO文件时,非阻塞标志(O_NONBLOCK)使用与否区别:

1. 没有使用: 访问要求无法满足时进程阻塞,(例如, 试图读取空的FIFO文件,将导致进程阻塞).
2. 有使用: 访问要求无法满足时进程不阻塞,立即返回错误码, errno是ENXIO.

```
//fifo_write.c
#include <sys/types.h>
#include <sys/stat.h>
#include <errno.h>
#include <fcntl.h>
 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
 
#define FIFO_SERVER "/tmp/myfifo"
 
void main(int argc, char** argv)
{
  int fd;
  char w_buf[100];
  int nwrite;
   
  fd = open(FIFO_SERVER, O_WRONLY|O_NONBLOCK, 0);
   
  if (argc == 1)
  {
    printf("Please send somthing\n");
    exit(-1);
  }
   
  strcpy(w_buf, argv[1]);
   
  nwrite = write(fd, w_buf, 100);
  if (nwrite == -1)
  {
    if (errno == EAGAIN)
    {
      printf("The FIFO has not been read yet. Please try later\n");
    }
  }
  else
  {
    printf("write %s to the FIFO\n", w_buf);
  }
}
```
```
//fifo_read.c
#include <sys/types.h>
#include <sys/stat.h>
#include <errno.h>
#include <fcntl.h>
 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
 
#define FIFO "/tmp/myfifo"

void main(int argc, char** argv)
{
  char buf_r[100];
  int fd;
  int nread;
  
  if ((mkfifo(FIFO, O_CREAT|O_EXCL) < 0)
   && (errno != EEXIST))
  {
    printif("Cannot create fifo server\n");
  }
  
  printif("Preparing for reading bytes...\n");
  memset(buf_r, 0, sizeof(buf_r));
  
  fd = open(FIFO, O_RDONLY|O_NONBLOCK, 0);
  if (fd == -1)
  {
    perror("open");
    exit(1);
  }
  
  while(1)
  {
    memset(buf_r, 0, sizeof(buf_r));
    
    nread = read(fd, buf_r, 100);
    if (nread == -1)
    {
      if (errno == EAGAIN)
      {
        printf("no data yet\n");
      }
    }
    
    printf("read %s from FIFO\n", buf_r);
    sleep(1);
  }
  pause();//暂停, 等待信号
  unlink(FIFO);//删除文件
}
```

###信号通讯
* 信号(signal)机制是Unix系统中最为古老的进程间通讯机制, 产生信号的条件:
	1. 当用户按某些按键时;
	2. 硬件异常产生信号。硬件检测到异常 -> 通知内核 -> 内核产生信号通知进程;
	3. 进程用kill函数将信号发送给另一个进程;
	4. 用户可用kill命令将信号发送给其他进程;

* 信号类型
	1. SIGHUP - 从终端上发出的结束信号
	2. SIGINT - 来自键盘的中断信号 (Ctrl+C)
	3. SIGQUIT
	4. SIGILL
	5. SIGTRAP
	6. SIGIOT
	7. SIGBUS
	8. SIGFPE
	9. SIGKILL - 该信号结束接收信号的进程
	10. SIGUSR1
	11. SIGSEGV
	12. SIGUSR2
	13. SIGPIPE
	14. SIGALRM
	15. SIGTERM - kill命令发出的信号
	16. SIGCHLD - 标识子进程停止或结束的信号
	17. SIGCONT
	18. SIGSTOP - 来自键盘(Ctrl+Z)或调试程序的停止执行信号
	19. SIGTSTP
	20. SIGTTIN
	21. SIGTTOU
	22. SIGURG
	23. SIGXCPU
	24. SIGXFSZ
	25. SIGVTALRM
	26. SIGPROF
	27. SIGWINCH
	28. SIGIO
	29. SIGPWR

* 信号处理
	1. 忽略此信号 - 除了SIGKILL和SIGSTOP, 其他大多数信号均可忽略, 因为这两种信号向超级用户提供了一种终止或停止进程的方法。
	2. 执行用户希望的动作 - 调用一个用户函数, 在用户函数中, 执行用户希望的处理。
	3. 执行系统默认动作 - 大多数信号的系统默认动作是终止该进程。

* 信号发送
	1. `int kill(pid_t pid, int signo)` - 既可向自身进程发送信号,也可向其他进程发送信号;
	2. `int raise(int signo)` - 只能向自身进程发送信号;

* kill的pid参数有4种不同的情况:
	1. pid > 0  : 将信号发送给进程ID为pid的进程;
	2. pid == 0 : 将信号发送给同组的进程;
	3. pid < 0  : 将信号发送给起进程组ID等于pid绝对值的进程
	4. pid == -1: 将信号发送给所有进程;

* Alarm信号

```
#include <unistd.h>
unsigned int alarm(unsigned int seconds)
// 功能描述: 经过指定seconds秒后产生信号SIGALRM, 如果不捕捉该信号, 默认动作是终止该进程。

```
> 每个进程只能有一个闹钟时间。如果在调用alarm时,以前已为该进程设置过闹钟时间,而且它还没超时, 以前设置的闹钟时间则被新值替换, 如果新值为0,则表示取消以前的闹钟。

* pause函数
`int pause(void)`

> 挂起进程直至捕捉到一个信号, 并执行一个信号处理函数后, 挂起才结束。

* 信号处理主要方法:
	1. 使用简单的signal函数
	2. 使用信号集函数组


```
// 使用shell命令: ps aux 可查询当前所有正在运行的进程及其进程号
// 使用shell命令: kill -s SIGQUIT 3687(pid) 可给指定进程发送指定信号
#include <signal.h>
void (*signal(int signo, void (*func)(int)))(int)

typedef void (*sighandler_t)(int)
sighandler_t signal(int signum, sighandler_t handler)

func/handler 可能的值:
1. SIG_IGN: 忽略此信号
2. SIG_DFL: 执行系统默认动作
3. 信号处理函数名: 使用该函数处理
```

```
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>

void my_func(int sign_no)
{
  if (sign_no == SIGINT)
  {
    printf("I have get SIGINT\n");
  }
  else if (sign_no == SIGQUIT)
  {
    printf("I have get SIGQUIT\n");
  }
}

int main()
{
  printf("Waiting for signal SIGINT or SIGQUIT\n");
  
  signal(SIGINT, my_func);
  signal(SIGQUIT, my_func);
  
  pause();
  exit(0);
}
```

###共享内存
> 被多个进程共享的一部分物理内存。共享内存是进程间共享数据的一种最快的方法, 一个进程向共享内存区域写入了数据, 共享这个内存区域的所有进程就可以立即看到其中的内容。

* 实现步骤
	1. 创建共享内存 - shmget
	2. 映射共享内存 - shmat
	3. 解除映射(使用完毕) - shmdt

```
int shmget(key_t key, int size, int shmflg)
key    : 0/IPC_PRIVATE(创建一块新的共享内存)
shmflg : IPC_PRIVATE(&& key == 0 创建一块新的共享内存)
return : -1 = failure; >0 = success, 共享内存标识符;
```
```
int shmat(int shmid, char* shmaddr, int flag)
shmid  : shmget返回的共享内存标识符;
flag   : 决定以什么方式来确定映射的地址(通常为0)
return : -1 = failure; >0 = success;
```
```
int shmat(char* shmaddr)
```
```
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#include <errno.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>

#define PERM S_IRUSR|S_IWUSR

#define SHM_SIZE  (1024)

int main(int argc, char** argv)
{
  int shmid;
  char* p_addr;
  char* c_addr;
  
  if (argc != 2)
  {
    fprintf(stderr, "Usage:%s\n\a", argv[0]);
    exit(1);
  }
  
  shmid = shmget(IPC_PRIVATE, SHM_SIZE, PERM);
  if (shmid == -1)
  {
    fprintf(stderr, "Create Share Memory Error:%s\n\a", strerror(errno));
    exit(1);
  }
  
  pid_t pid = fork();
  if (pid > 0)
  {// parent process -> write
    p_addr = shmat(shmid, 0, 0);
    memset(p_addr, '\0', SHM_SIZE)
    strncpy(p_addr, argv[1], 1024);
    wait(NULL); //释放资源, 不关心终止状态
    exit(0);
  }
  else if (pid == 0)
  {// child process -> read
    sleep(1);
    c_addr = shmat(shmid, 0, 0);
    printf("Client get %s\n", c_addr);
    exit(0);
  }
}
```

###消息队列
> 信号(signal)信息量有限, 管道只能传送无格式的字节流.
> 
> 消息队列克服了上述缺点.
> 
> 消息队列就是一个消息的链表。具有特定的格式。进程可以向其中按照一定的规则添加新消息;另一些进程则可以从消息队列中读走消息;

* 分类
	1. POSIX消息队列
	2. 系统V消息队列 (目前被大量使用).

* 持续性

> 系统V消息队列是随内核持续的, 只有在内核重启或者人工删除时, 该消息队列才会被删除.
> 
> 消息队列的内核持续性, 要求每个消息队列都在系统范围内对应唯一的<b>键值</b>, 要获得一个消息队列的描述符(id), 必须提供该消息队列的键值。

```
// 获取文件对应的键值
#include <sys/types.h>
#include <sys/ipc.h>
key_t ftok(char* pathname, char proj)
pathname: 文件名
proj    : 项目名(不为0即可)
```

* 打开/创建 消息队列

```
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
int msgget(key_t key, int msgflg)
key: 键值, 由ftok获得.(如果为IPC_PRIVATE时, 将创建一个新的消息队列)
msgflg: 标志位.
	IPC_CREAT - 创建新的消息队列(没有与key相对应的消息队列时)
	IPC_EXCL - 与IPC_CREAT一起使用, 表示要创建的消息队列已经存在时, 返回错误.
	IPC_NOWAIT - 读写消息队列要求无法得到满足时,不阻塞.
return: 与key对应的消息队列描述字.
```
```
int open_queue(key_t keyval)
{
  int qid;

  qid = msgget(keyval, IPC_CREAT);
  if (qid == -1)
  {
    return (-1);
  }
 
  return (qid);
}
```

* 发送消息

```
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
int msgsnd(int msqid, struct msgbuf* msgp, int msgsz, int msgflg)
msqid : 已打开的消息队列id
msgp  : 消息数据指针
msgsz : 消息数据长度
msgflg: 发送标志, IPC_NOWAIT -> 在消息队列没有足够空间容纳要发送的消息时是否等待.

struct msgbuf
{
  long mtype;    // 消息类型 > 0
  char mtext[1]; // 消息数据首地址
}
```

* 接收消息

```
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
int msgrcv(int msqid, struct msgbuf* msgp, int msgsz, long msgtyp, int msgflg)
msgtyp : 要读取的消息类型
```
> 注: 在成功读取了一条消息后, 队列中将自动删除这条消息

```
int read_message(int qid, long type, struct mymsgbuf* qbuf)
{
  int result, length;
  length = sizeof(struct mymsgbuf) - sizeof(long);
  
  result = msgrcv(qid, qbuf, length, type, 0);
  if (result == -1)
  {
    return (-1);
  }
  return (result);
}
```

```
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>

#include <unistd.h>

struct msg_buf
{
  int mtype;
  char data[255];
};

int main()
{
  key_t key;
  int msgid;
  int ret;
  struct msg_buf msgbuf;
  
  // 1. message queue create
  key = ftok("/tmp/2", 'a');
  printf("key = [%x]\n", key);
  msgid = msgget(key, IPC_CREAT|0666);
  if (msgid == -1)
  {
    printf("create message queue error\n");
    return -1;
  }
    
  // 2. message send
  msgbuf.mtype = getpid();
  strcyp(msgbuf.data, "test data haha");
  ret = msgsnd(msgid, &msgbuf, sizeof(msgbuf.data), IPC_NOWAIT);
  if (ret == -1)
  {
    printf("send message error\n");
    return -1;
  }
  
  // 3. message receive
  memset(&msgbuf, 0, sizeof(msgbuf));
  ret = msgrcv(msgid, &msgbuf, sizeof(msgbuf.data), getpid(), IPC_NOWAIT);
  if (ret == -1)
  {
    printf("recv message error\n");
    return -1;
  }
  printf("recv msg = [%s]\n", msgbuf.data);
}
```

###信号量
* 主要作用:
	1. <b>保护临界资源</b>. 进程可以根据它判定是否能够访问某些共享资源.
	2. <b>进程同步</b>.

* 分类:
	1. 二值信号量: 类似互斥锁, 两者区别:
		- 二值信号量 - 更强调共享资源, 只要共享资源可用, 其他进程同样可修改信号量的值;
		- 互斥锁 - 更强调进程, 占用资源的进程使用完资源后,必须由进程本身来解锁;
	2. 计数信号量: 可取任意非负值

* 信号量 - 创建/打开

```
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>
int semget(key_t key, int nsems, int semflg)
key: 由ftok获得
nsems: 指定打开或者新创建的[信号量集]将包含信号量的数目
semflg: 标识, 同消息队列
```

* 信号量 - 操作

```
int semop(int semid, struct sembuf* sops, unsigned nsops)
semid: 信号量集的ID
sops : 是一个操作数组, 表明要进行数目操作 
nsops: sops所指向数组的元素个数

struct sembuf
{
  unsigned short sem_num; //semaphore index in array
  short sem_op;  // semaphore operation
  short sem_flg; // opreation flags
  // opreation flags:
  // 1. IPC_NOWAIT: 操作不能满足时不阻塞 返回错误
  // 2. IPC_UNDO: 程序结束时(包括异常结束时)释放信号量
}
```

###套接字(socket)




