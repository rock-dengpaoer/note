线程间同步
#############

信号量
*******************

.. code-block:: C
   :caption: 信号量的使用

    #define THREAD_PRIORITY     25
    #define THREAD_TIMESLICE    5

    //指向信号量的指针
    static rt_sem_t dynamic_sem = RT_NULL;

    static char thread1_stack[1024];
    static struct rt_thread thread1;

    static void rt_thread1_entry(void* parameter)
    {
        static rt_uint8_t count = 0;

        while(1)
        {
            if(count <= 100)
            {
                count++;
            }
            else {
                return;
            }
            if(0 == (count % 10))
            {
                rt_kprintf("t1 release a dynamic semaphore.\n");
                rt_sem_release(dynamic_sem);//释放一次信号量
            }
        }
    }

    static char thread2_stack[1024];
    static struct rt_thread thread2;

    static void rt_thread2_entry(void* parameter)
    {
        static rt_err_t result;
        static rt_uint8_t number = 0;
        while(1)
        {
            //永久方式等待信号量，获取到信号量，则执行number自加的操作
            result = rt_sem_take(dynamic_sem, RT_WAITING_FOREVER);
            if(result != RT_EOK)
            {
                rt_kprintf("t2 take a dynamic semaphore, faild.\n");
                rt_sem_delete(dynamic_sem);
                return;
            }
            else {
                number++;
                rt_kprintf("t2 take a dynamic semaphore. number = %d\n", number);
            }
        }
    }


    //信号量示例的初始化
    int semaphore_sample(void)
    {
        //创建一个动态信号量，初始化是0，先进先出
        dynamic_sem = rt_sem_create("dsm", 0, RT_IPC_FLAG_FIFO);

        if(dynamic_sem == RT_NULL)
        {
            rt_kprintf("create dynamic semaphore failed.\n");
            return -1;
        }
        else {
            rt_kprintf("create done. dynamic semaphore value = 0.\n");
        }

        rt_thread_init(&thread1, "thread1",
                        rt_thread1_entry,
                        RT_NULL,
                        &thread1_stack[0],
                        sizeof(thread1_stack),
                        THREAD_PRIORITY,
                        THREAD_TIMESLICE);

        rt_thread_startup(&thread1);

        rt_thread_init(&thread2, "thread2",
                            rt_thread2_entry,
                            RT_NULL,
                            &thread2_stack[0],
                            sizeof(thread1_stack),
                            THREAD_PRIORITY-1,
                            THREAD_TIMESLICE);

        rt_thread_startup(&thread2);

        return 0;
    }

.. code-block:: C
   :caption: 生产者消费者例程

    #define THREAD_PRIORITY     6
    #define THREAD_STACK_SIZE   512
    #define THREAD_TIMESLICE    5

    //定义最大能够产生5个元素
    #define MAXSEX 5

    //用于放置生产的整数数组
    rt_uint32_t array[MAXSEX];

    //指向生产者、消费者在array数组中的读写位置
    static rt_uint32_t set,get;

    static rt_thread_t producer_tid = RT_NULL;
    static rt_thread_t consumer_tid = RT_NULL;

    struct rt_semaphore sem_lock;
    struct rt_semaphore sem_empty, sem_full;

    //生产者线程入口函数
    void producer_thread_entry(void* parameter)
    {
        int cnt = 0;

        while(cnt < 10)
        {
            rt_sem_take(&sem_empty, RT_WAITING_FOREVER);

            rt_sem_take(&sem_lock, RT_WAITING_FOREVER);
            array[set % MAXSEX] = cnt + 1;
            rt_kprintf("the producer generates a number: %d\n", array[set % MAXSEX]);
            set++;
            rt_sem_release(&sem_lock);

            rt_sem_release(&sem_full);
            cnt++;

            rt_thread_delay(20);
        }

        rt_kprintf("the producer exit!\n");
    }

    //消费者入口函数
    void consumer_thread_entry(void *parameter)
    {
        rt_uint32_t sum = 0;

        while(1)
        {
            rt_sem_take(&sem_full, RT_WAITING_FOREVER);

            rt_sem_take(&sem_lock, RT_WAITING_FOREVER);

            sum += array[get % MAXSEX];

            rt_kprintf("the consumer[%d] get a number:%d\n", (get % MAXSEX), array[get % MAXSEX]);
            
            get++;

            rt_sem_release(&sem_lock);
            rt_sem_release(&sem_empty);
            if (get == 10 )
                break;

            rt_thread_mdelay(50);
        }

        rt_kprintf("the consumer sum is : %d\n", sum);
        rt_kprintf("the consumer exit!\n");
    }

    int producer_consumer(void)
    {
        //初始化
        get = 0;
        set = 0;

        //信号量初始化
        rt_sem_init(&sem_lock, "lock", 1, RT_IPC_FLAG_FIFO);
        rt_sem_init(&sem_empty, "empty", MAXSEX, RT_IPC_FLAG_FIFO);
        rt_sem_init(&sem_full, "full", 0, RT_IPC_FLAG_FIFO);

        //生产者线程
        producer_tid = rt_thread_create("producer",
                                        producer_thread_entry,
                                        RT_NULL,
                                        THREAD_STACK_SIZE,
                                        THREAD_PRIORITY -1,
                                        THREAD_TIMESLICE);

        if(producer_tid != RT_NULL)
        {
            rt_thread_startup(producer_tid);//运行线程
        }
        else
        {
            rt_kprintf("create thread producer failed");
            return -1;
        }

        //消费者线程
        consumer_tid = rt_thread_create("consumer",
                                        consumer_thread_entry,
                                        RT_NULL,
                                        THREAD_STACK_SIZE,
                                        THREAD_PRIORITY+1,
                                        THREAD_TIMESLICE);

        if(consumer_tid != RT_NULL)
        {
            rt_thread_startup(consumer_tid);//运行线程
        }
        else
        {
            rt_kprintf("create thread consumer failed");
            return -1;
        }

        return 0;
    }


互斥量
*****************************

互斥量是一种保护共享资源的方法，当一个线程拥有互斥量的时候，可以保护共享资源不被其他线程破坏。

 **例子** ::

    有两个线程，线程1和线程2，线程1对两个number分别进行加1操作；线程2也对两个number进行加1操作，使用互斥量保证线程改变两个number值的操作不被打断。

.. code-block:: C
   :caption: 互斥量例程

    #define THREAD_PRIORITY     8
    #define THREAD_TIMESLICE    5

    static rt_mutex_t dynamic_mutex = RT_NULL;
    static rt_uint8_t number1,number2 = 0;

    static char thread1_stack[1024];
    static struct rt_thread thread1;

    //线程1入口函数
    static void rt_thread_entry1(void *parameter)
    {
        while(1)
        {
            rt_mutex_take(dynamic_mutex, RT_WAITING_FOREVER);
            number1++;
            rt_thread_mdelay(10);
            number2++;
            rt_mutex_release(dynamic_mutex);
        }
    }

    static char thread2_stack[1024];
    static struct rt_thread thread2;

    //线程2入口函数
    static void rt_thread_entry2(void *parameter)
    {
        while(1)
        {
            rt_mutex_take(dynamic_mutex, RT_WAITING_FOREVER);
            if(number1 != number2)
            {
                rt_kprintf("not protect.number1 = %d, number2 = %d\n", number1, number2);
            }
            else {
                rt_kprintf("mutex protect, number1 = number2 is %d\n", number1);
            }

            number1++;
            number2++;
            rt_mutex_release(dynamic_mutex);

            if(number1 >= 50)
                return;
        }
    }

    int mutex_sample(void)
    {
        dynamic_mutex = rt_mutex_create("dmutex", RT_IPC_FLAG_FIFO);//创建互斥量，先进先出

        if(dynamic_mutex == RT_NULL)
        {
            rt_kprintf("create dynamic mutex failed.\n");
            return -1;
        }

        //初始化线程1
        rt_thread_init(&thread1,
                    "thrad1",
                    rt_thread_entry1,
                    RT_NULL,
                    thread1_stack,
                    sizeof(thread1_stack),
                    THREAD_PRIORITY,
                    THREAD_TIMESLICE);

        rt_thread_startup(&thread1);

        //初始化线程2
        rt_thread_init(&thread2,
                        "thrad2",
                        rt_thread_entry2,
                        RT_NULL,
                        thread2_stack,
                        sizeof(thread2_stack),
                        THREAD_PRIORITY-1,
                        THREAD_TIMESLICE);

        rt_thread_startup(&thread2);

        return 0;
    }


互斥量防止线程优先级翻转
******************************

优先级翻转，是指当一个高优先级线程试图通过信号量机智访问共享资源是，如果该信号量已被低优先级线程持有，而这个低优先级线程在运行过程中可能又被一些中等优先级的线程抢占，从而造成高优先级线程被许多较低优先级的线程堵塞，实时性难以得到保证。

在RT-Thread操作系统中，互斥量可以解决优先级翻转问题，使用优先级继承算法实现。优先级继承是通过在线程A尝试获取共享资源而被挂起的期间内，将线程C的优先级提升到线程A的优先级别，从而解决优先级翻转引起的问题。

优先级继承是指，提高某个占有某种资源的低优先级线程的优先级，使之与所有等待该资源的线程中优先级最高的那个线程的优先级相等，然后执行，而当这个低优先级线程释放该资源时，优先级重新回到初始设定。

这个例子创建3个动态线程以检查只有互斥量时，持有的线程优先级是否被调整到等待线程优先级中的最高级

.. code-block:: C
   :caption: 防止优先级翻转例程

    //指向线程控制块的指针
    static rt_thread_t tid1 = RT_NULL;
    static rt_thread_t tid2 = RT_NULL;
    static rt_thread_t tid3 = RT_NULL;
    static rt_mutex_t mutex =RT_NULL;

    #define THREAD_PRIORITY     10
    #define THREAD_STACK_SIZE   512
    #define THREAD_TIMESLICE    5

    //线程1入口函数
    static void thread1_entry(void *parameter)
    {
        //先让低优先级运行
        rt_thread_delay(100);

        //此时thread3持有mutex，而thread2等待持有mutex
    //    检查thread2与threa3的优先级情况

        if(tid2->current_priority != tid3->current_priority)
        {
    //        优先级不同，调试失败
            rt_kprintf("the priority of thread2 is: %d\n", tid2->current_priority);
            rt_kprintf("the priority of thread3 is: %d\n", tid3->current_priority);
            rt_kprintf("test failed");
            return;
        }
        else {
            rt_kprintf("the priority of thread2 is: %d\n", tid2->current_priority);
            rt_kprintf("the priority of thread3 is: %d\n", tid3->current_priority);
            rt_kprintf("test OK！\n");
        }
    }

    //线程2入口函数
    static void thread2_entry(void *parameter)
    {
        rt_err_t result;

        rt_kprintf("the priority of thread2 is %d\n", tid2->current_priority);

    //    让低优先级线程运行
        rt_thread_delay(50);

    //    试图持有互斥锁，此时threa3持有互斥锁，应把thread3的优先级提升到与thread2相同的优先级
        result = rt_mutex_take(mutex, RT_WAITING_FOREVER);

        if(result == RT_EOK)
        {
    //       释放互斥锁
            rt_mutex_release(mutex);
        }
    }

    //线程3入口函数
    static void thread3_entry(void *parameter)
    {
        rt_tick_t tick;
        rt_err_t result;

        rt_kprintf("the priority of thread3 is %d\n", tid3->current_priority);

    //    获取互斥量
        result = rt_mutex_take(mutex, RT_WAITING_FOREVER);
        if(result != RT_EOK)
        {
            rt_kprintf("thread3 take a mutex, failed\n");
        }
    //    做一个长时间的延时，5000ms
        tick = rt_tick_get();
        while(rt_tick_get() - tick < (RT_TICK_PER_SECOND / 2));

    //    释放互斥量
        rt_mutex_release(mutex);
    }

    int pri_inversion(void)
    {
        mutex = rt_mutex_create("mutex", RT_IPC_FLAG_FIFO);

        if(mutex == RT_NULL)
        {
            rt_kprintf("create dynamic mutex failed \n");
            return -1;
        }

        tid1 = rt_thread_create("thread1",
                                thread1_entry,
                                RT_NULL,
                                THREAD_STACK_SIZE,
                                THREAD_PRIORITY-1,
                                THREAD_TIMESLICE);

        if(tid1 != RT_NULL)
            rt_thread_startup(tid1);

        tid2 = rt_thread_create("thread2",
                                    thread2_entry,
                                    RT_NULL,
                                    THREAD_STACK_SIZE,
                                    THREAD_PRIORITY,
                                    THREAD_TIMESLICE);

        if(tid2 != RT_NULL)
                rt_thread_startup(tid2);

        tid3 = rt_thread_create("thread3",
                                    thread3_entry,
                                    RT_NULL,
                                    THREAD_STACK_SIZE,
                                    THREAD_PRIORITY+1,
                                    THREAD_TIMESLICE);
        if(tid3 != RT_NULL)
                rt_thread_startup(tid3);

        return 0;


    }
    //线程3先持有互斥量，而后线程2试图持有互斥量，此时线程3的优先级被提升为和线程2的优先级相同。

事件集
*********************

事件集的工作机制
=====================

特点是实现 **一对多、多对多** 的同步::

    即一个线程与多个事件的关系可以设置为其中任意一个事件唤醒线程，或几个事件都达到后才唤醒线程进行
    后续的处理；同样，事件也可以是多个线程同步多个事件。

线程通过“逻辑与”或“逻辑或”将一个或多个事件关联起来，形成事件组合。

事件的“逻辑或”也称为独立型同步，指的是线程与任何事件之一发生同步；

事件“逻辑与”也称关联型同步，指的是线程与若干事件都发生同步；



.. Note:: 事件集的特点

    - 事件只与线程相关，事件间相互独立。
    - 事件仅用于同步，不提供数据传输功能；
    - 事件无排队性，即多次向线程发送同一个事件（如果线程还未来得及读走），其效果等同于只发送一次。

每个线程都拥有一个事件信息标记

- RT_EVENT_FLAG_OR 逻辑或
- RT_EVENT_FLAG_AND 逻辑与
- RT_EVENT_FLAG_CLEAR 清除标记

事件集控制块
-------------------

事件集控制块是操作系统用于管理事件的一个数据结构，由结构体struct rt_event表示。

另外一种C表达方式rt_event_t 表示的是事件集的句柄，在C语言中实现的是事件集控制块的指针。

.. code:: c

    struct rt_event
    {
        struct rt_ipc_object parent;                        /**< 继承自ipc_object类 */
     
        rt_uint32_t          set;                           /**< 事件集合，每一位表示1个事件，位的值可以标记某事件是否发生 */
    };
    //rt_event_t是指向事件结构体的指针类型
    typedef struct rt_event *rt_event_t;

事件集的管理方式
-------------------

创建和删除事件集
^^^^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: c

    rt_event_t rt_event_create(const char *name, rt_uint8_t flag);

.. table:: rt_event_create输入参数
    :widths: auto

    =============  ============================================================
    参数            描述
    =============  ============================================================
    name            事件集的名称
    flag            事件集的标志，取值（RT_EVENT_FLAG_OR，RT_EVENT_FLAG_AND）
    =============  ============================================================

.. table:: rt_event_create返回值
    :widths: auto

    ===============  ============================================================   
    返回              描述
    ===============  ============================================================
    RT_NULL           创建失败
    事件对象的句柄     创建成功 
    ===============  ============================================================

.. code-block:: C
    :caption: 删除事件集

    rt_err_t rt_event_delete(rt_event_t event);

.. Important:: 
    在调用rt_event_delete函数删除一个事件集对象时，确保该事件集不再被使用。

    在删除前会唤醒所有挂起在该事件集上的线程（线程的返回值是 -RT_ERROR），然后释放事件集对象占用的内存块。

rt_event_delete函数输入的参数为事件集对象的句柄，返回RT_EOK时代表成功。

初始化和脱离事件集
^^^^^^^^^^^^^^^^^^^^^^^^^^

静态事件集对象的内存是在系统编译时由编译器分配的，一般放于读写数据段或未初始化数据段中。

在使用静态事件集对象前，需要先初始化。

.. code-block:: C
    :caption: 初始化事件集函数

    rt_err_t rt_event_init(rt_event_t event, const char *name, rt_uint8_t flag);

- event 事件集对象的句柄
- name 事件集的名称
- flag 事件集的标志，取值（RT_IPC_FLAG_FIFO或RT_IPC_FLAG_PRIO）

返回

- RT_EOK 代表成功

不再使用rt_event_init（）初始化的事件集对象时，调用脱离事件集对象控制来释放系统资源。

.. code-block:: C
    
    rt_err_t rt_event_detach(rt_event_t event);

- event 事件集对象的句柄

返回

- RT_EOK 代表成功

发送事件
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: 
    :caption: 发送函数

    rt_err_t rt_event_send(rt_event_t event, rt_uint32_t set)；

通过set指定的事件标志设定event事件集对象的事件标志值，然后遍历event事件集对象上的等待线程链表，判断是否有线程的事件激活要求与当前event对象事件
标志值匹配，如果有，则唤醒该线程。

- event 事件集对象的句柄
- set 发送的一个或多个事件的标志值

返回

- RT_EOK 代表成功

接收事件
^^^^^^^^^^^^^^^^^^^^^^^^^^

内核使用32位的无符号整数来标识事件集，它的每一位代表一个事件，因此一个事件集对象可同时接收32个事件，内核可以通过指定选择参数“逻辑与”或“逻辑或”来
选择如何激活线程，使用“逻辑与”参数表示只有当所有等待的事件都发生时才激活线程，而“逻辑或”参数则表示只要有一个等待的事件发生就激活线程。

.. code-block:: C
    :caption: 接受事件

    rt_err_t rt_event_recv(rt_event_t   event,
                       rt_uint32_t  set,
                       rt_uint8_t   option,
                       rt_int32_t   timeout,
                       rt_uint32_t *recved)

当调用此接口时，先根据set和option来判断要接收的事件是否发生。

如果已经发生，则根据参数opti上是否有RT_EVENT_FLAG_CLEAR来决定是否重置事件的相应标志位，然后返回。

如果没有发生，则把等待的set和option参数填入线程本身的结构中，然后把线程挂起在此事件上，知道其等待的事件
满足条件或等待事件超过指定的超时时间。

若超时时间设置为零，则表示当线程接收的事件没有满足条件时，就不等待，返回-RT_ETIMEOUT

参数的描述

- event 事件集对象的句柄
- set 接收线程感兴趣的事件
- option 接收选项
- timeout 指定超时时间
- recved 指向接收到的事件

返回

- RT_EOK 成功
- -RT_ETIMEOUT 超时
- -RT_ERROR 错误

.. code-block:: C
    :caption: 事件集的使用例程

    #define THREAD_PRIOITY      9
    #define THREAD_TIMESLICE    5

    #define EVENT_FLAG3     (1 << 3)
    #define EVENT_FLAG5     (1 << 5)

    static struct rt_event event;

    ALIGN(RT_ALIGN_SIZE)
    static char thread1_stack[1024];
    static struct rt_thread thread1;

    static void thread1_recv_event(void *spram)
    {
        rt_uint32_t e;

        if(rt_event_recv(&event, (EVENT_FLAG3|EVENT_FLAG5),
                RT_EVENT_FLAG_OR|RT_EVENT_FLAG_CLEAR, RT_WAITING_FOREVER, &e) == RT_EOK)
        {
            rt_kprintf("thread1: OR recy event 0x%x\n", e);
        }

        rt_kprintf("thread1: delay ls to prepare the second event\n");

        rt_thread_delay(1000);

        if(rt_event_recv(&event, (EVENT_FLAG3|EVENT_FLAG5),
                RT_EVENT_FLAG_AND|RT_EVENT_FLAG_CLEAR, RT_WAITING_FOREVER, &e) == RT_EOK)
        {
            rt_kprintf("thread1：AND recy event 0x%x\n", e);
        }

        rt_kprintf("thread1 leave.\n");
    }

    ALIGN(RT_ALIGN_SIZE)
    static char thread2_stack[1024];
    static struct rt_thread thread2;

    static void thread2_send_event(void *param)
    {
        rt_kprintf("thread2: send event3\n");
        rt_event_send(&event, EVENT_FLAG3);
        rt_thread_mdelay(200);

        rt_kprintf("thrad2: send event5\n");
        rt_event_send(&event, EVENT_FLAG5);
        rt_thread_mdelay(200);

        rt_kprintf("thread2: send event3\n");
        rt_event_send(&event, EVENT_FLAG3);
        rt_kprintf("thread2 leave\n");
    }

    int event_sample(void)
    {
        rt_err_t result;

        result = rt_event_init(&event, "event", RT_IPC_FLAG_FIFO);

        if(result != RT_EOK)
        {
            rt_kprintf("init event failed\n");
            return -1;
        }

        rt_thread_init(&thread1,
                "thread1",
                thread1_recv_event,
                RT_NULL,
                &thread1_stack[0],
                sizeof(thread1_stack),
                THREAD_PRIOITY - 1,
                THREAD_TIMESLICE);
        rt_thread_startup(&thread1);

        rt_thread_init(&thread2,
                "thread2",
                thread2_send_event,
                RT_NULL,
                &thread2_stack[0],
                sizeof(thread2_stack),
                THREAD_PRIOITY,
                THREAD_TIMESLICE);
        rt_thread_startup(&thread2);

        return 0;

    }


事件集可用于多种场合，它能够在一定程度上代替信号量，用于线程间同步。

一个线程或中断服务例程发送一个事件给事件集对象，而后等待的线程被唤醒并对应的事件进行处理。

事件集中， 事件的发送操作是在事件为清除前是不可累计的，而信号量的释放动作是累计的。

事件的另一个特性是，接收线程可等待多种事件，即多个事件对应一个线程或多个线程。


本章小结
***************

1. 信号量可以在中断中释放，但不能在中断服务中获取。
2. 在获取互斥量后，请尽快释放互斥量，并且在持有互斥量的过程中，不得更改持有互斥量线程的优先级。
3. 互斥量不能在中断服务例程中使用。

