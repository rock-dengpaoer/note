时钟管理
#######################

创建动态定时器例程

.. code-block:: C
   :caption: 定时器的控制块

    static rt_timer_t timer1;
    static rt_timer_t timer2;
    static int cnt = 0;

    //定时器1超时函数
    static void timeout1(void *parameter)
    {

        rt_kprintf("periodic timer is timeout %d\n", cnt);

        //运行10次，停止周期性定时器
        if(cnt++ >= 9)
        {
            rt_timer_stop(timer1);
            rt_kprintf("periodic timer was stopped! \n");
        }
    }

    //定时器2超时函数
    static void timeout2(void *parameter)
    {
        rt_kprintf("one shot timer is timeout \n");
    }

    int timer_sample(void)
    {
        //创建定时器1，周期性定时器
        timer1 = rt_timer_create("timer1",
                                timeout1,
                                RT_NULL,
                                10,
                                RT_TIMER_FLAG_PERIODIC);

        if(timer1 != RT_NULL)
            rt_timer_start(timer1);//启动定时器

        //创建定时器2，单次定时器
        timer2 = rt_timer_create("timer2",
                                timeout2,
                                RT_NULL,
                                30,
                                RT_TIMER_FLAG_ONE_SHOT);
        if(timer2 != RT_NULL)
                rt_timer_start(timer2);

        return 0;

    }



.. code-block:: C
   :caption: 初始化静态定时器例程

    //定时器的控制块
    static struct rt_timer timer1;
    static struct rt_timer timer2;
    static int cnt = 0;

    //定时器1超时函数
    static void timeout1(void* parameter)
    {

        rt_kprintf("periodic timer is timeout\n");

        if(cnt++ >= 9)
        {
            rt_timer_stop(&timer1);
        }
    }

    //定时器2超时函数
    static void timeout2(void* parameter)
    {
        rt_kprintf("one shot timer is timeout\n");
    }

    int timer_static_sample(void)
    {
        //初始化定时器
        rt_timer_init(&timer1,
                    "timer1", //定时器名字是timer1
                    timeout1, //超时时回调的处理函数
                    RT_NULL,  //超时时回调的入口参数
                    10,//定时长度，以OS Tick为单位
                    RT_TIMER_FLAG_PERIODIC);//周期性定时器

        rt_timer_init(&timer2,
                    "timeout2",
                    timeout2,
                    RT_NULL,
                    30,
                    RT_TIMER_FLAG_ONE_SHOT);//单次定时器

        //启动定时器
        rt_timer_start(&timer1);
        rt_timer_start(&timer2);

        return 0;
    }

