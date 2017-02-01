## Lesson 1 gcc+gdb+makefile
### 1.gcc
* `gcc hello.c -o hello` - 编译时指定可执行文件名
* `gcc -c hello.c` - 不连接生成可执行文件 只编译生成目标文件(.o)
* `gcc -g hello.c` - 编译时可执行文件加入调试信息 可使用gdb调试程序
* `gcc -O hello.c -o hello` - 编译后的可执行文件执行效率更高,但编译耗时更长
* `gcc -O2 hello.c -o hello` - 比上述更高优化等级
* `time ./hello` - 执行程序时显示程序运行耗时统计
* `-I` - 编译时指定自定义头文件路径
* `-L` - 编译时指定自定义库文件路径
* `-static` - 指定编译为静态连接可执行文件 (默认为动态连接)
* `-Wall` - 生成所有警告信息
* `-w` - 不生成任何警告信息 (默认生成部分警告信息)
* `-D` - 编译时加入一个宏定义 (相当于额外增加一个宏定义)

### 2.gdb
* `gdb hello` - 开始调试可执行文件hello (该可执行文件需加入调试信息)
* `break(b) 函数名` - 在指定函数名的入口加入一个断点
* `break(b) 行号` - 在指定行加入一个断点
* `break(b) 文件名:行号` - 在指定源代码文件的指定行加入一个断点
* `break(b) 行号 if` - 在指定行加入一个条件断点 满足一定条件才触发断点
* `info break(i b)` - 查看当前已加入的断点
* `delete(d) 断点编号` - 删除指定编号的断点
* `print(p) 变量名` - 查看指定变量的当前值
* `watch(w) 变量名` - 监视指定变量的值
* `run(r)` - 开始运行调试程序
* `next(n)` - 单步调试
* `step(s)` - 单步调试
* `continue(c)` - 继续运行调试程序
* `finish(f)` - 结束运行调试程序
* `list(l)` - 查看调试程序代码
* `quit(q)` - 退出调试

### 3.makefile
* 规则格式:

```
targets: prerequisites  

[tab]command
```
* 命令前必须为tab键
* 第一条规则的目标为makefile的最终目标

example：

```
hello: main.o func1.o func2.o
	gcc main.o func1.o func2.o -o hello

main.o: main.c
	gcc -c main.c

func1.o: func1.c
	gcc -c func1.c

func2.o: func2.c
	gcc -c func2.c	
```
* makefile 执行方法 (两种):

	1. `make` - makefile文件名默认为makefile或Makefile
	2. `make -f 文件名` - 当makefile的文件名不为默认文件名时 可使用这种方法(默认文件名也可以)

* 没有任何依赖只有执行命令的目标称为伪目标, 如:

```
clean:
	rm -f xxx
```
* 变量
	1. $^ - 所有依赖文件
	2. $@ - 目标文件
	3. $< - 第一个依赖文件

```
hello: $(obj) 
	gcc $(obj) -o hello
```
```
hello: main.o func1.o func2.o
	gcc $^ -o $@
```

* 取消回显

```
hello: hello.c
	@gcc hello.c -o hello
```















