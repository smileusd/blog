# IOCost model impact cpu soft lockup

title: IOCost model impact cpu soft lockup
update: 2024-03-06 14:42:45
tags: linux
categories: system

### Reproduce 

*environment*:

5.15 kernel

*way*:

Enable cgroup v2 and bfq, set bfq as scheduler on target deviceEnable io cost by /sys/fs/cgroup/io.cost.qos, set cgroup io.weightStart two or more fio process to read or write on the target device, at least contains 1 direct and 1 buffer io, each one use different cgroupsWait 10-30 minutes

### Debug

Kernel Dump 1 (Set hardlockup_panic=1 by sysctl):

```shell
[ 1408.828715] Call Trace:
[ 1408.828716]  <TASK>
[ 1408.828718]  _raw_spin_lock_irq+0x26/0x30
[ 1408.828722]  bfq_bio_merge+0x9f/0x160 [bfq]
[ 1408.828727]  __blk_mq_sched_bio_merge+0x36/0x190
[ 1408.828731]  blk_mq_submit_bio+0xd2/0x600
[ 1408.828735]  __submit_bio+0x1dc/0x210
[ 1408.828738]  submit_bio_noacct+0x263/0x2c0
[ 1408.828741]  submit_bio+0x48/0x130
[ 1408.828743]  iomap_submit_ioend.isra.0+0x52/0x80
[ 1408.828745]  iomap_writepage_map+0x490/0x630
[ 1408.828748]  iomap_do_writepage+0x75/0x230
[ 1408.828750]  ? clear_page_dirty_for_io+0xdf/0x1d0
[ 1408.828754]  write_cache_pages+0x184/0x450
[ 1408.828756]  ? iomap_truncate_page+0x60/0x60
[ 1408.828760]  iomap_writepages+0x20/0x40
[ 1408.828762]  xfs_vm_writepages+0x84/0xb0 [xfs]
[ 1408.828837]  do_writepages+0xc5/0x1c0
[ 1408.828839]  ? __wb_calc_thresh+0x3e/0x120
[ 1408.828842]  __writeback_single_inode+0x44/0x290
[ 1408.828845]  writeback_sb_inodes+0x22d/0x4b0
[ 1408.828849]  __writeback_inodes_wb+0x56/0xf0
[ 1408.828852]  wb_writeback+0x1ce/0x290
[ 1408.828855]  wb_workfn+0x31b/0x490
[ 1408.828858]  ? psi_task_switch+0xc3/0x240
[ 1408.828862]  process_one_work+0x22b/0x3d0
[ 1408.828865]  worker_thread+0x4d/0x3f0
[ 1408.828868]  ? process_one_work+0x3d0/0x3d0
[ 1408.828870]  kthread+0x12a/0x150
[ 1408.828872]  ? set_kthread_struct+0x40/0x40
[ 1408.828874]  ret_from_fork+0x22/0x30
[ 1408.828879]  </TASK>
[ 1408.829600] Kernel panic - not syncing: Hard LOCKUP
[ 1408.829602] CPU: 26 PID: 0 Comm: swapper/26 Kdump: loaded Tainted: P          IOE K   5.15.0-26-generic #26
[ 1408.829604] Hardware name: Dell Inc. PowerEdge C6420/0K2TT6, BIOS 2.4.8 11/27/2019
[ 1408.829605] Call Trace:
[ 1408.829606]  <NMI>
[ 1408.829608]  dump_stack_lvl+0x4a/0x5f
[ 1408.829615]  dump_stack+0x10/0x12
[ 1408.829617]  panic+0x149/0x321
[ 1408.829623]  nmi_panic.cold+0xc/0xc
[ 1408.829626]  watchdog_overflow_callback.cold+0x5c/0x70
[ 1408.829631]  __perf_event_overflow+0x57/0x100
[ 1408.829634]  perf_event_overflow+0x14/0x20
[ 1408.829637]  handle_pmi_common+0x1f3/0x2f0
[ 1408.829643]  ? flush_tlb_one_kernel+0xe/0x20
[ 1408.829647]  ? __set_pte_vaddr+0x37/0x40
[ 1408.829650]  ? set_pte_vaddr_p4d+0x3d/0x50
[ 1408.829652]  ? set_pte_vaddr+0x81/0xb0
[ 1408.829653]  ? __native_set_fixmap+0x28/0x40
[ 1408.829655]  ? native_set_fixmap+0x66/0x70
[ 1408.829657]  ? ghes_copy_tofrom_phys+0x7c/0x120
[ 1408.829663]  ? __ghes_peek_estatus.isra.0+0x4e/0xb0
[ 1408.829666]  intel_pmu_handle_irq+0xf4/0x210
[ 1408.829670]  perf_event_nmi_handler+0x2d/0x50
[ 1408.829672]  nmi_handle+0x66/0x120
[ 1408.829678]  default_do_nmi+0x45/0x110
[ 1408.829681]  exc_nmi+0x175/0x190
[ 1408.829683]  end_repeat_nmi+0x16/0x55
[ 1408.829685] RIP: 0010:native_queued_spin_lock_slowpath+0x14f/0x220
[ 1408.827472]  ? adjust_inuse_and_calc_cost+0x13a/0x250
[ 1408.827475]  ioc_rqos_merge+0xef/0x260
[ 1408.827478]  __rq_qos_merge+0x31/0x50
[ 1408.827481]  bio_attempt_back_merge+0x43/0xd0
[ 1408.827484]  blk_mq_sched_try_merge+0x147/0x1d0
[ 1408.827486]  bfq_bio_merge+0xe2/0x160 [bfq]
[ 1408.827491]  __blk_mq_sched_bio_merge+0x36/0x190
[ 1408.827495]  blk_mq_submit_bio+0xd2/0x600
[ 1408.827499]  __submit_bio+0x1dc/0x210
[ 1408.827502]  ? get_user_pages_fast+0x24/0x40
[ 1408.827507]  ? iov_iter_get_pages+0xc6/0x3f0
[ 1408.827513]  submit_bio_noacct+0xac/0x2c0
[ 1408.827516]  submit_bio+0x48/0x130
[ 1408.827518]  iomap_dio_submit_bio+0x79/0x80
[ 1408.827524]  iomap_dio_bio_iter+0x2d7/0x4a0
[ 1408.827528]  __iomap_dio_rw+0x531/0x7c0
[ 1408.827532]  iomap_dio_rw+0xe/0x30
[ 1408.827535]  xfs_file_dio_write_aligned+0xa0/0x110 [xfs]
[ 1408.827647]  xfs_file_write_iter+0x101/0x1a0 [xfs]
[ 1408.827736]  aio_write+0x116/0x210
[ 1408.827740]  ? update_load_avg+0x7c/0x640
[ 1408.827744]  ? set_next_entity+0xb7/0x200
[ 1408.827747]  __io_submit_one.constprop.0+0x40e/0x720
[ 1408.827749]  ? __io_submit_one.constprop.0+0x40e/0x720
[ 1408.827752]  ? __check_object_size+0x5d/0x150
[ 1408.827755]  ? _copy_to_user+0x20/0x30
[ 1408.827760]  ? aio_read_events+0x210/0x320
[ 1408.827762]  io_submit_one+0xe3/0x550
[ 1408.827764]  ? io_submit_one+0xe3/0x550
[ 1408.827766]  ? audit_filter_rules.constprop.0+0x3b1/0x12e0
[ 1408.827772]  __x64_sys_io_submit+0x8d/0x180
[ 1408.827775]  ? syscall_trace_enter.isra.0+0x146/0x1c0
[ 1408.827779]  do_syscall_64+0x5c/0xc0
[ 1408.827782]  ? exit_to_user_mode_prepare+0x3d/0x1c0
[ 1408.827785]  ? syscall_exit_to_user_mode+0x27/0x50
[ 1408.827788]  ? do_syscall_64+0x69/0xc0
[ 1408.827790]  ? irqentry_exit+0x19/0x30
[ 1408.827793]  ? sysvec_call_function_single+0x4e/0x90
[ 1408.827795]  ? asm_sysvec_call_function_single+0xa/0x20
[ 1408.827799]  entry_SYSCALL_64_after_hwframe+0x44/0xae
```





Kernel dump 2 :

```shell
[60977.577690] BUG: spinlock recursion on CPU#21, fio/11770
[60977.583007]  lock: 0xffff92600e348c20, .magic: dead4ead, .owner: fio/11770, .owner_cpu: 21
[60977.591275] CPU: 21 PID: 11770 Comm: fio Kdump: loaded Not tainted 5.15.0-26-generic #26
[60977.599362] Hardware name: Supermicro SYS-6029TP-H-EI012/X11DPT-PS-EI012, BIOS 1.0.9.0 01/31/2018
[60977.608229] Call Trace:
[60977.610682]  <IRQ>
[60977.612700]  dump_stack_lvl+0x58/0x7d
[60977.616367]  dump_stack+0x10/0x12
[60977.619687]  spin_dump.cold+0x24/0x39
[60977.623359]  do_raw_spin_lock+0x9a/0xd0
[60977.627198]  _raw_spin_lock_irqsave+0x7d/0x80
[60977.631561]  bfq_finish_requeue_request+0x55/0x540 [bfq]
[60977.636874]  blk_mq_free_request+0x3e/0x160
[60977.641058]  __blk_mq_end_request+0x102/0x110
[60977.645419]  scsi_end_request+0xce/0x190
[60977.649345]  scsi_io_completion+0x7e/0x5d0
[60977.653442]  scsi_finish_command+0xca/0x100
[60977.657629]  scsi_complete+0x74/0xe0
[60977.661212]  blk_complete_reqs+0x3b/0x50
[60977.665135]  blk_done_softirq+0x1d/0x20
[60977.668975]  __do_softirq+0xf7/0x31e
[60977.672554]  irq_exit_rcu+0x79/0xa0
[60977.676044]  sysvec_call_function_single+0x7c/0x90
[60977.680837]  </IRQ>
[60977.682945]  <TASK>
[60977.685051]  asm_sysvec_call_function_single+0x12/0x20
[60977.690189] RIP: 0010:_raw_spin_unlock_irq+0x2a/0x30
[60977.695155] Code: 0f 1f 44 00 00 55 48 89 e5 41 54 49 89 fc 48 8d 7f 18 48 8b 75 08 e8 75 6f 3d ff 4c 89 e7 e8 8d b0 3d ff fb 66 0f 1f 44 00 00 <41> 5c 5d c3 66 90 0f 1f 44 00 00 55 48 89 e5 41 54 49 89 fc 48 8d
[60977.713902] RSP: 0018:ffffa0781992b670 EFLAGS: 00000246
[60977.719128] RAX: 0000000000000015 RBX: 00000000001d9281 RCX: 0000000036d7555b
[60977.726260] RDX: 000000007a3bfa68 RSI: ffff9260bca80d48 RDI: ffff92600fd9f0e0
[60977.733392] RBP: ffffa0781992b678 R08: 00000000ffffffff R09: 00000000ffffffff
[60977.740527] R10: 0000000000000001 R11: ffff9231c6d2cc00 R12: ffff92600fd9f0e0
[60977.747661] R13: ffffa0781992b710 R14: 0000000000000001 R15: 000000000019007e
[60977.754794]  adjust_inuse_and_calc_cost+0x164/0x250
[60977.759673]  ioc_rqos_merge+0xef/0x260
[60977.763425]  __rq_qos_merge+0x31/0x50
[60977.767091]  bio_attempt_back_merge+0x64/0xf0
[60977.771451]  blk_mq_sched_try_merge+0x147/0x1d0
[60977.775982]  bfq_bio_merge+0xe2/0x150 [bfq]
[60977.780168]  __blk_mq_sched_bio_merge+0x36/0x1b0
[60977.784790]  blk_mq_submit_bio+0xd5/0x690
[60977.788803]  __submit_bio+0x23d/0x270
[60977.792465]  ? wake_up_page_bit+0xd3/0x100
[60977.796568]  submit_bio_noacct+0x26c/0x2d0
[60977.800664]  ? wake_up_page_bit+0xd3/0x100
[60977.804764]  submit_bio+0x48/0x140
[60977.808171]  iomap_submit_ioend.isra.0+0x52/0x80
[60977.812790]  iomap_writepage_map+0x490/0x630
[60977.817065]  iomap_do_writepage+0x8c/0x260
[60977.821161]  ? clear_page_dirty_for_io+0x113/0x220
[60977.825955]  write_cache_pages+0x19e/0x460
[60977.830054]  ? iomap_truncate_page+0x60/0x60
[60977.834330]  iomap_writepages+0x20/0x40
[60977.838167]  xfs_vm_writepages+0x85/0xb0 [xfs]
[60977.842718]  do_writepages+0xc6/0x1c0
[60977.846384]  ? _raw_spin_unlock+0x23/0x30
[60977.850398]  filemap_fdatawrite_wbc+0x7d/0xc0
[60977.854756]  __filemap_fdatawrite_range+0x54/0x70
[60977.859463]  generic_fadvise+0x299/0x2c0
[60977.863387]  xfs_file_fadvise+0x23/0x80 [xfs]
[60977.867807]  vfs_fadvise+0x1e/0x30
[60977.871214]  ksys_fadvise64_64+0x41/0x80
[60977.875140]  ? syscall_trace_enter.isra.0+0x16b/0x1e0
[60977.880194]  __x64_sys_fadvise64+0x1e/0x30
[60977.884291]  do_syscall_64+0x5c/0xc0
[60977.887870]  ? irqentry_exit_to_user_mode+0x9/0x20
[60977.892665]  ? irqentry_exit+0x19/0x30
[60977.896416]  ? exc_page_fault+0x94/0x1b0
[60977.900343]  ? asm_exc_page_fault+0x8/0x30
[60977.904444]  entry_SYSCALL_64_after_hwframe+0x44/0xae
[60977.909496] RIP: 0033:0x7f16f2c07d3e
[60977.913076] Code: 10 10 00 f7 d8 64 89 02 b8 ff ff ff ff eb b3 e8 28 d8 01 00 0f 1f 84 00 00 00 00 00 f3 0f 1e fa 41 89 ca b8 dd 00 00 00 0f 05 <89> c2 f7 da 3d 00 f0 ff ff b8 00 00 00 00 0f 47 c2 c3 41 57 41 56
[60977.931819] RSP: 002b:00007ffd83ec8338 EFLAGS: 00000246 ORIG_RAX: 00000000000000dd
[60977.939388] RAX: ffffffffffffffda RBX: 00007f16effca890 RCX: 00007f16f2c07d3e
[60977.946518] RDX: 0000000280000000 RSI: 0000000000000000 RDI: 0000000000000006
[60977.953652] RBP: 0000000000000000 R08: 00007f16e8f2b028 R09: 0000000000000000
[60977.960784] R10: 0000000000000004 R11: 0000000000000246 R12: 0000000280000000
[60977.967918] R13: 00007f16e8c16c80 R14: 00007f16effca890 R15: 00007f16e8c16c80
[60977.975053]  </TASK>

```



Kernel dump 3 (enable lockdep):

```shell
[ 4398.422037] WARNING: inconsistent lock state
[ 4398.426311] 5.15.0-26-generic #26 Not tainted
[ 4398.430666] --------------------------------
[ 4398.434939] inconsistent {IN-HARDIRQ-W} -> {HARDIRQ-ON-W} usage.
[ 4398.440947] fio/156503 [HC0[0]:SC0[0]:HE0:SE1] takes:
[ 4398.445997] ffff8b5bd4771c38 (&bfqd->lock){?.-.}-{2:2}, at: bfq_bio_merge+0x9f/0x150 [bfq]
[ 4398.454265] {IN-HARDIRQ-W} state was registered at:
[ 4398.459148]   lock_acquire+0xd8/0x300
[ 4398.462823]   _raw_spin_lock_irqsave+0x52/0xa0
[ 4398.467279]   bfq_idle_slice_timer+0x2d/0xc0 [bfq]
[ 4398.472079]   __hrtimer_run_queues+0x8b/0x420
[ 4398.476438]   hrtimer_interrupt+0x109/0x230
[ 4398.480621]   __sysvec_apic_timer_interrupt+0x78/0x1f0
[ 4398.485761]   sysvec_apic_timer_interrupt+0x77/0x90
[ 4398.490640]   asm_sysvec_apic_timer_interrupt+0x12/0x20
[ 4398.495868]   lock_is_held_type+0x115/0x150
[ 4398.500054]   rcu_read_lock_sched_held+0x5a/0x90
[ 4398.504673]   lock_acquire+0x19f/0x300
[ 4398.508426]   process_one_work+0x28d/0x5d0
[ 4398.512525]   worker_thread+0x4a/0x3d0
[ 4398.516276]   kthread+0x141/0x160
[ 4398.519597]   ret_from_fork+0x22/0x30
[ 4398.523289] irq event stamp: 1141506
[ 4398.526869] hardirqs last  enabled at (1141505): [<ffffffff880849b1>] _raw_spin_unlock_irqrestore+0x51/0x70
[ 4398.536601] hardirqs last disabled at (1141506): [<ffffffff88084744>] _raw_spin_lock_irq+0x74/0x90
[ 4398.545556] softirqs last  enabled at (1139712): [<ffffffff884002f5>] __do_softirq+0x2f5/0x4d3
[ 4398.554160] softirqs last disabled at (1139699): [<ffffffff872c9556>] irq_exit_rcu+0x96/0xc0
[ 4398.562594] 
               other info that might help us debug this:
[ 4398.569121]  Possible unsafe locking scenario:
               
[ 4398.575041]        CPU0
[ 4398.577494]        ----
[ 4398.579947]   lock(&bfqd->lock);
[ 4398.583188]   <Interrupt>
[ 4398.585813]     lock(&bfqd->lock);
[ 4398.589219] 
                *** DEADLOCK ***
               
[ 4398.595137] 2 locks held by fio/156503:
[ 4398.598978]  #0: ffff8b5c653c4e28 (&sb->s_type->i_mutex_key#14){++++}-{3:3}, at: xfs_ilock+0x104/0x1b0 [xfs]
[ 4398.608937]  #1: ffff8b5bd4771c38 (&bfqd->lock){?.-.}-{2:2}, at: bfq_bio_merge+0x9f/0x150 [bfq]
[ 4398.617637] 
               stack backtrace:
[ 4398.621999] CPU: 19 PID: 156503 Comm: fio Kdump: loaded Not tainted 5.15.0-26-generic #26
[ 4398.630170] Hardware name: Supermicro SYS-6029TP-H-EI012/X11DPT-PS-EI012, BIOS 1.0.9.0 01/31/2018
[ 4398.639036] Call Trace:
[ 4398.641488]  <TASK>
[ 4398.643596]  dump_stack_lvl+0x6e/0x9c
[ 4398.647284]  dump_stack+0x10/0x12
[ 4398.650605]  print_usage_bug.part.0+0x18e/0x19d
[ 4398.655138]  mark_lock_irq.cold+0x11/0x2c
[ 4398.659151]  ? stack_trace_save+0x4c/0x70
[ 4398.663164]  ? save_trace+0x45/0x2f0
[ 4398.666742]  mark_lock.part.0+0x11f/0x220
[ 4398.670756]  mark_held_locks+0x54/0x80
[ 4398.674510]  ? adjust_inuse_and_calc_cost+0x164/0x2c0
[ 4398.679563]  lockdep_hardirqs_on_prepare+0x7f/0x1c0
[ 4398.680859] patch_est_timer: disagrees about version of symbol module_layout
[ 4398.684449]  ? _raw_spin_unlock_irq+0x28/0x40
[ 4398.695856]  trace_hardirqs_on+0x21/0xe0
[ 4398.699791]  _raw_spin_unlock_irq+0x28/0x40
[ 4398.703974]  adjust_inuse_and_calc_cost+0x164/0x2c0
[ 4398.708858]  ioc_rqos_merge+0xef/0x280
[ 4398.712608]  __rq_qos_merge+0x31/0x50
[ 4398.716272]  bio_attempt_back_merge+0x63/0x160
[ 4398.720722]  blk_mq_sched_try_merge+0x147/0x1d0
[ 4398.725256]  bfq_bio_merge+0xe2/0x150 [bfq]
[ 4398.729448]  __blk_mq_sched_bio_merge+0x36/0x1b0
[ 4398.734069]  blk_mq_submit_bio+0xd5/0x900
[ 4398.738082]  __submit_bio+0x307/0x340
[ 4398.741746]  ? get_user_pages_fast+0x24/0x40
[ 4398.746018]  ? iov_iter_get_pages+0xc6/0x400
[ 4398.750292]  submit_bio_noacct+0x26c/0x2d0
[ 4398.754393]  submit_bio+0x48/0x140
[ 4398.757797]  iomap_dio_submit_bio+0x7c/0x90
[ 4398.761983]  iomap_dio_bio_iter+0x2da/0x4b0
[ 4398.766172]  __iomap_dio_rw+0x514/0x830
[ 4398.770012]  iomap_dio_rw+0xe/0x30
[ 4398.773413]  xfs_file_dio_write_aligned+0xb9/0x1a0 [xfs]
[ 4398.778821]  ? sched_clock_cpu+0x12/0xe0
[ 4398.782749]  xfs_file_write_iter+0x101/0x1a0 [xfs]
[ 4398.787613]  aio_write+0x131/0x310
[ 4398.791019]  ? __fget_files+0xcf/0x1c0
[ 4398.794771]  __io_submit_one.constprop.0+0x43a/0x570
[ 4398.799740]  ? __io_submit_one.constprop.0+0x43a/0x570
[ 4398.804885]  ? sched_clock_cpu+0x12/0xe0
[ 4398.808819]  ? io_submit_one+0xd9/0x870
[ 4398.812671]  io_submit_one+0x142/0x870
[ 4398.816429]  ? io_submit_one+0x142/0x870
[ 4398.820373]  __x64_sys_io_submit+0x97/0x2a0
[ 4398.824569]  ? syscall_trace_enter.isra.0+0x186/0x250
[ 4398.829632]  do_syscall_64+0x5c/0xc0
[ 4398.833217]  ? __x64_sys_io_cancel+0x250/0x250
[ 4398.837660]  ? do_syscall_64+0x5c/0xc0
[ 4398.841412]  ? do_syscall_64+0x69/0xc0
[ 4398.845165]  ? do_syscall_64+0x69/0xc0
[ 4398.848921]  ? sysvec_call_function_single+0x4e/0x90
[ 4398.853884]  ? asm_sysvec_call_function_single+0xa/0x20
[ 4398.859110]  entry_SYSCALL_64_after_hwframe+0x44/0xae
[ 4398.864165] RIP: 0033:0x7fd43e139697
[ 4398.867746] Code: 00 75 10 8b 47 0c 39 47 08 74 10 0f 1f 84 00 00 00 00 00 e9 bb ff ff ff 0f 1f 00 31 c0 c3 0f 1f 44 00 00 b8 d1 00 00 00 0f 05 <c3> 0f 1f 84 00 00 00 00 00 b8 d2 00 00 00 0f 05 c3 0f 1f 84 00 00
[ 4398.886488] RSP: 002b:00007fd439a49cc8 EFLAGS: 00000202 ORIG_RAX: 00000000000000d1
[ 4398.894057] RAX: ffffffffffffffda RBX: 00007fd43a24c000 RCX: 00007fd43e139697
[ 4398.901187] RDX: 00007fd43400d530 RSI: 0000000000000001 RDI: 00007fd43eb24000
[ 4398.908323] RBP: 00007fd43a24c000 R08: 00007fd43a7051f0 R09: 00000000000001f0
[ 4398.915454] R10: 0000000004455000 R11: 0000000000000202 R12: 00007fd43400ccf0
[ 4398.922587] R13: 00007fd43400d740 R14: 00007fd43400d530 R15: 00007fd43a251210
[ 4398.929725]  </TASK>
```



Lock chain:

1. bfq_bio_merge:

​	spin_lock_irq(&bfqd->lock);

​	blk_mq_sched_try_merge:

​		adjust_inuse_and_calc_cost:

​				spin_lock_irq(&ioc->lock);

​				spin_unlock_irq(&ioc->lock);

2. bfq_finish_requeue_request:

​		spin_lock_irqsave(&bfqd->lock, flags);

​		…

​		spin_unlock_irqrestore(&bfqd->lock, flags);

![img](https://lh7-us.googleusercontent.com/bq2FPR6rf3vTi3qwBroUf_ukX2q8_kuRj8GbANgmURAwdcfd3c6QBivBGFvtvvADlqdTU4CbjvhIBrHzJefKR8UpE5WR7A0jZ7tFH5odAsfF8JjofHbi2vq8AFHgUT6cdfmcOxm0Tc0gyZCdXJGYfog)

There is a process running on cpu and hold a lock spin_lock_irq(&bfqd->lock), then the process to hold the second lock by spin_lock_irq(&ioc->lock), after finish calculation, the process release the second lock spin_unlock_irq(&ioc->lock), at this time the kernel allow irq, and a irq preempts the cpu and when run into function *bfq_finish_requeue_request,* it try to hold the first lock by spin_lock_irqsave(&bfqd->lock, flags), but it is still hold by original process, so it is deadlock. The locked module can detect this case and reprot the risk into dmesg (stack 3). 

### Solution

This kernel patch fix the issue in kernel 6.3.13 version:

*https://lwn.net/Articles/937933/*

[*https://www.spinics.net/lists/stable/msg669695.html*](https://www.spinics.net/lists/stable/msg669695.html)

In this patch, the fix is using *spin_lock_irqsave* to replace *spin_lock_irq* in the function of “*adjust_inuse_and_calc_cost*” to block the irq when unlocked.

We will cherry-pick the patch into our 5.15 kernel.