# Linux内核开发基础

![](image/Linux内核代码构架图.png)


## Linux内核简介

* linux体系结构
	1. 用户空间
	2. 内核空间

![](image/Linux体系结构.png)

* Linux划分<b>用户控件</b>与<b>内核空间</b>的作用
> 现代CPU通常实现了不同的工作模式,具有不同级别的硬件权限.
> 
> Linux系统利用CPU这一特性,使用其中的两级来分别运行内核程序与应用程序,这样<b>使得操作系统本身得到充分的保护</b>.
> 
> 内核空间与用户空间是程序执行的两种不同状态,通过<b>系统调用</b>和<b>硬件中断</b>能够完成从用户空间到内核空间的转移.

* Linux内核架构

![](image/Linux内核架构.png)

  1. System Call Interface (SCI) - 系统调用接口
  2. Process Management (PM) - 进程管理
  3. Memory Management (MM) - 内存管理
  4. Network Stack - 网络协议栈
  5. Virtual File System (VFS) - 虚文件系统
  6. Device Drivers (DD) - 设备驱动
  7. Arch - CPU相关代码



## Linux内核源代码



## Linux内核配置与编译



## Linux内核模块开发




