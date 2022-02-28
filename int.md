https://www.cnblogs.com/wangzahngjun/p/5116317.html

中断：上半部和下半部
上半部

local_irq_save() 禁用本cpu的所有中断

中断里边只能使用自旋锁 spin_lock（）---忙等
假设某进程持有锁，此时发生中断，中断尝试拿锁，如果进程和中断不是同一个cpu，则不会发生死锁，就等待就行了。如果进程和中断是同一个cpu，则会发生死锁，对于可能在中断里边发生的临界变量，设计的时候应该使用spin_lock_irqsave()，禁止中断。
（如果一个资源可能在中断处理程序中获取，那么获取这种资源，使用自旋锁应该使用spin_lock_irqsave（），禁止中断。而不是spin_lock（）了）

自旋锁：禁止抢占，且不会睡眠，就是死等而已
#define _raw_spin_lock(lock)                    __LOCK(lock)
#define __LOCK(lock) \
   do { preempt_disable(); ___LOCK(lock); } while (0)
