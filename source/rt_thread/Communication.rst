线程间通信
##################

在裸机编程中，经常会使用全局变量进行功能间的通信，如某些功能可能由于一些操作而改变全局变量的值，
另一个功能对此全局变量进行读取，根据读取到的全局变量值执行相应的动作，达到通信协作的目的。

邮箱
**************

邮箱服务是实时操作系统中一种典型的线程间通信方法。

邮箱的工作机制
==================

邮箱中的每一封邮件只能容纳固定的4字节内容（针对32位处理系统，指针的大小即位4个字节，所以
一封邮件恰好能够容纳一个指针）。

典型的邮箱也称作交换信息，线程或中断服务例程把一封4字节长度的邮件发送到邮箱中，而一个或多个线程
可以从邮箱中接收这些邮件并进行处理。

非堵塞方式的邮件发送过程能够安全地应用于中断服务中，是线程、中断服务、定时器向线程发送消息的
有效手段。

通常来说，邮件收取过程可能是堵塞的，这取决于邮箱中是否有邮件，以及收取邮件时设置的超时时间。当
邮箱中不存在邮件且超时时间不为0时，邮件收取过程将变成堵塞方式， **在这类情况** 下，只能由线程
进行邮件的收取。

当一个线程向邮箱发送邮件时，如果邮箱没满，将把邮件复制到邮箱中。如果邮箱已经满了，发送线程可以设置
超时时间，选择挂起等待或直接返回-RT_EFULL。如果发送线程选择挂起等待，那么当邮箱中的邮件被收取而
空出空间来时，等待挂起的发送线程将被唤醒继续发送。

当一个线程从邮箱中接收邮件时，如果邮箱是空的，接收线程可以选择是否等待挂起直到收到新的邮件而唤醒，
或可以设置超时时间。当达到设置的超时时间，邮箱依然未收到邮件时，这个选择超时等待的线程将被唤醒并
返回-RT_ETIMEOUT。如果邮箱中存在邮件，那么接收线程将复制邮箱中的4个字节邮件到接收缓存中。

邮箱控制块
==================

邮箱控制块是操作系统用于管理优先的一个数据结构，由结构体struct_mailbox表示，另一种C表达方式rt_mailbox_t，
表示的是邮箱的句柄，在C语言中的实现是邮箱控制块的指针。

.. code-block:: C

        struct rt_mailbox
    {
        struct rt_ipc_object parent;                        /**< inherit from ipc_object */

        rt_ubase_t          *msg_pool;                      /**< 邮箱缓冲区的开始地址 */

        rt_uint16_t          size;                          /**<邮箱缓冲区的大小 */

        rt_uint16_t          entry;                         /**< 邮箱中邮件的数目*/
        rt_uint16_t          in_offset;                     /**< 邮箱缓冲的进指针 */
        rt_uint16_t          out_offset;                    /**< 邮箱缓冲的出指针 */

        rt_list_t            suspend_sender_thread;         /**< 发送线程的挂起等待队列 */
    };


邮箱的管理方式
==================

邮箱控制块是一个结构体，其中含有事件相关的重要参数，在邮箱的功能实现中起着重要的作用。

创建和删除邮箱
------------------------

.. code-block:: c
    :caption: 动态创建邮箱对象

        rt_mailbox_t rt_mb_create(const char *name, rt_size_t size, rt_uint8_t flag)
    {
        rt_mailbox_t mb;

        RT_DEBUG_NOT_IN_INTERRUPT;

        /* allocate object */
        mb = (rt_mailbox_t)rt_object_allocate(RT_Object_Class_MailBox, name);
        if (mb == RT_NULL)
            return mb;

        /* set parent */
        mb->parent.parent.flag = flag;

        /* initialize ipc object */
        rt_ipc_object_init(&(mb->parent));

        /* initialize mailbox */
        mb->size     = size;
        mb->msg_pool = (rt_ubase_t *)RT_KERNEL_MALLOC(mb->size * sizeof(rt_ubase_t));
        if (mb->msg_pool == RT_NULL)
        {
            /* delete mailbox object */
            rt_object_delete(&(mb->parent.parent));

            return RT_NULL;
        }
        mb->entry      = 0;
        mb->in_offset  = 0;
        mb->out_offset = 0;

        /* initialize an additional list of sender suspend thread */
        rt_list_init(&(mb->suspend_sender_thread));

        return mb;
    }

创建优先对象时会先从对象管理器中分配一个邮箱对象，然后给邮箱动态分配一块内存空间用来存放邮件，这块内存的大小等于邮件大小（4字节）与邮箱容量的乘积，
接着初始化接收邮件数目和发送邮件在邮箱中的偏移量。

输入参数：

- name 邮件名称
- size 邮箱容量
- flag 邮箱标志，它可以取RT_IPC_FLAG_FIFO或RT_IPC_FLAG_PRIO

返回值

- RT_NULL 创建失败
- 邮箱对象的句柄  创建成功

.. code-block:: c
    :caption: 删除邮箱

        rt_err_t rt_mb_delete(rt_mailbox_t mb)
    {
        RT_DEBUG_NOT_IN_INTERRUPT;

        /* parameter check */
        RT_ASSERT(mb != RT_NULL);
        RT_ASSERT(rt_object_get_type(&mb->parent.parent) == RT_Object_Class_MailBox);
        RT_ASSERT(rt_object_is_systemobject(&mb->parent.parent) == RT_FALSE);

        /* resume all suspended thread */
        rt_ipc_list_resume_all(&(mb->parent.suspend_thread));

        /* also resume all mailbox private suspended thread */
        rt_ipc_list_resume_all(&(mb->suspend_sender_thread));

        /* free mailbox pool */
        RT_KERNEL_FREE(mb->msg_pool);

        /* delete mailbox object */
        rt_object_delete(&(mb->parent.parent));

        return RT_EOK;
    }

删除邮箱时，如果有线程被挂起在该邮箱对象上，内核先唤醒挂起在该邮箱上的所有线程（线程返回值是-RT_ERROR），然后再释放邮箱使用的内存，
最后删除邮箱对象。

参数的描述

- mb 邮箱对象的句柄

返回值

- RT_EOK 成功

初始化和脱离邮箱
------------------------

初始化邮箱用于静态邮箱对象的初始化。

静态邮箱对象的内存是在系统编译时由编译器分配的，一般放于读写数据段或未初始化数据段中，其余的初始化工作与创建邮箱时相同。

.. code-block:: c
    :caption: 初始化邮箱

        rt_err_t rt_mb_init(rt_mailbox_t mb,
                        const char  *name,
                        void        *msgpool,
                        rt_size_t    size,
                        rt_uint8_t   flag)
    {
        RT_ASSERT(mb != RT_NULL);

        /* initialize object */
        rt_object_init(&(mb->parent.parent), RT_Object_Class_MailBox, name);

        /* set parent flag */
        mb->parent.parent.flag = flag;

        /* initialize ipc object */
        rt_ipc_object_init(&(mb->parent));

        /* initialize mailbox */
        mb->msg_pool   = (rt_ubase_t *)msgpool;
        mb->size       = size;
        mb->entry      = 0;
        mb->in_offset  = 0;
        mb->out_offset = 0;

        /* initialize an additional list of sender suspend thread */
        rt_list_init(&(mb->suspend_sender_thread));

        return RT_EOK;
    }

输入参数：

- mb 邮箱对象的句柄
- name 邮箱名称
- msgpool 缓冲区指针
- size 邮箱容量
- flag 邮箱标志，它可以取RT_IPC_FLAG_FIFO或RT_IPC_FLAG_PRIO

返回值：

- RT_EOK 成功

如果msgpool指向的缓冲区的字节数是N，那么邮箱的容量应该是N/4

.. code-block:: c
    :caption: 脱离邮箱

        rt_err_t rt_mb_detach(rt_mailbox_t mb)
    {
        /* parameter check */
        RT_ASSERT(mb != RT_NULL);
        RT_ASSERT(rt_object_get_type(&mb->parent.parent) == RT_Object_Class_MailBox);
        RT_ASSERT(rt_object_is_systemobject(&mb->parent.parent));

        /* resume all suspended thread */
        rt_ipc_list_resume_all(&(mb->parent.suspend_thread));
        /* also resume all mailbox private suspended thread */
        rt_ipc_list_resume_all(&(mb->suspend_sender_thread));

        /* detach mailbox object */
        rt_object_detach(&(mb->parent.parent));

        return RT_EOK;
    }

使用该函数接口后，内核先唤醒所有挂在该邮箱上的线程（线程返回值是-RT_ERROR），然后将邮箱对象从内核对象管理器中脱离。

参数的描述：

- mb 邮箱对象的句柄

返回值

-RT_EOK 成功

发送邮件
------------------------

线程或者中断服务程序可以通过邮箱给其他线程发送邮件。

.. code-block:: c
    :caption: 发送邮件函数

        rt_err_t rt_mb_send(rt_mailbox_t mb, rt_ubase_t value)
    {
        return rt_mb_send_wait(mb, value, 0);
    }

发送的邮件是32位任意格式的数据，可以是一个整型值或者一个指向缓冲区的指针。

当邮箱中的邮件已满时，发送邮件的线程或者中断程序会收到-RT_EFULL的返回值。

输入参数：

- mb 邮箱对象的句柄
- value 邮件内容

返回值

- RT_EOK 发送成功
- -RT_EFULL 邮箱已经满了


等待方式发送邮件
------------------------

.. code-block:: c

    rt_err_t rt_mb_send_wait(rt_mailbox_t mb,
                            rt_ubase_t   value,
                            rt_int32_t   timeout)
    {
        struct rt_thread *thread;
        register rt_ubase_t temp;
        rt_uint32_t tick_delta;

        /* parameter check */
        RT_ASSERT(mb != RT_NULL);
        RT_ASSERT(rt_object_get_type(&mb->parent.parent) == RT_Object_Class_MailBox);

        /* initialize delta tick */
        tick_delta = 0;
        /* get current thread */
        thread = rt_thread_self();

        RT_OBJECT_HOOK_CALL(rt_object_put_hook, (&(mb->parent.parent)));

        /* disable interrupt */
        temp = rt_hw_interrupt_disable();

        /* for non-blocking call */
        if (mb->entry == mb->size && timeout == 0)
        {
            rt_hw_interrupt_enable(temp);

            return -RT_EFULL;
        }

        /* mailbox is full */
        while (mb->entry == mb->size)
        {
            /* reset error number in thread */
            thread->error = RT_EOK;

            /* no waiting, return timeout */
            if (timeout == 0)
            {
                /* enable interrupt */
                rt_hw_interrupt_enable(temp);

                return -RT_EFULL;
            }

            RT_DEBUG_IN_THREAD_CONTEXT;
            /* suspend current thread */
            rt_ipc_list_suspend(&(mb->suspend_sender_thread),
                                thread,
                                mb->parent.parent.flag);

            /* has waiting time, start thread timer */
            if (timeout > 0)
            {
                /* get the start tick of timer */
                tick_delta = rt_tick_get();

                RT_DEBUG_LOG(RT_DEBUG_IPC, ("mb_send_wait: start timer of thread:%s\n",
                                            thread->name));

                /* reset the timeout of thread timer and start it */
                rt_timer_control(&(thread->thread_timer),
                                RT_TIMER_CTRL_SET_TIME,
                                &timeout);
                rt_timer_start(&(thread->thread_timer));
            }

            /* enable interrupt */
            rt_hw_interrupt_enable(temp);

            /* re-schedule */
            rt_schedule();

            /* resume from suspend state */
            if (thread->error != RT_EOK)
            {
                /* return error */
                return thread->error;
            }

            /* disable interrupt */
            temp = rt_hw_interrupt_disable();

            /* if it's not waiting forever and then re-calculate timeout tick */
            if (timeout > 0)
            {
                tick_delta = rt_tick_get() - tick_delta;
                timeout -= tick_delta;
                if (timeout < 0)
                    timeout = 0;
            }
        }

        /* set ptr */
        mb->msg_pool[mb->in_offset] = value;
        /* increase input offset */
        ++ mb->in_offset;
        if (mb->in_offset >= mb->size)
            mb->in_offset = 0;
        
        if(mb->entry < RT_MB_ENTRY_MAX)
        {
            /* increase message entry */
            mb->entry ++;
        }
        else
        {
            rt_hw_interrupt_enable(temp); /* enable interrupt */
            return -RT_EFULL; /* value overflowed */
        }
        
        /* resume suspended thread */
        if (!rt_list_isempty(&mb->parent.suspend_thread))
        {
            rt_ipc_list_resume(&(mb->parent.suspend_thread));

            /* enable interrupt */
            rt_hw_interrupt_enable(temp);

            rt_schedule();

            return RT_EOK;
        }

        /* enable interrupt */
        rt_hw_interrupt_enable(temp);

        return RT_EOK;
    }

rt_mb_send_wait和rt_mb_send的区别在于是否有等待时间。

如果邮箱已经满了，那么发送线程将根据设定的timeout参数等待邮箱。如果设置的超时时间到达，但依然没有空出空间，发送线程将被唤醒并返回错误码。

- timeout 超时时间

接收邮件
------------------------

只有当接收者接收的邮箱中有邮件时，接收者才能立即取到邮件并返回RT_ROK的返回值，否则接收线程会更具超时时间设置，或挂起在邮箱的等待线程队列上，或直接返回。

.. code-block:: c
    :caption: 接收邮件函数

        rt_err_t rt_mb_recv(rt_mailbox_t mb, rt_ubase_t *value, rt_int32_t timeout)
    {
        struct rt_thread *thread;
        register rt_ubase_t temp;
        rt_uint32_t tick_delta;

        /* parameter check */
        RT_ASSERT(mb != RT_NULL);
        RT_ASSERT(rt_object_get_type(&mb->parent.parent) == RT_Object_Class_MailBox);

        /* initialize delta tick */
        tick_delta = 0;
        /* get current thread */
        thread = rt_thread_self();

        RT_OBJECT_HOOK_CALL(rt_object_trytake_hook, (&(mb->parent.parent)));

        /* disable interrupt */
        temp = rt_hw_interrupt_disable();

        /* for non-blocking call */
        if (mb->entry == 0 && timeout == 0)
        {
            rt_hw_interrupt_enable(temp);

            return -RT_ETIMEOUT;
        }

        /* mailbox is empty */
        while (mb->entry == 0)
        {
            /* reset error number in thread */
            thread->error = RT_EOK;

            /* no waiting, return timeout */
            if (timeout == 0)
            {
                /* enable interrupt */
                rt_hw_interrupt_enable(temp);

                thread->error = -RT_ETIMEOUT;

                return -RT_ETIMEOUT;
            }

            RT_DEBUG_IN_THREAD_CONTEXT;
            /* suspend current thread */
            rt_ipc_list_suspend(&(mb->parent.suspend_thread),
                                thread,
                                mb->parent.parent.flag);

            /* has waiting time, start thread timer */
            if (timeout > 0)
            {
                /* get the start tick of timer */
                tick_delta = rt_tick_get();

                RT_DEBUG_LOG(RT_DEBUG_IPC, ("mb_recv: start timer of thread:%s\n",
                                            thread->name));

                /* reset the timeout of thread timer and start it */
                rt_timer_control(&(thread->thread_timer),
                                RT_TIMER_CTRL_SET_TIME,
                                &timeout);
                rt_timer_start(&(thread->thread_timer));
            }

            /* enable interrupt */
            rt_hw_interrupt_enable(temp);

            /* re-schedule */
            rt_schedule();

            /* resume from suspend state */
            if (thread->error != RT_EOK)
            {
                /* return error */
                return thread->error;
            }

            /* disable interrupt */
            temp = rt_hw_interrupt_disable();

            /* if it's not waiting forever and then re-calculate timeout tick */
            if (timeout > 0)
            {
                tick_delta = rt_tick_get() - tick_delta;
                timeout -= tick_delta;
                if (timeout < 0)
                    timeout = 0;
            }
        }

        /* fill ptr */
        *value = mb->msg_pool[mb->out_offset];

        /* increase output offset */
        ++ mb->out_offset;
        if (mb->out_offset >= mb->size)
            mb->out_offset = 0;

        /* decrease message entry */
        if(mb->entry > 0)
        {
            mb->entry --;
        }

        /* resume suspended thread */
        if (!rt_list_isempty(&(mb->suspend_sender_thread)))
        {
            rt_ipc_list_resume(&(mb->suspend_sender_thread));

            /* enable interrupt */
            rt_hw_interrupt_enable(temp);

            RT_OBJECT_HOOK_CALL(rt_object_take_hook, (&(mb->parent.parent)));

            rt_schedule();

            return RT_EOK;
        }

        /* enable interrupt */
        rt_hw_interrupt_enable(temp);

        RT_OBJECT_HOOK_CALL(rt_object_take_hook, (&(mb->parent.parent)));

        return RT_EOK;
    }

接收邮件时，接收者需指定接收邮件的邮箱句柄，并指定接收到邮件的存放位置，以及最多能够等待的超时时间。

如果接收时设定了超时，当指定的时间内依然未收到邮件时，将返回-RT_ETIMEOUT

参数的描述

- mb 邮箱对象的句柄
- value 邮件内容
- timeout 超时时间

返回值：

- RT_EOK 发送成功
- -RT_ETIMEOUT 超时
- -RT_ERROR 失败，返回错误

.. code-block:: c
    :caption: 邮箱使用例程

    #define THREAD_PRIOITY      10
    #define THREAD_TIMESLICE    5

    static struct rt_mailbox mb;

    static char mb_pool[128];

    static char mb_str1 = "I'm a mail!";
    static char mb_str2 = "this is another mail!";
    static char mb_str3 = "over!";

    ALIGN(RT_ALIGN_SIZE)
    static char  thread1_stack([1024]);
    static struct rt_thread thread1;

    static void thread1_entry(void *parameter)
    {
        char *str;

        while(1)
        {
            rt_kprintf("thread1: try to recv a mail\n");

            if(rt_mb_recv(&mb, (rt_uint32_t *)&str, RT_WAITING_FOREVER) == RT_EOK)
            {
                rt_kprintf("thread1: get a mail from mailbox, the content:%s\n", str);
                if(str == mb_str3)
                    break;

                rt_thread_delay(100);
            }
        }

        rt_mb_detach(&mb);

    }

    ALIGN(RT_ALIGN_SIZE)
    static char  thread2_stack([1024]);
    static struct rt_thread thread2;

    static void thread2_entry(void *parameter)
    {
        rt_uint8_t count;

        count = 0;

        while(1)
        {
            count ++;
            if(count & 0x01)
            {
                rt_mb_send(&mb, (rt_uint32_t)&mb_str1);
            }
            else{
                rt_mb_send(&mb, (rt_uint32_t)&mb_str2);
            }

        rt_thread_delay(200);
        }

        rt_mb_send(&mb, (rt_uint32_t)&mb_str3);
    }

    int mailbox_sample(void)
    {
        rt_err_t result;

        result = rt_mb_init(&mb, "mbt", &mb_pool, sizeof(mb_pool)/4, RT_IPC_FLAG_FIFO);

        if(result != RT_EOK)
        {
            rt_kprintf("init mailbox failed.\n");
            return -1;
        }

        rt_thread_init(&thread1,
                "thread1",
                thread1_entry,
                RT_NULL,
                &thread1_stack[0],
                sizeof(thread1_stack),
                THREAD_PRIOITY,
                THREAD_TIMESLICE);

        rt_thread_startup(&thread1);

        rt_thread_init(&thread2,
                "thread2",
                thread2_entry,
                RT_NULL,
                &thread2_stack[0],
                sizeof(thread2_stack),
                THREAD_PRIOITY,
                THREAD_TIMESLICE);

        rt_thread_startup(&thread2);

        return 0;


    }

邮箱的使用场合
------------------------

邮箱在RT-Thread操作系统的实现中能够一次传递一个4字节大小的邮件，并且具备一定的存储功能，能够缓存一定数量的邮件数。

.. tip::　

    由于在32系统上4字节的内容恰好可以放置一个指针，因此当需要在线程间传递比较大的消息时，可以把指向一个缓冲区的指针作为邮件
    发送到邮箱中，即邮箱也可以传递指针。

    

.. code-block:: c
    :caption: 指针例子

    //消息结构体，包含指向数据的指针data_ptr和数据块长度的变量data_size

    struct msg
    {
        rt_uint8_t *data_ptr;
        rt_uint32_t data_size;
    }

    //发送给另一个线程

    struct msg* msg_ptr;

    msg_ptr = (struct msg*)rt_malloc(sizeof(struct msg));
    msg_ptr->data_ptr = ...;//指向相应的数据块地址
    msg_ptr->data_size = len;//数据块长度
    //发送消息指针给mb邮箱
    rt_mb_send(mb, (rt_uint32_t)msg_ptr);

    //在接收的过程中，因为收取过来的是指针，而msg_ptr是一个新分配出来的内存块，所以在接收线程处理完毕后，需要释放相应的内存块

    struct msg* msg_ptr;
    if(rt_mb_resc(mb, (rt_uint32_t*)&msg_ptr) == RT_EOK)
    {
        rt_free(msg_ptr);//在接收完线程处理完毕后，需要释放相应的内存块
    }

消息队列
******************

消息队列是另一种常见的线程间通信方式，是邮箱的扩展。

应用于：线程间的消息交换、使用串口接收不定长数据等。

消息队列的工作机制
======================

消息队列能够接收来自线程或终端服务例程中不固定长度的信息，并把消息缓存存在自己的内存空间中。

其他线程也能够从消息队列中读取相应的消息，而当消息队列是空的时候，可以挂起读取线程。

当有新的消息到达时，挂起的线程将被唤醒以接收并处理消息。

**消息队列是一种异步通信方式**

当有多个消息发送到消息队列是，线程先得到的是最先进入消息队列的消息，即先进先出原则。

消息队列控制块
=======================

消息队列控制块是操作系统用于管理消息队列的一个数据结构，有结构体rt_messagequeue表示，另一种C表达方式rt_mg_t，表示
的是消息队列的句柄，在C语言中的实现是消息队列控制块的指针。

.. code-block:: c
    :caption: 消息队列控制块定义

    struct rt_messagequeue
    {
        struct rt_ipc_object parent;                        /**< inherit from ipc_object */

        void                *msg_pool;                      /**< start address of message queue */

        rt_uint16_t          msg_size;                      /**< message size of each message */
        rt_uint16_t          max_msgs;                      /**< max number of messages */

        rt_uint16_t          entry;                         /**< index of messages in the queue */

        void                *msg_queue_head;                /**< list head */
        void                *msg_queue_tail;                /**< list tail */
        void                *msg_queue_free;                /**< pointer indicated the free node of queue */

        rt_list_t            suspend_sender_thread;         /**< sender thread suspended on this message queue */
    };

消息队列的管理方式
=====================

创建和删除消息队列
------------------------

.. code-block:: c
    :caption: 动态创建消息队列

    rt_mq_t rt_mq_create(const char *name,
                        rt_size_t   msg_size,
                        rt_size_t   max_msgs,
                        rt_uint8_t  flag)
    {
        struct rt_messagequeue *mq;
        struct rt_mq_message *head;
        register rt_base_t temp;

        RT_DEBUG_NOT_IN_INTERRUPT;

        /* allocate object */
        mq = (rt_mq_t)rt_object_allocate(RT_Object_Class_MessageQueue, name);
        if (mq == RT_NULL)
            return mq;

        /* set parent */
        mq->parent.parent.flag = flag;

        /* initialize ipc object */
        rt_ipc_object_init(&(mq->parent));

        /* initialize message queue */

        /* get correct message size */
        mq->msg_size = RT_ALIGN(msg_size, RT_ALIGN_SIZE);
        mq->max_msgs = max_msgs;

        /* allocate message pool */
        mq->msg_pool = RT_KERNEL_MALLOC((mq->msg_size + sizeof(struct rt_mq_message)) * mq->max_msgs);
        if (mq->msg_pool == RT_NULL)
        {
            rt_object_delete(&(mq->parent.parent));

            return RT_NULL;
        }

        /* initialize message list */
        mq->msg_queue_head = RT_NULL;
        mq->msg_queue_tail = RT_NULL;

        /* initialize message empty list */
        mq->msg_queue_free = RT_NULL;
        for (temp = 0; temp < mq->max_msgs; temp ++)
        {
            head = (struct rt_mq_message *)((rt_uint8_t *)mq->msg_pool +
                                            temp * (mq->msg_size + sizeof(struct rt_mq_message)));
            head->next = (struct rt_mq_message *)mq->msg_queue_free;
            mq->msg_queue_free = head;
        }

        /* the initial entry is zero */
        mq->entry = 0;

        /* initialize an additional list of sender suspend thread */
        rt_list_init(&(mq->suspend_sender_thread));

        return mq;
    }

创建消息队列时先从对象管理器中分配一个消息队列对象，然后给消息队列对象分配一个内存空间，组织成空闲消息链表，
这块内存的大小=[消息大小+消息头（用于链表连接）的大小]*消息队列最大个数，接着再初始化消息队列，此时消息队列为空。

输入参数：

- name 消息队列的名称
- msg_size 消息队列中一条消息的最大长度，单位为字节
- max_msgs 消息队列的最大个数
- falg 消息队列采用的等待方式，取RT_IPC_FLAG_FIFO和RT_IPC_FLAG_PRIO

返回值：

- 消息队列对象的句柄 成功
- RT_NULL 失败

当消息队列不再被使用时，应该删除它以释放系统资源，一旦操作完成，消息队列将被永久性地删除。

.. code-block:: c
    :caption: 删除函数

    rt_err_t rt_mq_delete(rt_mq_t mq)
    {
        RT_DEBUG_NOT_IN_INTERRUPT;

        /* parameter check */
        RT_ASSERT(mq != RT_NULL);
        RT_ASSERT(rt_object_get_type(&mq->parent.parent) == RT_Object_Class_MessageQueue);
        RT_ASSERT(rt_object_is_systemobject(&mq->parent.parent) == RT_FALSE);

        /* resume all suspended thread */
        rt_ipc_list_resume_all(&(mq->parent.suspend_thread));
        /* also resume all message queue private suspended thread */
        rt_ipc_list_resume_all(&(mq->suspend_sender_thread));

        /* free message queue pool */
        RT_KERNEL_FREE(mq->msg_pool);

        /* delete message queue object */
        rt_object_delete(&(mq->parent.parent));

        return RT_EOK;
    }

删除消息队列时，如果有线程被挂起在该消息队列等待队列上，则内核先唤醒挂起在该消息等待队列上的所有线程（线程返回值是-RT_ERROR），
然后再释放消息队列使用的内存，最后删除消息队列对象。

函数输入的是消息队列对象的句柄，返回RT_EOK时代表成功

初始化消息队列
--------------

静态创建消息队列

.. code-block:: c
    :caption: 初始化

    rt_err_t rt_mq_init(rt_mq_t     mq,
                        const char *name,
                        void       *msgpool,
                        rt_size_t   msg_size,
                        rt_size_t   pool_size,
                        rt_uint8_t  flag)
    {
        struct rt_mq_message *head;
        register rt_base_t temp;

        /* parameter check */
        RT_ASSERT(mq != RT_NULL);

        /* initialize object */
        rt_object_init(&(mq->parent.parent), RT_Object_Class_MessageQueue, name);

        /* set parent flag */
        mq->parent.parent.flag = flag;

        /* initialize ipc object */
        rt_ipc_object_init(&(mq->parent));

        /* set message pool */
        mq->msg_pool = msgpool;

        /* get correct message size */
        mq->msg_size = RT_ALIGN(msg_size, RT_ALIGN_SIZE);
        mq->max_msgs = pool_size / (mq->msg_size + sizeof(struct rt_mq_message));

        /* initialize message list */
        mq->msg_queue_head = RT_NULL;
        mq->msg_queue_tail = RT_NULL;

        /* initialize message empty list */
        mq->msg_queue_free = RT_NULL;
        for (temp = 0; temp < mq->max_msgs; temp ++)
        {
            head = (struct rt_mq_message *)((rt_uint8_t *)mq->msg_pool +
                                            temp * (mq->msg_size + sizeof(struct rt_mq_message)));
            head->next = (struct rt_mq_message *)mq->msg_queue_free;
            mq->msg_queue_free = head;
        }

        /* the initial entry is zero */
        mq->entry = 0;

        /* initialize an additional list of sender suspend thread */
        rt_list_init(&(mq->suspend_sender_thread));

        return RT_EOK;
    }

参数的描述：

- mq 消息队列对象的句柄
- name 消息队列的名称
-  msgpool 指向存放消息的缓冲区的指针
- msg_size 消息队列中一条消息的最大长度，单位为字节
- pool_size 存放消息的缓冲区大小
- flag 消息队列采用的等待方式，取RT_IPC_FLAG_FIFO和RT_IPC_FLAG_PRIO

返回值：

- RT_EOK 成功

脱离消息队列
----------------

.. code-block:: c
    :caption: 脱离消息队列

    rt_err_t rt_mq_detach(rt_mq_t mq)
    {
        /* parameter check */
        RT_ASSERT(mq != RT_NULL);
        RT_ASSERT(rt_object_get_type(&mq->parent.parent) == RT_Object_Class_MessageQueue);
        RT_ASSERT(rt_object_is_systemobject(&mq->parent.parent));

        /* resume all suspended thread */
        rt_ipc_list_resume_all(&mq->parent.suspend_thread);
        /* also resume all message queue private suspended thread */
        rt_ipc_list_resume_all(&(mq->suspend_sender_thread));

        /* detach message queue object */
        rt_object_detach(&(mq->parent.parent));

        return RT_EOK;
    }

输入消息队列对象的句柄，返回RT_EOK时代表成功。

发送信息
----------

线程或者中断服务程序都可以给消息队列发送消息。

当发送消息时，消息队列对象先从空闲消息链表上取下一个空闲消息块，把线程或者中断服务程序发送的消息内容复制到消息块上，
然后把该消息块挂到消息队列的尾部。

当且仅当空闲消息链表上有可用的空闲消息块时，发送者才能成功发送消息； 当空闲消息链表上无可用消息块时，说明消息队列已满。
此时发送消息的线程或者中断程序会收到一个错误码（-RT_EFULL）。

.. code-block:: c
    :caption: 发送消息函数

    rt_err_t rt_mq_send(rt_mq_t mq, const void *buffer, rt_size_t size)
    {
        return rt_mq_send_wait(mq, buffer, size, 0);
    }

.. code-block:: c
    :caption: 发送消息函数(带延迟)

    rt_err_t rt_mq_send_wait(rt_mq_t     mq,
                            const void *buffer,
                            rt_size_t   size,
                            rt_int32_t  timeout)
    {
        register rt_ubase_t temp;
        struct rt_mq_message *msg;
        rt_uint32_t tick_delta;
        struct rt_thread *thread;

        /* parameter check */
        RT_ASSERT(mq != RT_NULL);
        RT_ASSERT(rt_object_get_type(&mq->parent.parent) == RT_Object_Class_MessageQueue);
        RT_ASSERT(buffer != RT_NULL);
        RT_ASSERT(size != 0);

        /* greater than one message size */
        if (size > mq->msg_size)
            return -RT_ERROR;

        /* initialize delta tick */
        tick_delta = 0;
        /* get current thread */
        thread = rt_thread_self();

        RT_OBJECT_HOOK_CALL(rt_object_put_hook, (&(mq->parent.parent)));

        /* disable interrupt */
        temp = rt_hw_interrupt_disable();

        /* get a free list, there must be an empty item */
        msg = (struct rt_mq_message *)mq->msg_queue_free;
        /* for non-blocking call */
        if (msg == RT_NULL && timeout == 0)
        {
            /* enable interrupt */
            rt_hw_interrupt_enable(temp);

            return -RT_EFULL;
        }

        /* message queue is full */
        while ((msg = mq->msg_queue_free) == RT_NULL)
        {
            /* reset error number in thread */
            thread->error = RT_EOK;

            /* no waiting, return timeout */
            if (timeout == 0)
            {
                /* enable interrupt */
                rt_hw_interrupt_enable(temp);

                return -RT_EFULL;
            }

            RT_DEBUG_IN_THREAD_CONTEXT;
            /* suspend current thread */
            rt_ipc_list_suspend(&(mq->suspend_sender_thread),
                                thread,
                                mq->parent.parent.flag);

            /* has waiting time, start thread timer */
            if (timeout > 0)
            {
                /* get the start tick of timer */
                tick_delta = rt_tick_get();

                RT_DEBUG_LOG(RT_DEBUG_IPC, ("mq_send_wait: start timer of thread:%s\n",
                                            thread->name));

                /* reset the timeout of thread timer and start it */
                rt_timer_control(&(thread->thread_timer),
                                RT_TIMER_CTRL_SET_TIME,
                                &timeout);
                rt_timer_start(&(thread->thread_timer));
            }

            /* enable interrupt */
            rt_hw_interrupt_enable(temp);

            /* re-schedule */
            rt_schedule();

            /* resume from suspend state */
            if (thread->error != RT_EOK)
            {
                /* return error */
                return thread->error;
            }

            /* disable interrupt */
            temp = rt_hw_interrupt_disable();

            /* if it's not waiting forever and then re-calculate timeout tick */
            if (timeout > 0)
            {
                tick_delta = rt_tick_get() - tick_delta;
                timeout -= tick_delta;
                if (timeout < 0)
                    timeout = 0;
            }
        }

        /* move free list pointer */
        mq->msg_queue_free = msg->next;

        /* enable interrupt */
        rt_hw_interrupt_enable(temp);

        /* the msg is the new tailer of list, the next shall be NULL */
        msg->next = RT_NULL;
        /* copy buffer */
        rt_memcpy(msg + 1, buffer, size);

        /* disable interrupt */
        temp = rt_hw_interrupt_disable();
        /* link msg to message queue */
        if (mq->msg_queue_tail != RT_NULL)
        {
            /* if the tail exists, */
            ((struct rt_mq_message *)mq->msg_queue_tail)->next = msg;
        }

        /* set new tail */
        mq->msg_queue_tail = msg;
        /* if the head is empty, set head */
        if (mq->msg_queue_head == RT_NULL)
            mq->msg_queue_head = msg;

        if(mq->entry < RT_MQ_ENTRY_MAX)
        {
            /* increase message entry */
            mq->entry ++;
        }
        else
        {
            rt_hw_interrupt_enable(temp); /* enable interrupt */
            return -RT_EFULL; /* value overflowed */
        }

        /* resume suspended thread */
        if (!rt_list_isempty(&mq->parent.suspend_thread))
        {
            rt_ipc_list_resume(&(mq->parent.suspend_thread));

            /* enable interrupt */
            rt_hw_interrupt_enable(temp);

            rt_schedule();

            return RT_EOK;
        }

        /* enable interrupt */
        rt_hw_interrupt_enable(temp);

        return RT_EOK;
    }

输入参数：

- name 消息队列对象的句柄
- buffer 消息内容
- size 消息大小

返回:

- RT_EOK 成功
- -RT_EFULL 消息队列已满
- -RT_ERROR 失败，表示发送的消息长度大于消息队列中消息的最大长度

发送紧急信息
---------------

当发送紧急消息时，从空闲链表上取下来的消息块不是挂到消息队列的队尾，而是挂到队首，这样接收者就能够优先接受到紧急信息，
从而及时进行消息处理。

.. code-block:: C
    :caption: 紧急信息函数

    rt_err_t rt_mq_urgent(rt_mq_t mq, const void *buffer, rt_size_t size)

- mq 消息队列对象的句柄
- buffer 消息内容
- size 消息大小

返回值：

- RT_EOK 成功
- -RT_EFULL 消息队列已满
- -RT_ERROR 失败

接收消息
----------

当消息队列中有消息时，接受者才能接收消息，否则接收者会根据超时时间设置，或挂起在消息队列的等待线程上，或直接返回。

.. code-block:: c
    :caption: 接收信息函数

    rt_err_t rt_mq_recv(rt_mq_t    mq,
                        void      *buffer,
                        rt_size_t  size,
                        rt_int32_t timeout)

接收消息时，接收者需指定存储消息的消息队列对象句柄，并且指定一个内存缓冲区，接受到的消息内容将被复制到该缓冲区里。
此外还需指定未能及时取到消息时的超时时间。

接收一个消息后，消息队列上的队首信息被转移到了空闲消息链表的尾部。

- mq 消息队列对象的句柄
- buffer 消息内容
- size 消息大小
- timeout 超时时间

返回值：

- RT_EOK 成功
- -RT_ETIMEOUT 超时
- -RT_ERROR 失败，返回错误
