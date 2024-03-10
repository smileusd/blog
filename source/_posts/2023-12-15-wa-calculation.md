---
title: what is wa annd how is it calculated 
update: 2023-12-29 14:42:45
tags: linux
categories: system
---
### Question: wa usage

When you run top, you can see the usage of wa. What does it means and how it calculated?



```shell
%Cpu(s):  0.1 us,  2.4 sy,  0.0 ni, 93.6 id,  3.9 wa,  0.0 hi,  0.0 si,  0.0 st
```

Values related to processor utilization are displayed on the third line. They provide insight into exactly what the CPUs are doing.

- `us` is the percent of time spent running user processes.
- `sy` is the percent of time spent running the kernel.
- `ni` is the percent of time spent running processes with manually configured [nice values](https://www.redhat.com/sysadmin/manipulate-process-priority).
- `id` is the percent of time idle (if high, CPU may be overworked).
- **`wa` is the percent of wait time (if high, CPU is waiting for I/O access).**
- `hi` is the percent of time managing hardware interrupts.
- `si` is the percent of time managing software interrupts.
- `st` is the percent of virtual CPU time waiting for access to physical CPU.

In the context of the `top` command in Unix-like operating systems, the "wa" field represents the percentage of time the CPU spends waiting for I/O operations to complete. Specifically, it stands for "waiting for I/O."

The calculation includes the time the CPU is idle while waiting for data to be read from or written to storage devices such as hard drives or SSDs. High values in the "wa" field may indicate that the system is spending a significant amount of time waiting for I/O operations to complete, which could be a bottleneck if not addressed.

The formula for calculating the "wa" value is as follows:

**```wa %=(Time spent waiting for I/O / Total CPU time)×100```**

Also we can use `vmstat 1` to watch the `wa` state in system, but it is the not the percent:

```shell
# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 5  9      0 25994684     24 64404172    0    0     1    16    0    0  4  3 93  0  0
 0 10      0 25994436     24 64404172    0    0 342776 341104 63906 52162  2 10 83  4  3
 0 10      0 25994188     24 64404172    0    0 350392 350968 65593 52742  2 12 80  3  3
 1  9      0 25994188     24 64404172    0    0 342656 341128 63793 51367  2 11 80  3  4
 0 10      0 25994188     24 64404172    0    0 350760 350216 65439 52811  2 10 81  3  4
 5  8      0 25993940     24 64404172    0    0 355224 358952 66762 52421  2 10 80  3  5
 0 10      0 25993444     24 64404172    0    0 350536 350576 64998 52346  1 10 81  3  4
 1 10      0 25992948     24 64404172    0    0 348344 347912 64709 51990  1 11 81  3  4
 2 10      0 25992700     24 64404172    0    0 351944 349616 65443 51371  2 11 80  3  5
 0 10      0 26000764     24 64404172    0    0 343368 345000 64589 51969  2 12 80  3  4
 1  9      0 26000516     24 64404172    0    0 348976 352352 65800 53079  1  9 84  3  3
 1  9      0 26000516     24 64404172    0    0 345520 341800 63737 51454  1  8 84  3  3
 1  9      0 26000516     24 64404172    0    0 349192 352336 65719 52241  2  9 82  3  4
 1  9      0 26000516     24 64404172    0    0 352384 355696 66567 52232  2 11 80  3  4
 8  5      0 26000268     24 64404172    0    0 346368 345496 64541 52114  2 12 80  3  3
 0 10      0 26000020     24 64404172    0    0 340064 341296 63603 51670  2 11 81  3  2
 0  0      0 26007496     24 64399156    0    0 306736 306040 57770 46495  2 10 82  3  3
 0  0      0 26007588     24 64398828    0    0     0     0  853 1480  0  0 100  0  0
 0  0      0 26007588     24 64398828    0    0     0     0  871 1553  0  0 100  0  0
 0  0      0 26007588     24 64398828    0    0     0     0  748 1430  0  0 100  0  0
 0  0      0 26007588     24 64398828    0    0     0     2  731 1399  0  0 100  0  0
 0  0      0 26007588     24 64398828    0    0     0     0  695 1319  0  0 100  0  0
 0  0      0 26007588     24 64398828    0    0     0     0  743 1418  0  0 100  0  0
```

 ```shell
 Procs
     r: The number of processes waiting for run time.
     b: The number of processes in uninterruptible sleep.
 Memory
     swpd: the amount of virtual memory used.
     free: the amount of idle memory.
     buff: the amount of memory used as buffers.
     cache: the amount of memory used as cache.
     inact: the amount of inactive memory. (-a option)
     active: the amount of active memory. (-a option)
 Swap
     si: Amount of memory swapped in from disk (/s).
     so: Amount of memory swapped to disk (/s).
 IO
     bi: Blocks received from a block device (blocks/s).
     bo: Blocks sent to a block device (blocks/s).
 System
     in: The number of interrupts per second, including the clock.
     cs: The number of context switches per second.
 CPU
     These are percentages of total CPU time.
     us: Time spent running non-kernel code. (user time, including nice time)
     sy: Time spent running kernel code. (system time)
     id: Time spent idle. Prior to Linux 2.5.41, this includes IO-wait time.
     wa: Time spent waiting for IO. Prior to Linux 2.5.41, included in idle.
     st: Time stolen from a virtual machine. Prior to Linux 2.6.11, unknown.
 ```

In summary, a high "wa" value in the output of `top` suggests that your system is spending a considerable amount of time waiting for I/O operations to be completed, which could impact overall system performance. Investigating and optimizing disk I/O can be beneficial in such cases, possibly by improving disk performance, optimizing file systems, or identifying and addressing specific I/O-intensive processes. We can use `iotop` to find the high io or slow io processes in system:

```shell
Current DISK READ:     341.78 M/s | Current DISK WRITE:     340.53 M/s
    TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
3487445 be/4 root       34.44 M/s   33.36 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487446 be/4 root       33.40 M/s   34.27 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487447 be/4 root       34.10 M/s   33.78 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487448 be/4 root       32.21 M/s   31.64 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487449 be/4 root       34.22 M/s   34.44 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487450 be/4 root       34.46 M/s   34.33 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487451 be/4 root       34.17 M/s   34.21 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487452 be/4 root       33.05 M/s   32.95 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487453 be/4 root       34.23 M/s   34.74 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
3487454 be/4 root       37.51 M/s   36.79 M/s  ?unavailable?  fio -filename=/mnt/hostroot/var/mnt/testfile -direct=1 -iodepth 64 -thread -rw=randrw -ioengine=libaio -bs=8k -size=20G -numjobs=10 -group_reporting --runtime=100 -name=mytest
      1 be/4 root        0.00 B/s    0.00 B/s  ?unavailable?  systemd --switched-root --system --deserialize 29
      2 be/4 root        0.00 B/s    0.00 B/s  ?unavailable?  [kthreadd]
      3 be/4 root        0.00 B/s    0.00 B/s  ?unavailable?  [rcu_gp]
      4 be/4 root        0.00 B/s    0.00 B/s  ?unavailable?  [rcu_par_gp]
      6 be/4 root        0.00 B/s    0.00 B/s  ?unavailable?  [kworker/0:0H-events_highpri]
```

### Code tracing

From strace the top command, the `wa`  value is get the status from `/proc/*/stat`. The `/proc/*/stat `is wrote by the proc ops and get the iowait time from `kernel_cpustat.cpustat[CPUTIME_IOWAIT]` accroding to the code `fs/proc/stat.c`. 

```c
static const struct proc_ops stat_proc_ops = {
	.proc_flags	= PROC_ENTRY_PERMANENT,
	.proc_open	= stat_open,
	.proc_read_iter	= seq_read_iter,
	.proc_lseek	= seq_lseek,
	.proc_release	= single_release,
};

static int __init proc_stat_init(void)
{
	proc_create("stat", 0, NULL, &stat_proc_ops);
	return 0;
}

static int stat_open(struct inode *inode, struct file *file)
{
	unsigned int size = 1024 + 128 * num_online_cpus();

	/* minimum size to display an interrupt count : 2 bytes */
	size += 2 * nr_irqs;
	return single_open_size(file, show_stat, NULL, size);
}

static int show_stat(struct seq_file *p, void *v)
{
  ...
  		iowait		+= get_iowait_time(&kcpustat, i);
	...
}

static u64 get_iowait_time(struct kernel_cpustat *kcs, int cpu)
{
	u64 iowait, iowait_usecs = -1ULL;

	if (cpu_online(cpu))
		iowait_usecs = get_cpu_iowait_time_us(cpu, NULL);

	if (iowait_usecs == -1ULL)
		/* !NO_HZ or cpu offline so we can rely on cpustat.iowait */
		iowait = kcs->cpustat[CPUTIME_IOWAIT];
	else
		iowait = iowait_usecs * NSEC_PER_USEC;

	return iowait;
}

```

In the Linux kernel, the calculation of the "wa" (waiting for I/O) value that is reported by tools like `top` is handled within the kernel's scheduler. The specific code responsible for updating the CPU usage statistics can be found in the `kernel/sched/cputime.c` file.

One of the key functions related to this is `account_idle_time()`. 

```c
/*
 * Account for idle time.
 * @cputime: the CPU time spent in idle wait
 */
void account_idle_time(u64 cputime)
{
	u64 *cpustat = kcpustat_this_cpu->cpustat;
	struct rq *rq = this_rq();

	if (atomic_read(&rq->nr_iowait) > 0)
		cpustat[CPUTIME_IOWAIT] += cputime;
	else
		cpustat[CPUTIME_IDLE] += cputime;
}

	u32 nr_iowait;		/* number of blocked threads (waiting for I/O)    */


```

This function is part of the kernel's scheduler and is responsible for updating the various CPU time statistics, including the time spent waiting for I/O. The function takes into account different CPU states, such as idle time and time spent waiting for I/O.When The blocked threads with waiting for I/O, the cpu time accumulated into CPUTIME_IOWAIT.

Here is a basic outline of how the "wa" time is accounted for in the Linux kernel:

1. **Idle time accounting:** The kernel keeps track of the time the CPU spends in an idle state, waiting for tasks to execute.
2. **I/O wait time accounting:** When a process is waiting for I/O operations (such as reading or writing to a disk), the kernel accounts for this time in the appropriate CPU state, including the "wa" time.

```C
# block/fops.c
static ssize_t __blkdev_direct_IO(struct kiocb *iocb, struct iov_iter *iter,
		unsigned int nr_pages)
{
  ...
  	for (;;) {
		set_current_state(TASK_UNINTERRUPTIBLE);
		if (!READ_ONCE(dio->waiter))
			break;
		blk_io_schedule();
	}
	__set_current_state(TASK_RUNNING);
  ...
}  


```

```C
# block/blk-core.c
void blk_io_schedule(void)
{
	/* Prevent hang_check timer from firing at us during very long I/O */
	unsigned long timeout = sysctl_hung_task_timeout_secs * HZ / 2;
	if (timeout)
		io_schedule_timeout(timeout);
	else
		io_schedule();
}
EXPORT_SYMBOL_GPL(blk_io_schedule);
```



```c
# kernel/sched/core.c
/*
 * This task is about to go to sleep on IO. Increment rq->nr_iowait so
 * that process accounting knows that this is a task in IO wait state.
 */
long __sched io_schedule_timeout(long timeout)
{
	int token;
	long ret;

	token = io_schedule_prepare();
	ret = schedule_timeout(timeout);
	io_schedule_finish(token);

	return ret;
}
EXPORT_SYMBOL(io_schedule_timeout);

int io_schedule_prepare(void)
{
	int old_iowait = current->in_iowait;

	current->in_iowait = 1;
	blk_flush_plug(current->plug, true);
	return old_iowait;
}

void io_schedule_finish(int token)
{
	current->in_iowait = token;
}

asmlinkage __visible void __sched schedule(void)
{
	struct task_struct *tsk = current;

	sched_submit_work(tsk);
	do {
		preempt_disable();
		__schedule(SM_NONE);
		sched_preempt_enable_no_resched();
	} while (need_resched());
	sched_update_worker(tsk);
}
EXPORT_SYMBOL(schedule);

static void __sched notrace __schedule(unsigned int sched_mode)
{
  ...
			if (prev->in_iowait) {
				atomic_inc(&rq->nr_iowait);
				delayacct_blkio_start();
			}
  ...
}

```

```C
# kernel/time/timer.c
signed long __sched schedule_timeout(signed long timeout)
{
	struct process_timer timer;
	unsigned long expire;

	switch (timeout)
	{
	case MAX_SCHEDULE_TIMEOUT:
		/*
		 * These two special cases are useful to be comfortable
		 * in the caller. Nothing more. We could take
		 * MAX_SCHEDULE_TIMEOUT from one of the negative value
		 * but I' d like to return a valid offset (>=0) to allow
		 * the caller to do everything it want with the retval.
		 */
		schedule();
		goto out;
      ...
  }
}
```

```C
static int
try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
  ...
   if (task_cpu(p) != cpu) {
		if (p->in_iowait) {
			delayacct_blkio_end(p);
			atomic_dec(&task_rq(p)->nr_iowait);
		}

		wake_flags |= WF_MIGRATED;
		psi_ttwu_dequeue(p);
		set_task_cpu(p, cpu);
	}
  ...
}
```

