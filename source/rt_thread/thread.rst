线程
############

线程是实现任务的载体，是RT-Thread中最基本的调度单元，他描述了一个任务执行的运行环境。

系统中共存在两类线程，分别是 **系统线程** 和 **用户线程** 。

RT-Thread的线程调度器是抢占式的，主要工作就是从就绪线程列表中查找最高优先级线程，保证优先级最高的线程能够被运行。

.. code-block:: C
   :linenos:
   :caption: 线程控制块的定义

    struct rt_thread
    {
        /* rt object */
        char        name[RT_NAME_MAX];                      /** 线程名字 */
        rt_uint8_t  type;                                   /**< 对象类型 */
        rt_uint8_t  flags;                                  /**< 标志位 */

    #ifdef RT_USING_MODULE
        void       *module_id;                              /**< id of application module */
    #endif

        rt_list_t   list;                                   /**< 对象列表 */
        rt_list_t   tlist;                                  /**< 线程列表 */

        /* 栈指针和入口指针 */
        void       *sp;                                     /**< 栈指针 */
        void       *entry;                                  /**< 入口函数指针 */
        void       *parameter;                              /**< 参数 */
        void       *stack_addr;                             /**< 栈地址指针 */
        rt_uint32_t stack_size;                             /**< 栈大小 */

        /* 错误代码 */
        rt_err_t    error;                                  /**< 线程错误代码 */

        rt_uint8_t  stat;                                   /**< 线程状态 */

    #ifdef RT_USING_SMP
        rt_uint8_t  bind_cpu;                               /**< thread is bind to cpu */
        rt_uint8_t  oncpu;                                  /**< process on cpu*/
        rt_uint16_t scheduler_lock_nest;                    /**< scheduler lock count */
        rt_uint16_t cpus_lock_nest;                         /**< cpus lock count */
        rt_uint16_t critical_lock_nest;                     /**< critical lock count */
    #endif /*RT_USING_SMP*/

        /* 优先级 */
        rt_uint8_t  current_priority;                       /**< 当前优先级 */
        rt_uint8_t  init_priority;                          /**< 初始优先级 */
    #if RT_THREAD_PRIORITY_MAX > 32
        rt_uint8_t  number;
        rt_uint8_t  high_mask;
    #endif
        rt_uint32_t number_mask;

    #if defined(RT_USING_EVENT)
        /* thread event */
        rt_uint32_t event_set;
        rt_uint8_t  event_info;
    #endif

    #if defined(RT_USING_SIGNALS)
        rt_sigset_t     sig_pending;                        /**< the pending signals */
        rt_sigset_t     sig_mask;                           /**< the mask bits of signal */

    #ifndef RT_USING_SMP
        void            *sig_ret;                           /**< the return stack pointer from signal */
    #endif
        rt_sighandler_t *sig_vectors;                       /**< vectors of signal handler */
        void            *si_list;                           /**< the signal infor list */
    #endif

        rt_ubase_t  init_tick;                              /**< 线程初始化计数值 */
        rt_ubase_t  remaining_tick;                         /**< 线程剩余计数值 */

        struct rt_timer thread_timer;                       /**< 内置线程计数器 */

        void (*cleanup)(struct rt_thread *tid);             /**< 用户退出清除函数 */

        /* light weight process if present */
    #ifdef RT_USING_LWP
        void        *lwp;
    #endif

        rt_ubase_t user_data;                             /**< 用户数据 */
    };




**线程栈**：RT-Thread线程具有独立的栈，当进行线程切换时，会将当前线程的上下文保存在栈中，当线程要恢复运行是，再从栈中读取上下文信息进行恢复。

线程栈还用来存放函数中的局部变量。



**线程状态**：

- 初始状态::

    线程不参与调度。

- 就绪状态::

    线程按照优先级排队，等待被执行；一旦当前线程运行完毕让出处理器，操作系统就会马上寻找最高优先级的就绪状态线程运行。

- 运行状态::

    线程当前正在运行，在**单核系统**中，只有rt_thread_self()函数返回的线程处于运行状态；
    在多核系统中，可能就不止一个线程处于运行状态。

- 挂起状态::

    也称堵塞态。在挂起状态下，线程不参与调度。可能是因为资源不可用而挂起等待，或线程主动延时一段时间而挂起。

- 关闭状态::

    当线程运行结束时将处于关闭状态。关闭状态的线程不参与线程的调度。

.. code-block:: c
   :linenos:
   :caption: 线程状态

    #define RT_THREAD_INIT                  0x00                /**< 初始状态 */
    #define RT_THREAD_READY                 0x01                /**< 就绪状态 */
    #define RT_THREAD_SUSPEND               0x02                /**< 挂起状态 */
    #define RT_THREAD_RUNNING               0x03                /**< 运行状态 */
    #define RT_THREAD_BLOCK                 RT_THREAD_SUSPEND   /**< 新增的状态，将挂起状态再细分，分为锁死状态 */
    #define RT_THREAD_CLOSE                 0x04                /**< 关闭状态 */




**线程优先级**：RT-Thread最大支持256个线程优先级（0~255），数值越小的优先级越高，0为最高优先级。

在一些资源比较紧张的系统中，可以根据实际情况选择只支持8个或32个优先级的系统配置。

对于ARM Cortex-M系列，普遍采用32个优先级。最低优先级默认分配给空闲线程使用，用户一般不使用。

.. code-block:: c
   :linenos:
   :caption: 创建线程示例

    static rt_thread_t tid1 = RT_NULL;

    #define THREAD_PRIORITY     25 //线程优先级
    #define THREAD_STACK_SIZE   512 //线程栈大小
    #define THREAD_TIMESLICE    5   //线程时间片大小

    //线程1入口函数
    void thread1_entry(void* paramenter)
    {
        rt_uint32_t count = 0;

        while(1)
        {
            rt_kprintf("thread1 count: %d\n", count++); //打印字符
            rt_thread_delay(500); //延时0.5秒
        }
    }

    static char thread2_stack[1024];
    static struct rt_thread thread2;

    void thread2_entry(void* paramenter)
    {
        rt_uint32_t count = 0;

        for(count = 0;count < 10; count++)
        {
            rt_kprintf("thread2 count:%d\n", count);
        }
    }

    int main(void)
    {
    //    void thread1_entry();

        //动态创建线程1
        tid1 = rt_thread_create("thread1",
                                thread1_entry,
                                RT_NULL,
                                THREAD_STACK_SIZE,
                                THREAD_PRIORITY,
                                THREAD_TIMESLICE);

        //静态初始化线程2
        rt_thread_init(&thread2,
                    "thread2",
                    thread2_entry,
                    RT_NULL,
                    &thread2_stack[0],
                    sizeof(thread2_stack),
                    THREAD_PRIORITY - 1,
                    THREAD_TIMESLICE);

        if(tid1 != RT_NULL)
        {
            rt_kprintf("thread1 has been init.\n");
            rt_thread_delay(1000);
            rt_kprintf("thread1 start!\n");
            rt_thread_startup(tid1);    //启动线程1
        }
        else {
            rt_kprintf("thread1 init error!\n");
        }

        rt_thread_startup(&thread2);//启动线程2


    }

.. code-block:: c
   :linenos:
   :caption: 线程时间片轮转调度示例

    #define THREAD_PRIORITY     20
    #define THREAD_STACK_SIZE   1024
    #define THREAD_TIMESLICE    10


    static void thread_entry(void* parameter)
    {
        rt_uint32_t value;
        rt_uint32_t count = 0;

        value = (rt_uint32_t)parameter;
        while(1)
        {
            if(0 == (count % 5))
            {
                rt_kprintf("thread %d is running, thread %d count = %d\n", value, value, count);

                if(count > 200)
                    return;
            }

            count++;
        }
    }


    int main(void)
    {
        rt_thread_t tid = RT_NULL;

        tid = rt_thread_create("thread1",
                                thread_entry,
                                (void*)1,
                                THREAD_STACK_SIZE,
                                THREAD_PRIORITY,
                                THREAD_TIMESLICE);

        if(tid != RT_NULL)
        {
            rt_thread_startup(tid);
        }

    tid = rt_thread_create("thread2",
                            thread_entry,
                            (void*)2,
                            THREAD_STACK_SIZE,
                            THREAD_PRIORITY,
                            THREAD_TIMESLICE-5);
    if(tid != RT_NULL)
        {
            rt_thread_startup(tid);
        }

    return 0;
    }

.. code-block:: C
   :linenos:
   :caption: 线程调度钩子示例
    #define THREAD_STACK_SIZE   1024
    #define THREAD_PRIORITY     20
    #define THREAD_TIMESLICE    10

    volatile rt_uint32_t count[2]; //锟斤拷锟矫匡拷锟斤拷叱痰募锟斤拷锟斤拷锟�

    //线程1和线程2共同的入口函数，只是传递的参数不一样
    static void thread_entry(void* parameter)
    {
        rt_uint32_t value;

        value = (rt_uint32_t)parameter;

        while(1)
        {
            rt_kprintf("thread %d is runing \n", value);
            rt_thread_delay(1000);
        }
    }

    static rt_thread_t tid1 = RT_NULL;
    static rt_thread_t tid2 = RT_NULL;


    //调度器钩子函数
    //from -----------表示系统所要切换出的线程控制块指针
    //to -------------表示系统所要切换到的线程控制块指针
    static void hook_of_schedular(struct rt_thread* from, struct rt_thread* to)
    {
        rt_kprintf("from: %s --> to:%s \n", from->name, to->name);
    }


    int sceduler_hook(void)
    {
        //设置调度器钩子函数为hook_of_schedular
        rt_scheduler_sethook(hook_of_schedular);

        //创建线程1
        tid1 = rt_thread_create("thread1",
                                thread_entry,
                                (void*)1,
                                THREAD_STACK_SIZE,
                                THREAD_PRIORITY,
                                THREAD_TIMESLICE);

        if(tid1 != RT_NULL)
            rt_thread_startup(tid1);

        //创建线程2
        tid2 = rt_thread_create("thread2",
                                thread_entry,
                                (void*)2,
                                THREAD_STACK_SIZE,
                                THREAD_PRIORITY,
                                THREAD_TIMESLICE-5);
        if(tid2 != RT_NULL)
                rt_thread_startup(tid2);

        return 0;
    }


**小结**：

1. 普通线程不能陷入死循环操作，必须要有让出CPU使用权的动作。
2. 线程调度基于优先级抢占的方式进行，优先级相同的线程通过时间片轮询执行
3. 动态线程的创建与删除调用接口是rt_thread_create和rt_thread_delete；静态线程的初始化与脱离调用接口是rt_thread_init和rt_thread_detach。**注意**，由于能执行完毕的线程会被系统自动删除，不推荐用户使用删除/脱离线程接口。
