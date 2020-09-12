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
/*
socketpair注解：
socketpair() 函数跟 pipe() 函数是类似的，也只能在同个主机上具有亲缘关系的进程间通信，但 pipe() 创建的匿名管道是半双工的，
而 socketpair() 可以认为是创建一个全双工的管道。

当 socketpair() 执行成功时，sv[2]这两个套接字具备下列关系：
向sv[0]套接字写入数据，将可以从sv[1]套接字中读取到刚写入的数据；同样，向sv[1]套接字写入数据，也可以从sv[0]中读取到写入的数据。
通常，在父、子进程通信前，会先调用socketpair方法创建这样一组套接字，在调用fork方法创建出子进程后，将会在父进程中关闭sv[1]套接字，仅使用sv[0]套接字用于向子进程发送数据以及接收子进程发送来的数据：而在子进程中则关闭sv[0]套接字，仅使用sv[1]套接字既可以接收父进程发来的数据，也可以向父进程发送数据。
```

```C
/* 
fcntl(FD_CLOEXEC) 注解：
 用来设置文件的close-on-exec状态标准 
 在exec()调用后，close-on-exec标志为0的情况下，此文件不被关闭；非零则在exec()后被关闭 
 默认close-on-exec状态为0。 通过FD_CLOEXEC设置后为非0
 
fcntl(ngx_processes[s].channel, F_SETFD, FD_CLOEXEC) 的意思是当Master父进程执行了exec()调用后，关闭socket； 也就是平滑升级时新的master不希望与老的worker通信；谁的儿子谁负责(谁fork的谁维护)！   
```

```C
/*
SO_REUSEADDR: 
【表象】允许在一个应用程序可以把 n 个套接字绑在一个端口上而不出错。同时，这 n 个套接字发送信息都正常，没有问题。但是，这些套接字并不是所有都能读取信息，只有最后一个套接字会正常接收数据。
【用途】防止服务器重启时之前绑定的端口还未释放或者程序突然退出而系统没有释放端口。这种情况下如果设定了端口复用，则新启动的服务器进程可以直接绑定端口。如果没有设定端口复用，绑定会失败，提示ADDR已经在使用中——那只好等等再重试了，麻烦！

https://www.jianshu.com/p/711be2f1ec6a
```



### 4. 疑问

```C
1: 为什么mater父进程不关闭 ngx_processes[s].channel[1] ? (他使用channel[0]的)
2：没有看到worker兄弟进程之间的通信，他们通信做什么呢？
```

### 5.  关键函数

#### ngx_init_cycle   master调用

```C
//1.时间、正则、错误日志、ssl等初始化 			ngx_timezone_update()
//2.读入命令行参数
//3.OS相关初始化
//4.读入并解析配置												ngx_conf_param(&conf),  ngx_conf_parse(), environ
//5.核心模块初始化
//6.创建各种临时文件和目录									ngx_create_paths()...
//7.创建共享内存												 cycle->shared_memory.part
//8.打开listen的端口											设置TCP_DEFER_ACCEPT和.open标记, ngx_open_listening_sockets()
//9.所有模块初始化
//10.启动worker进程
```


#### ngx_master_process_cycle 

master进程的主函数。如果是单进程模式，则走 ngx_single_process_cycle。

```C
//1: 信号处理
//2: 进程名称设置
//3: 启动worker进程
//4: 信号的延迟处理.   sigprocmask()  -- sigsuspend()
// 0: master进程 主循环
        // 0.1 Nginx自己的时间处理
				// 0.2 收到子进程退出的信号.
				// 0.3 处理mater退出流程.
				// 0.4 收到sigint 信号. (退出信号)
				// 0.5 收到reconfig信号. (启动新的worker读取新的配置)
								// 0.5.1 热升级判断和处理，如果是热升级从这里退出,等下个循环走到热升级处理流程中去。
								// 0.5.2 创建新的worker进程.
								// 0.5.3 sleep(100) 等待新进程启动
								// 0.5.4 向老的worker进程(旧配置)发信号,让他们自己退出.	ngx_processes[s].just_spawn标记新老worker
				// 0.6 收到reopen信号.
				// 0.7 ngx_change_binary 热升级信号
				// 0.8 ngx_noaccept ,不再accept.

```

#### ngx_single_process_cycle

收到下面4个信号时，会变成singel模式。 他并不是一个正常的工作终态. Stop，quit，reopen, reload 

```C
/* 
1. ngx_set_environment() 多进程下在worker中设置的.
2. 核心模块初始化.
0. 主循环	
				0.1 Nginx自己的时间处理
				0.2 处理重新加载配置文件
				0.3 处理重新打开文件
				0.4 处理信号(eg：退出信号)
    0.5 处理事件
```

#### ngx_open_listening_sockets 

master 打开监听套接字的关键函数.

```C
/*
1: 整体尝试5次. 因为: 端口被bind?
2: fd != -1. continue.   // 对应重试的时候部分成功的情况。 还有就是fd是热升级集成过来的,则啥也不用做。
3： socket(). 
4:  setsockopt( SO_REUSEADDR )
5:  ngx_nonblocking()
6： bind()
7： UDP套接字到此初始化完毕
8： listen()
9:	ls[i].listen = 1 
10: 如果有创建失败的情况，则睡眠500ms, 再去尝试. 
```

#### ngx_spawn_process

```C
/*
1. respawn < 0 , 当前进程需要重启。 (比如 进程假死的情况)。 大于0是ngx_process_slot，小于0是进程重启的原因.
2: 创建进程间通信的channel.
3: pid = fork(),  子进程直接执行自己的main了.
4：[master进程]更新全局进程管理结构 ngx_processes
5：[master进程]ngx_last_process
```



### 6. 全局变量

```C
/*
1. ngx_process   			     全局变量标记进程模式(单进程，mater，worker，helper,config_tester)
2. ngx_process_slot        全局变量，新子进程所在的slot
3. ngx_processes[];				 存储所有子进程的数组  ngx_spawn_process中赋值
4: ngx_pid								 进程的PID
5: ngx_last_process				 最大的进程槽ID，优化手段.
```



### 7. TODO

```C
/*
1. 信号分析
2. 热升级分析
3. 退出流程
```

 ## 8. 疑问

```C
/*
1: tcp listen 端口 什么时候加入epool的 ？
```



