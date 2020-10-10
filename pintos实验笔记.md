## Project 1

老师指导：**把实验 1 的说明都浏览一遍。特别是文档的模板和提交哪些代码，以及常见问题。而后完成 2.1－2.3 阅读，参考 2.1.2 查看 pintos 源代码结构，阅读分析 pintos 线程结构体，以及线程创建、线程调度等相关的函数。先分析已有代码，结构清楚了再开始设计编写自己的代码。**

1. 实验目的：理解并实现 PintOS 中的线程同步问题



2. 评分要求：**（1.2.2）**

- 设计文档：（**完整、明确、清晰**）

  - 数据结构
  - 算法
  - 同步实现：This section asks about how you chose to synchronize this particular type of activity.
  - 设计理由

- 代码：

  - 重点在于与操作系统相关的设计
  - 与源代码风格一致：注释、语法、间隔等

  

3. 提交要求：

- 将 threads.tmpl 复制到pintos/src/threads/DESIGNDOC 中去



4. pintos 文件结构：

![截屏2020-10-10 16.59.27](/Users/kxw/Library/Application Support/typora-user-images/截屏2020-10-10 16.59.27.png)



5. 阅读重点：（**帮助理解源代码**）

**from Section A.1 [Pintos Loading], page 58 through Sec- tion A.5 [Memory Allocation], page 75, especially Section A.3 [Synchronization], page 66, Appendix B [4.4BSD Scheduler], page 91.**



6. 已有实现：线程创建、线程完成、线程调度、同步方式



7. 理解线程：

- thread_create()、没有线程准备好时 run idle()
- threads/switch.S 实现线程调度 switch_threads()   
- **Warninig:**  内核无法检测栈的溢出，局部变量声明无法过大，详见Section 5
- loader.S / loader.h : 在磁盘上寻找内核并将其加载至内存中，跳转至 start.S
- start.S :  基本配置操作，如内存保护、CPU上的32位操作指令
- init.c / init.h : 内核初始化程序（内核的 main 函数）
- thread.c / threads.h : 基本线程支持



8. 理解同步：

- 掌握何时使用何种手段解决同步问题（信号量、锁、条件变量、关闭中断）

- **? ? ? turn of interrupts**

  ![截屏2020-10-10 17.11.43](/Users/kxw/Library/Application Support/typora-user-images/截屏2020-10-10 17.11.43.png)

  

9. 具体需求

   ![截屏2020-10-10 20.15.18](/Users/kxw/Library/Application Support/typora-user-images/截屏2020-10-10 20.15.18.png)

- 重新实现 timer_sleep()：

  源代码中循环确认当前时间并调用 thread_yield() 一段时间（等待），修改后使其避免 **busy waiting  **

- 实现优先调度:

  实现 thread_set_priority() 和 thread_get_priority()

  - 当准备运行线程队列中存在比当前线程优先级高的线程时，当前线程立刻让渡 CPU
  - 优先级高的先获取资源
  - 线程随时会调整自己的优先级
  - 初始优先级作为参数传递给 thread_create()
  - **priority inversion**（2.2.3）：优先级捐献 （嵌套情况特殊考虑）

- 实现高级调度：（类似4.4BSD）

  - 不进行优先级捐献
  - pintos 启动时默认优先调度，通过输入指令来切换为高级调度
  - 线程不直接控制自身优先级

  

10. Appendix A

#### Loading

  loader 被加载至内存中，

​	进而被执行去加载内核程序进入内存，

​	并最后跳转到内核程序开头 start 准备执行。

-->  start 启用 A20 line 以便 Pintos 访问更多内存，

​	并创建基本的页表，

​	加载 CPU 控制寄存器、段寄存器，开启保护模式，

​	禁用中断，

​	调用 main()

--> main 调用 bss_init() 清空内核中的 BSS（用以初始化变量），

​	调用 read_command_line() 和 parse_options() 读入分析指令，

​	调用thread_init() 初始化线程系统，初始化控制台，

​	初始化内存系统：palloc_init() 初始化内核页分配器、malloc_init() 初始化随机内存分配器、paging_init() 初始化内核的页表，

​	初始化中断系统：intr_init() 初始化中断描述表、timer_init() 和 kdb_init() 初始化计时中断和键盘中断，

​	input_init() 初始化输入流，

​	初始化调度程序和线程 thread_init()：创建 idle 线程和启用中断、进而 serial_init_queue() 恢复 I/O ，

​	timer_calibrate() 校准时间延迟

--> run_actions() 执行命令行命令

ps:

- use  *TAGS*  **? ? ?**
- loader 只是一段引导程序，不属于内核部分，它先被加载至内存中，进而被执行去加载内核程序进入内存。



#### Threads

数据结构：struct thread（可作自行修改）

- tid_t tid：唯一、tid_t 默认为 int 、递增

- enum thread_status status ：级联状态四选一

  ![截屏2020-10-10 21.38.53](/Users/kxw/Library/Application Support/typora-user-images/截屏2020-10-10 21.38.53.png)

- uint8_t 8stack ：切换线程时保存状态

线程函数

线程转换： schedule() 函数调用 switch_threads() 函数决定下一个运行的线程，并将前一个运行的线程状态改变，再调用 thread_schedule_tail() 标记新线程的状态。

