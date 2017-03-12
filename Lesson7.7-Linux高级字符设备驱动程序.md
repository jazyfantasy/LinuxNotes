# Linux 高级字符设备驱动程序

## 设备ioctl控制

* 功能
  - 对硬件进行控制
  - 用户使用方法
    - `int ioctl(int fd, unsigned long cmd, ...)`
  - 驱动ioctl方法
    - `int (*ioctl)(struct inode* inode, struct file* filp, unsigned int cmd, unsigned long arg)`

* 实现
  1. 定义命令
    - 命令号应该在系统范围内唯一
    - ioctl命令编码被划分为几个位段, 定义在`include/asm/ioctl.h`中, 符号定义在`linux/ioctl.h`中:
      1. `Type` - 类型(幻数)
        - `Documentation/ioctl-numbre.txt`中罗列了内核中已经使用了的幻数;
        - 表明哪个设备的命令, 参考了`ioctl-numbre.txt`之后选出, 8位宽;
      2. `Number` - 序号
        - 表明设备命令中的第几个, 8位宽
      3. `Direction` - 传送方向
        - 数据传送的方向:
          1. `_IOC_NONE` - (没有数据传输)
          2. `_IOC_READ` - 从设备读
          3. `_IOC_WRITE` - 往设备写
      4. `Size` - 参数的大小
        - 用户数据的大小(13/14位宽, 视处理器而定)
    - 定义命令的宏
      1. `_IO(type, nr)` - 没有参数的命令
      2. `_IOR(type, nr, datatype)` - 从设备中读数据
      3. `_IOW(type, nr, datatype)` - 往设备中写数据
      3. `_IOWR(type, nr, datatype)` - 双向传输
    - 范例:
      ```
      #define MEM_IOC_MAGIC   'm' //定义幻数

      #define MEM_IOCSET    _IOW(MEM_IOC_MAGIC, 0, int)
      #define MEM_IOCGQSET  _IOR(MEM_IOC_MAGIC, 1, int)
      ```
  2. 实现命令
    1. 返回值
      - 当命令号不能匹配任何一个设备所支持的命令时, 通常返回 `-EINVAL`(非法参数)
    2. 参数使用
      - 整数参数 -> 直接使用
      - 指针参数 -> 使用前必须先进行正确的检查, 以确保该用户地址有效
          - 不需要检查:
            - `copy_from_user`
            - `copy_to_user`
            - `get_user`
            - `put_user`
          - 需要检查:
            - `__get_user`
            - `__put_user`
      - 检查函数:
        ```
        int access_ok(int type, const void* addr, unsigned long size)
        parameters:
          - type: VERIFY_READ/VERIFY_WRITE => 读/写用户内存
          - addr: 用户内存地址;
          - size: 操作的长度;
        return:
          1=成功(存取ok)
          0=失败(存取error)
          返回失败 => ioctl应当返回 -EFAULT
        ```

      ```
      if (_IOC_DIR(CMD) & _IOC_READ)
      {
        err = !access_ok(VERIFY_WRITE, (void __user*)arg, _IOC_SIZE(cmd));
      }
      else if (_IOC_DIR(CMD) & _IOC_WRITE)
      {
        err = !access_ok(VERIFY_READ, (void __user*)arg, _IOC_SIZE(cmd))
      }

      if (err)
      {
        return -EFAULT;
      }
      ```

    3. 命令操作
      ```
      switch (cmd)
      {
        case MEM_IOCSQUANTUM:
          retval = __get_user(scull_quantum, (int*)arg);
        break;

        case MEM_IOCGQUANTUM:
          retval = __put_user(scull_quantum, (int*)arg);
        break;

        default:
          return -EINVAL;
      }
      ```

## 内核等待队列

* 等待队列作用:
  - 可用于实现进程的阻塞, 可看做保存进程的容器;
* 等待队列操作:
  1. 定义等待队列
    - `wait_queue_head_t my_queue`
  2. 初始化等待队列
    - `init_waitqueue_head(&my_queue)`
  3. 定义并初始化等待队列
    - `DECLARE_WAIT_QUEUE_HEAD(my_queue)`
  4. 有条件睡眠
    - `condition`为真时, 立即返回; 否则进程进入睡眠;
    - `wait_event(queue, condition)`
      - 睡眠模式`TASK_UNINTERRUPTIBLE`
    - `wait_event_interruptible(queue, condition)`
      - 睡眠模式`TASK_INTERRUPTIBLE`
    - `int wait_event_killable(wait_queue_t queue, condition)`
      - 睡眠模式`TASK_KILLABLE`
  5. 无条件睡眠(老版本, 建议不再使用)
    - `sleep_on(wait_queue_head_t *q)`
    - `interruptible_sleep_on(wait_queue_head_t *q)`
  6. 从等待队列中唤醒进程
    - `wake_up(wait_queue_t *q)`
    - `wake_up_interruptible(wait_queue_t *q)`

## 阻塞型字符设备驱动

* 当一个设备无法立即满足用户的读写请求时, 应如何处理?
  - 驱动程序应当(缺省的)阻塞进程, 使它进入睡眠, 直到请求可以得到满足;

* 阻塞方式
  - `read`实现方式:
    - 如果进程调用`read`, 但设备没有数据或数据不足,进程阻塞; 当新数据到达后, 唤醒被阻塞进程;
  - `write`实现方式:
    - 如果进程调用`write`, 但设备没有足够的空间供其写入数据, 进程阻塞; 当设备空出部分空间, 则唤醒进程;
* 非阻塞方式
  - 应用程序使用`O_NONBLOCK`标志设置为非阻塞模式
  - `read`/`write`操作条件不足时, 返回`-EAGAIN`, 而不阻塞进程;

## Poll设备操作

| 系统调用(用户空间) | 驱动(内核空间) |
| ---------- |:-------------:|
| open | open |
| close | close |
| read | read |
| write | write |
| ioctl | ioctl |
| lseek | llseek |
| select | poll |

* select系统调用
  - `int select(int maxfd, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, const struct timeval* timeout)`
  - 功能:
    - 用于多路监控, 当没有一个文件满足要求时, `select`将阻塞调用进程;
  - 参数:
    - `maxfd`: 文件描述符的范围, 比待检测的最大文件描述符大1
    - `readfds`: 被读监控的文件描述符集
    - `writefds`: 被写监控的文件描述符集
    - `exceptfds`: 被异常监控的文件描述符集
    - `timeout`: 定时器
      - timeout值 = 0: 不管是否有文件满足要求, 都立即返回; 无满足返回0, 有满足返回正值;
      - timeout = NULL: 阻塞进程, 直至有满足;
      - timeout值 > 0: 等待的最长时间, 在timeout时间内阻塞进程;
  - 返回值:
    1. 正整数: 满足要求的文件描述符个数
    2. 0: 无满足要求的文件;
    3. -1: 异常出错, 如果`errno`=`EINTR`, 则出错原因为被某个信号中断;
  - 使用方法:
    1. 将要监控的文件添加到文件描述符集
    2. 调用`select`开始监控
    3. 判断文件是否发生变化
  - 对描述符集进行操作的宏
    - `#include <sys/select.h>`
    - `void FD_SET(int fd, fd_set* fdset)`
      - 将文件描述符fd, 添加到文件描述符集fdset中;
    - `void FD_CLR(int fd, fd_set* fdset)`
      - 将文件描述符fd, 从文件描述符集fdset中清除;
    - `void FD_ZERO(fd_set* fdset)`
      - 清空文件描述符集fdset;
    - `void FD_ISSET(int fd, fd_set* fdset)`
      - 检测文件描述符集fdset中的文件fd是否发生了变化;

  ```
  FD_ZERO(&fds);
  FD_SET(fd1, &fds);
  FD_SET(fd2, &fds);

  maxfdp = fd1+1;// 描述符最大值加1, (假设fd1>fd2)
  switch (select(maxfdp, &fds, NULL, NULL, &timeout))
  {
    case -1:
      exit(-1);
    break;

    case 0:
    break;

    default:
      if (FD_ISSET(fd1, &fds))
      {
        // 测试fd1是否可读
      }
    break;
  }
  ```

  * Poll方法
    - 原型: `unsigned int (*poll)(struct file *filp, poll_table* wait)`
    - 负责工作:
      1. 使用`poll_wait`将等待队列添加到`poll_table`中
      2. 返回描述设备是否可读/写的掩码:
        - `POLLIN` - 设备可读
        - `POLLOUT` - 设备可写
        - `POLLRDNORM` - 数据可读
        - `POLLWRNORM` - 数据可写
        - 设备可读通常返回 `(POLLIN | POLLRDNORM)`
        - 设备可写通常返回 `(POLLOUT | POLLWRNORM)`
    - 工作原理:
      - poll方法只是做一个登记, 真正的阻塞发生在`select.c`中的`do_select`函数;
    - 范例:
      ```
      static unsigned int mem_poll(struct file* filp, poll_table* wait)
      {
        struct scull_pipe* dev = filp->private_data;
        unsigned int mask = 0;

        // 把等待队列添加到poll_table
        poll_wait(filp, &dev->inq, wait);

        // 返回掩码
        if (有数据可读)
        {
          mask = POLLIN | POLLRDNORM;
        }

        return mask;
      }
      ```

## 自动创建设备文件

* 创建设备文件的方法:
  1. `mknod`手工创建
  2. 自动创建

* 自动创建
  1. 2.4内核

```
devfs_register(devfs_handle_t dir, const char* name, unsigned int flags,
  unsigned int major, unsigned int minor, umode_t mode, void* ops, void* info)
function:
  - 在指定的目录中创建设备文件.
parameters:
  - dir: 目录名, 为空表示在/dev/目录下创建;
  - name: 文件名;
  - flags: 创建标志;
  - major: 主设备号;
  - minor: 次设备号;
  - mode: 创建模式;
  - ops: 操作函数集;
  - info: 通常为空;
```
  2. 2.6内核
    - `udev`代替了`devfs`;
    - `udev(mdev)`存在于应用层;
    - 利用`udev(mdev)`实现设备文件的自动创建很简单:
      - 在驱动初始化的代码里调用`class_creat`为该设备创建一个`class`;
      - 为每个设备调用`device_create`创建对应的设备;
    - 范例:
      ```
      struct class* myclass = class_creat(THIS_MODULE, "my_device_driver");
      device_create(myclass, NULL, MKDEV(major_num, 0), NULL, "my_device");

      // 当驱动被加载时, udev(mdev)就会自动在/dev下创建my_device设备文件
      ```
