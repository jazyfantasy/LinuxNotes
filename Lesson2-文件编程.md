# Lesson 2 - 文件编程
## 文件访问 - 系统调用
> 依赖于Linux系统

* 系统调用 - 创建
`int creat(const char *filename, mode_t mode)`
	* filename: 要创建的文件名(包含路径,缺省为当前路径)
	* mode:文件属性
	  - 常见文件属性(宏定义/数字):
	 		- S_IRUSR / 4 / 可读
	 		- S_IWUSR / 2 / 可写
	 		- S_IXUSR / 1 / 可执行
	 		- S_IRWXU / 7 / 可读写、执行
	 		- 0 / 无任何权限

```C
#include  <stdio.h>
#include  <stdlib.h>

#include  <sys/types.h>
#include  <sys/stat.h>
#include  <fcntl.h>

void creat_file(char* filename)
{
  if (creat(filename, 0755) < 0)
  {
    printf("creat file %s failure!\n", filename);
    exit(EXIT_FAILURE);
  }
  else
  {
    printf("creat file %s success!\n", filename);
  }
}

int main(int argc, char *argv[])
{
  int i;
  if (argc < 2)
  {
    perror("you haven't input the filename, please try again!\n");
    exit(EXIT_FAILURE);
  }
  for(i=1; i<argc; i++)
  {
  	creat_file(argv[i]);
  }
  exit (EXIT_SUCCESS);
}
```

* 系统调用 - 打开
`int open(const char *pathname, int flags)`
`int open(const char *pathname, int flags, mode_t mode)`
	* pathname: 要打开的文件名(包含路径,缺省为当前路径)
	* flags: 打开标志
		- O_RDONLY	只读
		- O_WRONLY	只写
		- O_RDWR		读写
		- O_APPEND	追加
		- O_CREAT		创建(当不存在此文件时)
		- O_NOBLOCK	非阻塞
	* mode: 文件属性 (flags = O_CREAT 时使用)
* 系统调用 - 关闭
`int close(int fd)`
	* fd: 文件描述符

```
#include  <stdio.h>
#include  <stdlib.h>

#include  <sys/types.h>
#include  <sys/stat.h>
#include  <fcntl.h>

int main (int argc, char *argv[])
{
  int fd;
  if (argc<2)
  {
    puts("please input the open file pathname!\n");
    exit(1)
  }
  
  fd = open(argv[1], O_CREAT|O_RDWR, 0755);
  if (fd < 0)
  {
    perror("open file failure: %d\n", fd);
    exit(1);
  }
  else
  {
    printf("open file %d success!\n", fd);
  }
  
  close(fd);
  exit(0);
}
```

* 系统调用 - 读
`int read(int fd, const void *buf, size_t length)`
	> 功能: 从文件描述符fd 所指定的文件中读取length个字节到buf所指向的缓冲区中, 返回值为实际读取到的字节数。

* 系统调用 - 写
`int write(int fd, const void *buf, size_t length)`
	> 功能: 把length个字节从buf指向的缓冲区中写入文件描述符fd所指向的文件中, 返回值为实际写入的字节数。

* 系统调用 - 定位
`int lseek(int fd, offset_t offset, int whence)`
	> 功能: 将文件读写指针相对whence移动offset个字节。操作成功时, 返回文件指针相对文件头的位置。
	* offset 可取负值, 表示向前移动。
	* whence 可使用如下宏定义:
		- SEEK_SET: 相对文件开头
		- SEEK_CUR: 相对文件读写指针的当前位置
		- SEEK_END: 相对文件末尾

```
// 利用lseek计算文件长度
int fileLen = lseek(fd, 0, SEEK_END);
```

* 系统调用 - 访问判断
`int access(const char *pathname, int mode)`
	* pathname: 文件名称
	* mode: 要判断的访问权限, 可取以下值或其组合:
		- R_OK: 文件可读
		- W_OK: 文件可写
		- X_OK: 文件可执行
		- F_OK: 文件存在
	* return: 0 = 所有权限测试通过; -1 = 有权限没有测试通过;

```
// 综合实例: (文件拷贝)
#include  <stdio.h>
#include  <stdlib.h>

#include  <sys/types.h>
#include  <sys/stat.h>
#include  <fcntl.h>
#include  <errno.h>

#define  BUFFER_SIZE  (1024)

int main(int argc, char **argv)
{
  int fd_src, fd_dst;
  int bytes_read, bytes_write;
  char buffer[BUFFER_SIZE];
  char *ptr;
  
  if (argc != 3)
  {
    fprintf(stderr, "Usage:%s from file to file/n/a", argv[0]);
    exit(1);
  }
  
  fd_src = open(argv[1], O_RDONLY);
  if (fd_src == -1)
  {
    fprintf(stderr, "Open %s Error: %s/n", argv[1], strerror(errno));
    exit(1);
  }
  fd_dst = open(argv[2], O_WRONLY|O_CREAT, S_IRUSR|S_IWUSR);
  if (fd_dst == -1)
  {
    fprintf(stderr, "Open %s Error: %s/n", argv[2], strerror(errno));
    exit(1);
  }
  
  while (bytes_read = read(fd_src, buffer, BUFFER_SIZE))
  {
    if ((bytes_read == -1) && (errno != EINTR))
    {// death error!!!
      break;
    }
    else if (byte_read > 0)
    {
      ptr = buffer;
      while (bytes_write = write(fd_dst, ptr, bytes_read))
      {
        if ((bytes_read == -1) && (errno != EINTR))
        {// death error!!!
          break;
        }
        else if (bytes_write == bytes_read)
        {// done.
          break;
        }
        else if (bytes_write > 0)
        {// continue writing
          ptr += bytes_write;
          bytes_read -= bytes_write;
        }
      }
      
      if (bytes_write == -1)
      {// death error!!!
        break;
      }
    }
  }
  
  close (fd_src);
  close (fd_dst);
  exit(0);
}
```

## 文件访问 - C库函数
> 不依赖于操作系统 依赖于C库

* 库函数 - 创建和打开
`FILE* fopen(const char *filename, const char* mode)`
	* filename: 要创建/打开的文件名(包含路径,缺省为当前路径)
	* mode:创建/打开模式
		- r,rb 只读 (b用于区分二进制文件和文本文件, 但Linux不区分)
		- w,wb 只写,(如果文件不存在,自动创建)
		- a,ab 追加,(如果文件不存在,自动创建)
		- r+,r+b,rb+ 读写方式打开
		- w+,w+b,wh+ 读写方式打开,(如果文件不存在,自动创建)
		- a+,a+b,ab+ 读和追加方式打开,(如果文件不存在,自动创建)

* 库函数 - 读
`size_t fread(void *ptr, size_t size, size_t n, FILE* stream)`
	> 功能: 从stream指向的文件中读取n个字段, 每个字段为size字节, 并将读取的数据放入ptr所指向的字符数组中,返回实际已读取的字节数.

* 库函数 - 写
`size_t fwrite(void *ptr, size_t size, size_t n, FILE* stream)`

* 库函数 - 读字符	`int fgetc(FILE* stream)`

* 库函数 - 写字符	`int fputc(int c, FILE* stream)`

```
#include <stdio.h>
main()
{
  FILE *fp;
  char ch;
  fp = fopen("c1.txt", "rt");
  if (fp == NULL)
  {
    printf("\nCannot open file, strike any key exit!");
    getch();
    exit(1);
  }
  
  ch = fget(fp);
  while(ch != EOF)
  {
    putchar(ch);
    ch = fgetc(fp);
  }
  fclose(fp);
}
```

* 库函数 - 格式化读
`fscanf(FILE * stream, char* format[,argument,...])`

```
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
  int i;
  printf("Input an integer:");
  if (fscanf(stdin, "%d", &i))// stdin: 标准输入文件
  {
    printf("The integer read was: %i\n", i);
  }
  
  return 0;
}
```

* 库函数 - 格式化写
`int fprintf(FILE* stream, char* format[,argument,...])`

* 库函数 - 定位
`int fseek(FILE* stream, long offset, int whence)`
	* whence: 可使用如下宏定义:
		- SEEK_SET: 相对文件开头
		- SEEK_CUR: 相对文件读写指针的当前位置
		- SEEK_END: 相对文件末尾

* 路径获取

```
#include <unistd.h>
char* getcwd(char* buffer, size_t size)
```
> 提供一个size大小的buffer, 把当前路径名copy到buffer中; 如果buffer太小, 函数返回-1

* 创建目录

```
#include <sys/stat.h>
int mkdir(char* dir, int mode)
return: 0=success; -1=failure;
```

## 时间编程
* 时间类型
	* UTC(Coordinated Universal Time) => GMT(Greenwich Mean Time)
	* Calendar Time: 日历时间(unix 时间戳 - 自1970年1月1日0点起)

* 时间获取 - 日历时间

```
#include  <time.h>
time_t time(time_t *tloc)
```
```
struct tm {
  int tm_sec;
  int tm_min;
  int tm_hour;
  
  int tm_mday; // 本月第几日
  int tm_mon;
  int tm_year; // tm_year+1900=哪一年
  int tm_wday; // 本周第几日
  int tm_yday; // 本年第几日
  int tm_isdst;// 日光节约时间
};
```

* 时间转换

```
//日历时间转换为 UTC(GMT)
struct tm *gmtime(const time_t *timep)
```
```
//日历时间转换为 本地时间
struct tm *localtime(const time_t *timep)
```

* 时间显示

```
// tm格式的时间转换为字符串
char* asctime(const struct tm* tm)

// 日历时间转换为本地时间的字符串
char* ctime(const time_t *timep)
```

* 获取时间差

```
// 获取从今天凌晨到现在的时间差, 常用于计算事件耗时
int gettimeofday(struct timeval* tv, struct timezone* tz)

struct timeval
{
  int tv_sec;  //秒
  int tv_usec; //微秒
}
```

* 延时执行

```
// 秒级延时
unsigned int sleep(unsigned int seconds)

// 微秒级延时
void usleep(unsigned long usec)
```


