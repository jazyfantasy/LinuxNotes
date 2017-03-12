# Linux 硬件设备访问

## mmap设备操作

* `mmap`系统调用
  - 原型: `void* mmap(void* addr, size_t len, int prot, int flags, int fd, off_t offset)`
  - 功能:
    - 内存映射函数`mmap`, 负责把文件内容映射到进程的虚拟空间, 通过对这段内存的读取和修改, 来实现对文件的读取和修改, 而不需要再调用`read`/`write`等操作
  - 参数:
    - `addr`: 指定映射的起始地址, 通常设为NULL, 由系统指定;
    - `len`: 映射到内存的文件长度;
    - `prot`: 映射区的保护方式, 可以是:
      - `PROT_EXEC`: 映射区可被执行
      - `PROT_READ`: 映射区可被读取
      - `PROT_WRITE`: 映射区可被写入
    - `flags`: 映射区的特性, 可以是:
      - `MAP_SHARED`: 写入映射区的数据会复制回文件, 且允许其他映射该文件的进程共享;
      - `MAP_PRIVATE`: 对映射区的写入操作会产生一个映射区的复制, 对此区域所做的修改不会写回原文件;
    - `fd`: 由`open`返回的文件描述符, 代表要映射的文件;
    - `offset` 以文件开始处的偏移量, 必须是分页大小的整数倍, 通常为0, 表示从文件头开始映射;
* 解除映射
  - `int munmap(void* start, size_t length)`
  - 功能: 取消参数start所指向的映射内存, 参数length表示欲取消的内存大小;
  - 返回值: 解除成功返回0, 否则返回-1, 错误原因存于errno中;
* 虚拟内存区域
  - 定义:
    - 进程的虚拟地址空间中的一个同质区间, 即具有同样特性的连续地址范围;
  - 组成:
    1. 程序代码;
    2. 数据;
    3. BSS和栈区域;
    4. 内存映射的区域;
  - 进程的内存区域查看: `/proc/pid/maps`
    - 格式: `start_end perm offset major:minor inode`
      - `start`: 该区域起始虚拟地址;
      - `end`: 该区域结束虚拟地址;
      - `perm`: 读写和执行权限,以及私有(p)/共享(s)特性;
      - `offset`: 被映射部分在文件中的起始地址;
      - `major`/`minor`: 主/次设备号;
      - `inode`: 索引结点;
  - 描述结构 - `vm_area_struct`(`<linux/mm_types.h>`), 主要成员:
    - `unsigned long vm_start`: 虚拟内存区域起始地址
    - `unsigned long vm_end`: 虚拟内存区域结束地址
    - `unsigned long vm_flags`: 该区域的标记
      - `VM_IO`: 将该VMA标记为内存映射的IO区域, 会阻止系统将该区域包含在进程的存放转存(core dump)中;
      - `VM_RESERVED`: 标志内存区域不能被换出;
* mmap设备操作
  - 映射一个设备
    - 把用户空间的一段地址关联到设备内存上;
    - 当程序读写这段用户空间地址时, 实际上是在访问设备;
  - mmap设备方法需要完成的功能
    - 建立虚拟地址到物理地址的页表.
  - 原型: `int (*mmap)(struct file*, struct vm_area_struct*)`
  - 如何完成页表的建立?
    1. 使用`remap_pfn_range`一次建立所有页表;
      ```
      int remap_pfn_range(struct vm_area_struct* vma,
        unsigned long addr, unsigned long pfn,
        unsigned long size, pgprot_t prot)
      parameters:
        - vma: 虚拟内存区域指针;
        - addr: 虚拟地址的起始值;
        - pfn: 要映射的物理地址所在的物理页帧号, 可将物理地址>>PAGE_SHIFT得到;
        - size: 要映射的区域大小;
        - prot: VMA的保护属性;
      ```
      ```
      int memdev_mmap(struct file* filp, struct vm_area_struct* vma)
      {
        vma->vm_flags |= VM_IO;
        vma->vm_flags |= VM_RESERVED;

        if (remap_pfn_range(vma, vma->vm_start,
          virt_to_phys(dev->data)>>PAGE_SHIFT, size, vma->vm_page_prot))
        {
          return -EAGAIN;
        }

        return 0;
      }
      ```
    2. 使用`nopage VMA`方法每次建立一个页表;

## 硬件访问

* 寄存器与内存的区别
  - 寄存器操作有副作用(side effect 或 边际效果): 读取某个地址时可能导致该地址内容发生变化;
* 内存与I/O
  - 内存空间与I/O空间是彼此独立的地址空间
  - IO端口: 位于IO空间的寄存器/内存
    - 操作I/O端口:
      1. 申请
        - 原型:
          ```
          struct resource* request_region(
            unsigned long first, unsigned long n, const char* name)
          function:
            - 使用从first开始的n个端口;
          parameters:
            - name: 设备名字
          return:
            !=NULL: 申请成功;
            =NULL: 申请失败;
          ```
        - 查看IO端口的分配情况: `/proc/ioports`
      2. 访问
        - `unsigned inb(unsigned port)` - 读字节端口(8位宽)
        - `void outb(unsigned char byte, unsigned port)` - 写字节端口(8位宽)
        - `unsigned inw(unsigned port)` - 读端口(16位宽)
        - `void outw(unsigned short word, unsigned port)` - 写端口(16位宽)
        - `unsigned inl(unsigned port)` - 读端口(32位宽)
        - `void outl(unsigned long word, unsigned port)` - 写端口(32位宽)
      3. 释放
        - `void release_region(unsigned long start, unsigned long n)`
  - IO内存: 位于内存空间的寄存器/内存
    1. 申请
      - 原型:
        ```
        struct resource* request_mem_region(
          unsigned long start, unsigned long len, const char* name)
        function:
          - 申请一个从start开始的长度为len字节的内存区;
        parameters:
          - name: 设备名字
        return:
          !=NULL: 申请成功;
          =NULL: 申请失败;
        ```
      - 所有已经在使用的I/O内存列出在: `/proc/iomem`
    2. 映射 - (物理地址到虚拟地址的映射)
      - `void* ioremap(unsigned long phys_addr, unsigned long size)`
    3. 访问
      - `unsigned ioread8(void* addr)`
      - `unsigned ioread16(void* addr)`
      - `unsigned ioread32(void* addr)`
      - `unsigned iowrite8(u8 value, void* addr)`
      - `unsigned iowrite16(u16 value, void* addr)`
      - `unsigned iowrite32(u32 value, void* addr)`
    4. 释放
      1. `void iounmap(void* addr)`
      2. `void release_mem_region(unsigned long start, unsigned long len)`

## 混杂设备驱动

* 混杂设备
  - Linux中存在一类字符设备, 共享一个主设备号, 但次设备号不同, 称之为混杂设备`miscdevice`;
  - 所有混杂设备形成一个链表, 对设备访问时内核根据次设备号查找到相应的`miscdevice`;
* 结构描述 `struct miscdevice`
  ```
  struct miscdevice
  {
    int minor;
    const char* name;
    const struct file_operations* fops;
    struct list_head list;
    struct device* parent;
    struct device* this_device;
  }
  ```
* 混杂设备注册
  - `int misc_register(struct miscdevice* misc)`
* 设备资源
  - 结构描述
    ```
    struct resource
    {
      resource_size_t start;
      resource_size_t end;
      const char* name;
      unsigned long flags;
      struct resource *parent, *sibling, *child;
    }
    ```
  - 实例:
    ```
    static struct resource s3c_wdt_resource1 = {
      .start = 0x44100000,
      .end = 0x44200000,
      .flags = IORESOURCE_MEM,
    }
    static struct resource s3c_wdt_resource2 = {
      .start = 20,
      .end = 20,
      .flags = IORESOURCE_IRQ,
    }
    ```

## LED驱动程序设计
