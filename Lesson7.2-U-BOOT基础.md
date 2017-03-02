# U-BOOT基础

## 1. BootLoader介绍

 * 什么是BootLoader?

> 一个嵌入式系统从软件角度来看分为三个层次
> 1. 引导加载程序 - 包括固化在固件中的boot程序(可选)和BootLoader程序
> 2. Linux内核 - 特定于嵌入式平台的定制内核
> 3. 文件系统 - 包括了系统命令和应用程序

一个同时装有<b>BootLoader</b>、<b>内核启动参数</b>、<b>内核映像</b>和<b>根文件系统</b>映像的固态存储设备的典型空间分配结构图:

![](image/嵌入式Linux软件层次空间分配结构图.png)

BootLoader主要用于完成<b>整个系统的加载启动任务</b>, 是在操作系统运行之前所运行的<b>一段小程序</b>。通过这段小程序,可以<b>初始化硬件设备</b>,从而将系统软硬件环境带到一个合适的状态,以便为最终<b>调用操作系统</b>做好准备。


* BootLoader安装

系统加电或复位后,所有CPU通常都从CPU制造商预先设定的地址开始执行。而嵌入式系统则将固态存储设备(如:FLASH)安排在这个地址上,而BootLoader程序又安排在固态存储器的最前端,这样就能保证在系统加电后,CPU首先执行BootLoader程序.

* BootLoader启动流程

BootLoader大多采用两阶段,即stage1和stage2:

  1. stage1
    - 完成初始化硬件
    - 为stage2准备内存空间
    - 并将stage2复制到内存中
    - 设置堆栈
    - 然后跳转到stage2.
  2. stage2
    - 初始化本阶段要使用到的硬件设备
    - 将内核映像和根文件系统映像从flash上读到RAM中
    - 调用内核

![](image/Bootloader运行时的内存分布.png)

* Bootloader工作模式

  1. 启动模式
    >　从目标机上的某个固态存储设备上将操作系统自动加载到RAM中运行，整个过程并没有用户的介入

  2. 下载模式
    > 目标机上的BootLoader将通过串口或网络等从主机下载文件, 然后控制启动流程.

## 2. 交叉工具链

* 安装交叉工具链
  1. 解压工具链到某一目录下;
  2. 修改`/etc/profile`, 添加`pathmunge/4.5.1/bin`
  3. 启动生效对环境变量的修改: `source /etc/profile`

* 使用交叉工具链
  1. 编译器: `arm-linux-gcc`

    `arm-linux-gcc hello.c -o hello`

  2. 反汇编工具: `arm-linux-objdump`

    `arm-linux-objdump -D -S hello`

  3. ELF文件查看工具: `arm-linux-readelf`

    ```
    arm-linux-readelf -a hello
    arm-linux-readelf -d hello
    ```

## 3. U-Boot介绍

* U-Boot作用

U-Boot是德国DENX小组开发的用于多种嵌入式CPU的BootLoader程序, 不仅支持嵌入式Linux系统的引导, 还支持VxWorks, QNX等多种嵌入式操作系统.

* U-Boot下载

`ftp://ftp.denx.de/pub/u-boot`

* U-Boot目录结构
  - Board - 和开发板有关的文件.
  - Common - 实现Uboot支持的命令
  - Cpu - 与特定CPU架构相关的代码.
  - Disk - 对磁盘的支持
  - Doc - 参考文档
  - Drivers - Uboot支持的设备驱动程序.
  - Fs - 文件系统的支持
  - Include - Uboot使用的头文件.
    - configs - 与开发板相关的配置头文件.
    - asm - 与CPU体系结构相关的头文件.
  - Net - 与网络协议栈相关的代码.
  - Tools - Uboot的工具

* U-Boot编译
  1. 选择要使用的Board

  `$make smdk6410_config`

  2. 编译生成u-boot.bin

  `$make CROSS_COMPILE=arm-linux-`

## 4. U-Boot命令

* 常用命令

  - `help` - 查看当前单板所支持的命令
  - `printenv` - 查看环境变量
  - `setenv` - 添加、修改、删除环境变量

    `setenv name value` - 添加/修改

    `setenv name` - 删除

  - `saveenv` - 保存环境变量至flash中
  - `tftp` - 通过网络下载文件(需先配置好网络)

    `tftp c0800000 uImage` - 把server中服务目录下的uImage通过TFTP读入到0xc0800000处

  - `md` - 显示内存区的内容

    `md [.b, .w, .l] address`
  - `mm` - 修改内存, 地址自动递增

    `mm [.b, .w, .l] address`
  - `nand info` - 查看NandFlash大小等信息
  - `nand erase` - 擦除NandFlash

    `nand erase start length`
  - `nand write` - 向NandFlash写入数据

    `nand write [内存地址] [flash地址] length`
    `nand write.i c0800000 100000 600000`

  - `nand read` - 从NandFlash读出数据

    `nand read [内存地址] [flash地址] length`
    `nand read.i c0800000 100000 600000`

  - `go` - 执行内存中的二进制代码(简单跳转到指定地址)

    `go addr [arg...]`

  - `bootm` - 执行内存中的二进制代码(要求二进制代码有固定格式的文件头)

    `bootm [addr [arg...]]`

  - `bdinfo` - 显示开发板信息(包括内存地址和大小、时钟频率、MAC地址等)

* 设置自动启动系统

```
setenv bootcmd tftp c0008000 uImage \; bootm c0008000
saveenv
```


* NorFlash 和 NandFlash区别:

  1. NorFlash读取速度比NandFlash快
  2. NandFlash的写入速度比NorFlash快
  3. NandFlash的擦除速度更快
  4. NandFlash的成本更低
  5. NandFlash的擦除寿命更长(百万次), NorFlash(10万次)
  6. NandFlash通常采用自己独立编制, NorFlash则采用统一编制.

  基于以上特性, NorFlash更适合用于存储代码, NandFlash更适合用于存储大容量数据.

## end
