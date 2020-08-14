### 1.  master 生产 worker的过程

```C
ngx_start_worker_processes()
    for i to n ...	//创建N个worker，个数在配置文件中设定
        ngx_spawn_process(ngx_worker_process_cycle) 
                socketpair(ngx_processes[s].channel)	// 创建全双工套接字用于进程间通信
                ngx_nonblocking()		//channel设置为非阻塞
                ioctl(ngx_processes[s].channel[0], FIOASYNC, &1)	//channel设置为异步
                fcntl(ngx_processes[s].channel[0], F_SETFD, FD_CLOEXEC)  //设置为：平滑升级时，这些channel自动关闭
                ngx_channel = ngx_processes[s].channel[1];			// 全局变量 新子进程的通信channel
                ngx_process_slot = s; 								// 全局变量，新子进程所在的slot
                pid = fork()										// !!! 启动worker进程 
                		if (pid == 0) ngx_worker_process_cycle()	//子进程永远不会退出这个函数了！	
                ngx_processes[ngx_process_slot].pid = pid;			// 保存子进程的pid到全局进程表
		ngx_pass_open_channel()		// 逐个告诉所有的儿子,有新儿子诞生 他的通信端口是xxx.
       
ngx_worker_process_cycle()	//worker的主循环
	
```


### 2.  worker 进程的启动过程

```C
ngx_worker_process_cycle()	//子进程的 main()
        ngx_worker_process_init()
                ngx_set_environment()
                setpriority(ccf->priority)	//系统调用，设置现场优先级, 也就是内核nice值.
                setrlimit()
                ngx_get_cpu_affinity(n)	//从配置文件中读取n号worker对应的cpu masker
                ngx_setaffinity()		//调用sched_setaffinity()做进程和CPU的绑定
                sigemptyset(&set);		//信号处理
                ngx_add_channel_event(ngx_channel,READ,ngx_channel_handler)		// 底层是epoll add,监听可读事件
                		ngx_channel_handler()
                    			case NGX_CMD_OPEN_CHANNEL: 	//有新兄弟进程启动,记录通信端口,用父进程的通道和他对话
								case NGX_CMD_QUIT:		 	//退出信号 ngx_quit 置为1, 
								case NGX_CMD_TERMINATE:		//退出信号 ngx_terminate 置为1,
								case NGX_CMD_REOPEN:		//退出信号 ngx_reopen 置为1,
								case NGX_CMD_CLOSE_CHANNEL:	//有兄弟死掉啦,关闭和他的对话端口.
		
        //worker的主循环 主循环 主循环        
        for(;;)					
                if (ngx_exiting) ...
                ngx_process_events_and_timers(cycle);
                if (ngx_terminate) ...
                if (ngx_quit) ...
                if (ngx_reopen) ...
```





### 3. 注解

```C
socketpair注解：
socketpair() 函数跟 pipe() 函数是类似的，也只能在同个主机上具有亲缘关系的进程间通信，但 pipe() 创建的匿名管道是半双工的，
而 socketpair() 可以认为是创建一个全双工的管道。

当 socketpair() 执行成功时，sv[2]这两个套接字具备下列关系：
向sv[0]套接字写入数据，将可以从sv[1]套接字中读取到刚写入的数据；同样，向sv[1]套接字写入数据，也可以从sv[0]中读取到写入的数据。
通常，在父、子进程通信前，会先调用socketpair方法创建这样一组套接字，在调用fork方法创建出子进程后，将会在父进程中关闭sv[1]套接字，仅使用sv[0]套接字用于向子进程发送数据以及接收子进程发送来的数据：而在子进程中则关闭sv[0]套接字，仅使用sv[1]套接字既可以接收父进程发来的数据，也可以向父进程发送数据。
```

```
 fcntl(FD_CLOEXEC) 注解：
 用来设置文件的close-on-exec状态标准 
 在exec()调用后，close-on-exec标志为0的情况下，此文件不被关闭；非零则在exec()后被关闭 
 默认close-on-exec状态为0。 通过FD_CLOEXEC设置后为非0
 
fcntl(ngx_processes[s].channel, F_SETFD, FD_CLOEXEC) 的意思是当Master父进程执行了exec()调用后，关闭socket； 也就是平滑升级时新的master不希望与老的worker通信；谁的儿子谁负责(谁fork的谁维护)！   
```



### 4. 疑问

```C
1: 为什么mater父进程不关闭 ngx_processes[s].channel[1] ? (他使用channel[0]的)
2：没有看到worker兄弟进程之间的通信，他们通信做什么呢？
```

