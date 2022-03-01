https://www.cnblogs.com/wangzahngjun/p/5116317.html

中断：上半部和下半部
上半部

local_irq_save() 禁用本cpu的所有中断

中断里边只能使用自旋锁 spin_lock（）---忙等
假设某进程持有锁，此时发生中断，中断尝试拿锁，如果进程和中断不是同一个cpu，则不会发生死锁，就等待就行了。如果进程和中断是同一个cpu，则会发生死锁，对于可能在中断里边发生的临界变量，设计的时候应该使用spin_lock_irqsave()，禁止中断。
（如果一个资源可能在中断处理程序中获取，那么获取这种资源，使用自旋锁应该使用spin_lock_irqsave（），禁止中断。而不是spin_lock（）了）

自旋锁：**禁止抢占，且不会睡眠**，就是死等而已

#define _raw_spin_lock(lock)                    __LOCK(lock)
#define __LOCK(lock) \
   do { preempt_disable(); ___LOCK(lock); } while (0)
#define ___LOCK(lock) \
  do { __acquire(lock); (void)(lock); } while (0)

#define spin_lock_irqsave(lock, flags)                          \
do {                                                            \
        raw_spin_lock_irqsave(spinlock_check(lock), flags);     \
} while (0)
raw_spin_lock_irqsave-->_raw_spin_lock_irqsave()-->
#define __LOCK_IRQSAVE(lock, flags) \
  do { local_irq_save(flags); __LOCK(lock); } while (0)

open_softirq 函数实际上用 softirq_action 参数填充了 softirq_vec 数组。由 open_softirq 注册的延后中断处理函数会由 raise_softirq 调用。这个函数只有一个参数 -- 软中断序号 nr。来看下它的实现：
```
void raise_softirq(unsigned int nr)
{
        unsigned long flags;

        local_irq_save(flags);
        raise_softirq_irqoff(nr);
        local_irq_restore(flags);
}

```
可以看到在 local_irq_save 和 local_irq_restore 两个宏中间调用了 raise_softirq_irqoff 函数。local_irq_save 的定义位于 include/linux/irqflags.h 头文件，它保存了 eflags 寄存器中的 IF 标志位并且禁用了当前处理器的中断。local_irq_restore 宏定义于相同头文件中，它做了完全相反的事情：装回之前保存的中断标志位然后允许中断。这里之所以要禁用中断是因为将要运行的 softirq 中断处理运行于中断上下文中。
----软中断处理运行在中断上下文中。
raise_softirq_irqoff 函数设置**当前处理器**上和nr参数对应的软中断标志位(__softirq_pending)

如果不在中断上下文中，就会调用 `wakeup_softirqd` 函数：

```
if (!in_interrupt())
	wakeup_softirqd();

```    
wakeup_softirqd 函数会激活当前处理器上的 `ksoftirqd` 内核线程：
```
static void wakeup_softirqd(void)
{
	struct task_struct *tsk = __this_cpu_read(ksoftirqd);

    if (tsk && tsk->state != TASK_RUNNING)
        wake_up_process(tsk);
}

```
每个 `ksoftirqd` 内核线程都运行 run_ksoftirqd 函数来检测是否有延后中断需要处理，如果有的话就会调用 `__do_softirq` 函数。__do_softirq 读取当前处理器对应的 __softirq_pending 软中断标记，并调用所有已被标记中断对应的处理函数。

在执行一个延后函数的同时，可能会发生新的软中断。这会导致用户态代码由于 __do_softirq 要处理很多延后中断而很长时间不能返回。为了解决这个问题，系统限制了延后中断处理的最大耗时：
```
unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
...
...
...
restart:
while ((softirq_bit = ffs(pending))) {
	...
	h->action(h);
	...
}
...
...
...
pending = local_softirq_pending();
if (pending) {
	if (time_before(jiffies, end) && !need_resched() &&
		--max_restart)
            goto restart;
}
...
```
除周期性检测是否有延后中断需要执行之外，系统还会在一些关键时间点上检测。一个主要的检测时间点就是当定义在 arch/x86/kernel/irq.c 的 do_IRQ 函数被调用时，这是 Linux 内核中执行延后中断的主要时机.在这个函数将要完成中断处理时它会exiting_irq 然后调用irq_exit 函数会检测当前处理器上下文是否有延后中断，有的话就会调用 invoke_softirq。

每个 softirq 都有如下的阶段：通过 open_softirq 函数注册一个软中断，通过 raise_softirq 函数标记一个软中断来激活它，然后所有被标记的软中断将会在 Linux 内核下一次执行周期性软中断检测时得以调度，对应此类型软中断的处理函数也就得以执行。

工作队列运行于内核进程上下文，而 tasklets 运行于软中断上下文。这意味着工作队列函数不必像 tasklets 一样必须是原子性的。Tasklets 总是运行于它提交自的那个处理器，工作队列在默认情况下也是这样

<http://kernel.meizu.com/linux-workqueue.html>
<https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-9.html>

