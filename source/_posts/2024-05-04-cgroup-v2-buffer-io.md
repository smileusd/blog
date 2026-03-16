---
title: Cgroup DIsk IO Deep Dive
update: 2024-05-06 14:42:45
tags: linux
categories: system

---



### What the cgroup init difference between v1 and v2

We know the basic buffer IO flow is to write the data into a dirty page on memory, then an asynchronous writeback thread will flush the dirty page onto the disk. Unlike the direct IO, the buffer IO is not only controlled by the io controller but also controlled by the memory controller.

In March 2014, kernel merged the pr[ cgroup: implement unified hierarchy](https://lwn.net/Articles/592434/) about unified architecture in cgroup, that is the basic design of cgroup v2.

There is the code of the main process of cgroup init. In the process of cgroup init in the function of “cgroup_init”, after setting up the cgroup root, initiating the subsystem of the cgroups and creating the mountpoints, the kernel registers the different cgroup filesystems on that mount point according to different cgroup types:


```
int __init cgroup_init(void)
{
           …
    	BUG_ON(cgroup_setup_root(&cgrp_dfl_root, 0));
           …
	for_each_subsys(ss, ssid) {
		if (ss->early_init) {
			struct cgroup_subsys_state *css =
				init_css_set.subsys[ss->id];

			css->id = cgroup_idr_alloc(&ss->css_idr, css, 1, 2,
						   GFP_KERNEL);
			BUG_ON(css->id < 0);
		} else {
			cgroup_init_subsys(ss, false);
		}
                      …
		if (ss->dfl_cftypes == ss->legacy_cftypes) {
			WARN_ON(cgroup_add_cftypes(ss, ss->dfl_cftypes));
		} else {
			WARN_ON(cgroup_add_dfl_cftypes(ss, ss->dfl_cftypes));
			WARN_ON(cgroup_add_legacy_cftypes(ss, ss->legacy_cftypes));
		}
           }
           …
	WARN_ON(sysfs_create_mount_point(fs_kobj, "cgroup"));
	WARN_ON(register_filesystem(&cgroup_fs_type));
	WARN_ON(register_filesystem(&cgroup2_fs_type));
	WARN_ON(!proc_create_single("cgroups", 0, NULL, proc_cgroupstats_show));
}

```


When register_filesystem, cgroup v1 and v2 register different filesystem operations:


```
static const struct fs_context_operations cgroup_fs_context_ops = {
	.free		= cgroup_fs_context_free,
	.parse_param	= cgroup2_parse_param,
	.get_tree	= cgroup_get_tree,
	.reconfigure	= cgroup_reconfigure,
};

static const struct fs_context_operations cgroup1_fs_context_ops = {
	.free		= cgroup_fs_context_free,
	.parse_param	= cgroup1_parse_param,
	.get_tree	= cgroup1_get_tree,
	.reconfigure	= cgroup1_reconfigure,
};
```


When doing the mount, the kernel tries to call the .get_tree interface to bind the filesystem on the directory by do_new_mount -> vfs_get_tree -> fc->ops->get_tree(fc). There are the different call path of the cgroup v1 and v2 implementations of .get_tree:


```
int cgroup1_get_tree(struct fs_context *fc)
{
…
          	ret = cgroup1_root_to_use(fc);
	if (!ret)
		ret = cgroup_do_get_tree(fc);
…
}

/*
 * The guts of cgroup1 mount - find or create cgroup_root to use.
 */
static int cgroup1_root_to_use(struct fs_context *fc)
{
	for_each_root(root) {

		/*
		 * If we asked for a name then it must match.  Also, if
		 * name matches but sybsys_mask doesn't, we should fail.
		 * Remember whether name matched.
		 */
		if (ctx->name) {
			if (strcmp(ctx->name, root->name))
				continue;
			name_match = true;
		}

		ctx->root = root;
		return 0;
	}
            /*
	 * No such thing, create a new one.  
	 */
	root = kzalloc(sizeof(*root), GFP_KERNEL);
	if (!root)
		return -ENOMEM;

	ctx->root = root;
	init_cgroup_root(ctx);

	ret = cgroup_setup_root(root, ctx->subsys_mask);
}
```


Because the mountpoinst are different names, every mount will produce a new cgroup root and the cgroup root link to the subsys of cgroup. The cgroup_root represents the root of a cgroup hierarchy which is organized as a tree. Every tree is a hierarchy structure and it is independent. The child cgroups from cgroup root can only inherit the sussystem with parent cgroup and can not quickly visit other subsystems in cgroup v1. Though it has a root_list to store all the cgroup root, it is hard to cooperate with each other. 


```
/*
 * A cgroup_root represents the root of a cgroup hierarchy, and may be
 * associated with a kernfs_root to form an active hierarchy.  This is
 * internal to cgroup core.  Don't access directly from controllers.
 */
struct cgroup_root {
	struct kernfs_root *kf_root;

	/* The bitmask of subsystems attached to this hierarchy */
	unsigned int subsys_mask;

	/* Unique id for this hierarchy. */
	int hierarchy_id;

	/*
	 * The root cgroup. The containing cgroup_root will be destroyed on its
	 * release. cgrp->ancestors[0] will be used overflowing into the
	 * following field. cgrp_ancestor_storage must immediately follow.
	 */
	struct cgroup cgrp;

	/* must follow cgrp for cgrp->ancestors[0], see above */
	struct cgroup *cgrp_ancestor_storage;

	/* Number of cgroups in the hierarchy, used only for /proc/cgroups */
	atomic_t nr_cgrps;

	/* A list running through the active hierarchies */
	struct list_head root_list;

	/* Hierarchy-specific flags */
	unsigned int flags;

	/* The path to use for release notifications. */
	char release_agent_path[PATH_MAX];

	/* The name for this hierarchy - may be empty */
	char name[MAX_CGROUP_ROOT_NAMELEN];
};

```


But in the cgroup v2, all the subsystems are bind to one cgroup_root because it is only one mountpoint and normally using the default cgroup root:


```
static int cgroup_get_tree(struct fs_context *fc)
{
	struct cgroup_fs_context *ctx = cgroup_fc2context(fc);
	int ret;

	WRITE_ONCE(cgrp_dfl_visible, true);
	cgroup_get_live(&cgrp_dfl_root.cgrp);
	ctx->root = &cgrp_dfl_root;

	ret = cgroup_do_get_tree(fc);
	if (!ret)
		apply_cgroup_root_flags(ctx->flags);
	return ret;
}

/* the default hierarchy */
struct cgroup_root cgrp_dfl_root = { .cgrp.rstat_cpu = &cgrp_dfl_root_rstat_cpu };
EXPORT_SYMBOL_GPL(cgrp_dfl_root);
```


The default cgroup has all subsys_mask. All the child cgroup from the root cgroup will bind the subsystem cgroup by enabling it in the parent cgroup’s cgroup.subtree_control. So the different subsystems can have easier cooperation with each other.


```
int __init cgroup_init(void)
{
	for_each_subsys(ss, ssid) {
		list_add_tail(&init_css_set.e_cset_node[ssid],
			      &cgrp_dfl_root.cgrp.e_csets[ssid]);

		cgrp_dfl_root.subsys_mask |= 1 << ss->id;
	}
}

```



### How kernel implement the cgroup v2 writeback 

After the kernel added a unified cgroup hierarchy, another patch [writeback: cgroup writeback support](https://lwn.net/Articles/628631/) merged in January, 2015. It adds a feature and framework to support the writeback control by cgroup. Then the ext4 filesystem added the support with writeback cgroup [ext4: implement cgroup writeback support](https://lwn.net/Articles/648299/) at Jun, 2015. But xfs filesystem implements the support at [[V2] xfs: implement cgroup writeback support](https://patchwork.kernel.org/project/xfs/patch/f13839700d372a4663a08a11883e9c89b13056ca.1521752282.git.shli@fb.com/) on March, 2018. So on some old kernel versions, the xfs filesystem does not support cgroup writeback. 

 

Now let’s do the deep dive about how the kernel implements the cgroup writeback. The implementation of cgroup writeback is complex with many different packages and adds many new changes after the original patch. Here we just analyze and focus on the latest kernel version and skip the history and change from the first patch.

There is full flow of write:

![cgroupv2_full_flow](/images/cgroupv2_full_flow.png)

As the write as example, there is the simplify flow from syscall of write to enter of block layer:


![cgroupv2_simple_flow](/images/cgroupv2_simple_flow.png)


When calling the write syscall with buffer io, vfs calls the generic_perform_write function and runs the interface write_begin and write_end in proper order. Between them, the raw_copy_from_user is the real data write from the user space into the kernel space of the memory area. In the write_end stage, mark_buffer_dirty function marks the memory page and inode is dirty and wait to writeout: 


```
void mark_buffer_dirty(struct buffer_head *bh)
{
	WARN_ON_ONCE(!buffer_uptodate(bh));

	trace_block_dirty_buffer(bh);

	/*
	 * Very *carefully* optimize the it-is-already-dirty case.
	 *
	 * Don't let the final "is it dirty" escape to before we
	 * perhaps modified the buffer.
	 */
	if (buffer_dirty(bh)) {
		smp_mb();
		if (buffer_dirty(bh))
			return;
	}

	if (!test_set_buffer_dirty(bh)) {
		struct folio *folio = bh->b_folio;
		struct address_space *mapping = NULL;

		folio_memcg_lock(folio);
		if (!folio_test_set_dirty(folio)) {
			mapping = folio->mapping;
			if (mapping)
				__folio_mark_dirty(folio, mapping, 0);
		}
		folio_memcg_unlock(folio);
		if (mapping)
			__mark_inode_dirty(mapping->host, I_DIRTY_PAGES);
	}
}
```


The __mark_inode_dirty function calls the __inode_attach_wb to create a bdi_writeback structure and attach the inode into it. The cgwb_create function creates a bdi_writeback for this memory cgroup and adds it into the bdi->cgwb_tree and by wb_get_lookup to get the bdi_writeback.


```
void __inode_attach_wb(struct inode *inode, struct folio *folio)
{
	struct backing_dev_info *bdi = inode_to_bdi(inode);
	struct bdi_writeback *wb = NULL;

	if (inode_cgwb_enabled(inode)) {
		struct cgroup_subsys_state *memcg_css;

		if (folio) {
			memcg_css = mem_cgroup_css_from_folio(folio);
			wb = wb_get_create(bdi, memcg_css, GFP_ATOMIC);
		} else {
			/* must pin memcg_css, see wb_get_create() */
			memcg_css = task_get_css(current, memory_cgrp_id);
			wb = wb_get_create(bdi, memcg_css, GFP_ATOMIC);
			css_put(memcg_css);
		}
	}

	if (!wb)
		wb = &bdi->wb;

	/*
	 * There may be multiple instances of this function racing to
	 * update the same inode.  Use cmpxchg() to tell the winner.
	 */
	if (unlikely(cmpxchg(&inode->i_wb, NULL, wb)))
		wb_put(wb);
}

//  wb_get_create - get wb for a given memcg, create if necessary
struct bdi_writeback *wb_get_create(struct backing_dev_info *bdi,
				    struct cgroup_subsys_state *memcg_css,
				    gfp_t gfp)
{
	struct bdi_writeback *wb;

	might_alloc(gfp);

	if (!memcg_css->parent)
		return &bdi->wb;

	do {
		wb = wb_get_lookup(bdi, memcg_css);
	} while (!wb && !cgwb_create(bdi, memcg_css, gfp));

	return wb;
}
```


The cgwb_create function is important, because it get the IO subsystem blkcg_css according to the memory cgroup by cgroup_get_e_css. 


```
static int cgwb_create(struct backing_dev_info *bdi,
		       struct cgroup_subsys_state *memcg_css, gfp_t gfp)
{
	struct mem_cgroup *memcg;
	struct cgroup_subsys_state *blkcg_css;
	struct list_head *memcg_cgwb_list, *blkcg_cgwb_list;
	struct bdi_writeback *wb;
	unsigned long flags;
	int ret = 0;

	memcg = mem_cgroup_from_css(memcg_css);
	blkcg_css = cgroup_get_e_css(memcg_css->cgroup, &io_cgrp_subsys);
	memcg_cgwb_list = &memcg->cgwb_list;
	blkcg_cgwb_list = blkcg_get_cgwb_list(blkcg_css);
            …
	ret = wb_init(wb, bdi, gfp);
	wb->memcg_css = memcg_css;
	wb->blkcg_css = blkcg_css;
           …
}

```


For the cgroup v2, the cgroup_get_e_css gets the IO css from the cgroup because all the subsystems bind to the cgroup. But for the cgroup v1, only the subsystem currently binds to the cgroup, so the cgroup_css returns NULL and goes back to the parent cgroup and finally gets the empty cgroup and breaks the loop. Then return the init IO css. But this css doesn't contain the process and cgroup information. I think it is just for compatibility with cgroup v1 to return the init css.


```
// cgroup_get_e_css - get a cgroup's effective css for the specified subsystem
struct cgroup_subsys_state *cgroup_get_e_css(struct cgroup *cgrp,
					     struct cgroup_subsys *ss)
{
	struct cgroup_subsys_state *css;

	if (!CGROUP_HAS_SUBSYS_CONFIG)
		return NULL;

	rcu_read_lock();

	do {
		css = cgroup_css(cgrp, ss);

		if (css && css_tryget_online(css))
			goto out_unlock;
		cgrp = cgroup_parent(cgrp);
	} while (cgrp);

	css = init_css_set.subsys[ss->id];
	css_get(css);
out_unlock:
	rcu_read_unlock();
	return css;
}
```


Besides binding the cgroup, bdi and inode, the bdi_writeback needs to attach a real worker function to flush the diary page out to disk. This step at wb_init:


```
	INIT_DELAYED_WORK(&wb->dwork, wb_workfn);
	INIT_DELAYED_WORK(&wb->bw_dwork, wb_update_bandwidth_workfn);
```


The __mark_inode_dirty adds this work into the delayed work queue for the idle worker to do the flush work. The wb_update_bandwidth_workfn according to the writeback IO time, dirty writeback numbers and the completion writeback numbers to calculate the bandwidth of the writeback and update the dirty_ratelimit. There are some algorithms but not the point in this topic, just skip them.


```
	static void __wb_update_bandwidth(struct dirty_throttle_control *gdtc,
				  struct dirty_throttle_control *mdtc,
				  bool update_ratelimit)
{
            /*
	 * Lockless checks for elapsed time are racy and delayed update after
	 * IO completion doesn't do it at all (to make sure written pages are
	 * accounted reasonably quickly). Make sure elapsed >= 1 to avoid
	 * division errors.
	 */
	elapsed = max(now - wb->bw_time_stamp, 1UL);
	dirtied = percpu_counter_read(&wb->stat[WB_DIRTIED]);
	written = percpu_counter_read(&wb->stat[WB_WRITTEN]);
           …
}
```



```
/*
 * Maintain wb->dirty_ratelimit, the base dirty throttle rate.
 *
 * Normal wb tasks will be curbed at or below it in long term.
 * Obviously it should be around (write_bw / N) when there are N dd tasks.
 */
static void wb_update_dirty_ratelimit(struct dirty_throttle_control *dtc,
				      unsigned long dirtied,
				      unsigned long elapsed)
{
	balanced_dirty_ratelimit = div_u64((u64)task_ratelimit * write_bw,
					   dirty_rate | 1);
	WRITE_ONCE(wb->dirty_ratelimit, max(dirty_ratelimit, 1UL));
	wb->balanced_dirty_ratelimit = balanced_dirty_ratelimit;
}
```


Now the wb has been prepared, it links to the real worker waiting to write out. It links to the inode, bdi, memory cgroup and IO cgoup. They can provide the information when writing out.

The wb_workfn function runs the __writeback_single_inode and calls the interface of writepages or writepage.  The different file system and backend implement the interfaces. But they have a similar process including wbc_init_bio and wbc_account_cgroup_owner before submitting the bio. 

![cgroupv2_simple_flow2](/images/cgroupv2_simple_flow2.png)


These two functions are patches added to the xfs and ext4 cgroup writeback support. We can see the cgroup v2 documentation about [Filesystem Support for Writeback](https://www.kernel.org/doc/Documentation/cgroup-v2.txt):


```
Filesystem Support for Writeback
--------------------------------

A filesystem can support cgroup writeback by updating
address_space_operations->writepage[s]() to annotate bio's using the
following two functions.

  wbc_init_bio(@wbc, @bio)
	Should be called for each bio carrying writeback data and
	associates the bio with the inode's owner cgroup and the
	corresponding request queue.  This must be called after
	a queue (device) has been associated with the bio and
	before submission.

  wbc_account_cgroup_owner(@wbc, @page, @bytes)
	Should be called for each data segment being written out.
	While this function doesn't care exactly when it's called
	during the writeback session, it's the easiest and most
	natural to call it as data segments are added to a bio.
```


The wbc is writeback_control which manages and controls total writeback flow and stores the bdi_writeback and inode. 


```
/*
 * A control structure which tells the writeback code what to do.  These are
 * always on the stack, and hence need no locking.  They are always initialised
 * in a manner such that unspecified fields are set to zero.
 */
struct writeback_control {
	long nr_to_write;		/* Write this many pages, and decrement
					   this for each page written */
            …
#ifdef CONFIG_CGROUP_WRITEBACK
	struct bdi_writeback *wb;	/* wb this writeback is issued under */
	struct inode *inode;		/* inode being written out */
#endif
};

```


wbc_init_bio() binds the specified bio to its cgroup which binds at cgwb_create function.  


```
static inline void wbc_init_bio(struct writeback_control *wbc, struct bio *bio)
{
	/*
	 * pageout() path doesn't attach @wbc to the inode being written
	 * out.  This is intentional as we don't want the function to block
	 * behind a slow cgroup.  Ultimately, we want pageout() to kick off
	 * regular writeback instead of writing things out itself.
	 */
	if (wbc->wb)
		bio_associate_blkg_from_css(bio, wbc->wb->blkcg_css);
}
```



```
//  bio_associate_blkg_from_css - associate a bio with a specified css
void bio_associate_blkg_from_css(struct bio *bio,
				 struct cgroup_subsys_state *css)
{
	if (bio->bi_blkg)
		blkg_put(bio->bi_blkg);

	if (css && css->parent) {
		bio->bi_blkg = blkg_tryget_closest(bio, css);
	} else {
		blkg_get(bdev_get_queue(bio->bi_bdev)->root_blkg);
		bio->bi_blkg = bdev_get_queue(bio->bi_bdev)->root_blkg;
	}
}
```


Here the bio binds to the IO cgroup on bi_blkg, if the css is empty, it will bind to the root cgroup.

The wbc_account_cgroup_owner is a solution for this question: if multiple processes from different cgroup write into the same inode, how to decide who is the owner of this inode right now. The wbc_account_cgroup_owner counts the pages to different cgroup by memory id. After finishing the current writeback, the wbc_detach_inode function uses Boyer-Moore majority vote algorithm to select the owner of this inode, then calls inode_switch_wbs to switch the bdi_writeback for the inode.


```
//  wbc_account_cgroup_owner - account writeback to update inode cgroup ownership
void wbc_account_cgroup_owner(struct writeback_control *wbc, struct page *page,
			      size_t bytes)
{
	struct folio *folio;
	struct cgroup_subsys_state *css;
	int id;

	/*
	 * pageout() path doesn't attach @wbc to the inode being written
	 * out.  This is intentional as we don't want the function to block
	 * behind a slow cgroup.  Ultimately, we want pageout() to kick off
	 * regular writeback instead of writing things out itself.
	 */
	if (!wbc->wb || wbc->no_cgroup_owner)
		return;

	folio = page_folio(page);
	css = mem_cgroup_css_from_folio(folio);
	/* dead cgroups shouldn't contribute to inode ownership arbitration */
	if (!(css->flags & CSS_ONLINE))
		return;

	id = css->id;

	if (id == wbc->wb_id) {
		wbc->wb_bytes += bytes;
		return;
	}

	if (id == wbc->wb_lcand_id)
		wbc->wb_lcand_bytes += bytes;

	/* Boyer-Moore majority vote algorithm */
	if (!wbc->wb_tcand_bytes)
		wbc->wb_tcand_id = id;
	if (id == wbc->wb_tcand_id)
		wbc->wb_tcand_bytes += bytes;
	else
		wbc->wb_tcand_bytes -= min(bytes, wbc->wb_tcand_bytes);
}
```



```
/**
 * mem_cgroup_css_from_folio - css of the memcg associated with a folio
 * @folio: folio of interest
 *
 * If memcg is bound to the default hierarchy, css of the memcg associated
 * with @folio is returned.  The returned css remains associated with @folio
 * until it is released.
 *
 * If memcg is bound to a traditional hierarchy, the css of root_mem_cgroup
 * is returned.
 */
struct cgroup_subsys_state *mem_cgroup_css_from_folio(struct folio *folio)
{
	struct mem_cgroup *memcg = folio_memcg(folio);

	if (!memcg || !cgroup_subsys_on_dfl(memory_cgrp_subsys))
		memcg = root_mem_cgroup;

	return &memcg->css;
}
```


The cgroup v1 returns the root cgroup as well because it can not handle this case.

So in cgroup v2, before entering the block layer, the filesystem layer has decided which IO cgroup binds to this bio according to the inode and memory cgroup. The bio contains the io cgroup and then it is transferred to direct IO after calling submit_bio. 


```
struct bio {
…
#ifdef CONFIG_BLK_CGROUP
	/*
	 * Represents the association of the css and request_queue for the bio.
	 * If a bio goes direct to device, it will not have a blkg as it will
	 * not have a request_queue associated with it.  The reference is put
	 * on release of the bio.
	 */
	struct blkcg_gq		*bi_blkg;
	struct bio_issue	bi_issue;
…
}
```



### How IO throttling in block layer

There are the different throttling files with cgroup v1 and v2


```
static struct cftype throtl_legacy_files[] = {
	{
		.name = "throttle.read_bps_device",
		.private = offsetof(struct throtl_grp, bps[READ][LIMIT_MAX]),
		.seq_show = tg_print_conf_u64,
		.write = tg_set_conf_u64,
	},
	{
		.name = "throttle.write_bps_device",
		.private = offsetof(struct throtl_grp, bps[WRITE][LIMIT_MAX]),
		.seq_show = tg_print_conf_u64,
		.write = tg_set_conf_u64,
	},
	{
		.name = "throttle.read_iops_device",
		.private = offsetof(struct throtl_grp, iops[READ][LIMIT_MAX]),
		.seq_show = tg_print_conf_uint,
		.write = tg_set_conf_uint,
	},
	{
		.name = "throttle.write_iops_device",
		.private = offsetof(struct throtl_grp, iops[WRITE][LIMIT_MAX]),
		.seq_show = tg_print_conf_uint,
		.write = tg_set_conf_uint,
	},
	{
		.name = "throttle.io_service_bytes",
		.private = offsetof(struct throtl_grp, stat_bytes),
		.seq_show = tg_print_rwstat,
	},
	{
		.name = "throttle.io_service_bytes_recursive",
		.private = offsetof(struct throtl_grp, stat_bytes),
		.seq_show = tg_print_rwstat_recursive,
	},
	{
		.name = "throttle.io_serviced",
		.private = offsetof(struct throtl_grp, stat_ios),
		.seq_show = tg_print_rwstat,
	},
	{
		.name = "throttle.io_serviced_recursive",
		.private = offsetof(struct throtl_grp, stat_ios),
		.seq_show = tg_print_rwstat_recursive,
	},
	{ }	/* terminate */
};
```



```
static struct cftype throtl_files[] = {
	{
		.name = "max",
		.flags = CFTYPE_NOT_ON_ROOT,
		.seq_show = tg_print_limit,
		.write = tg_set_limit,
		.private = LIMIT_MAX,
	},
	{ }	/* terminate */
};
```


In cgroup v2 we only need to set all the disk IO limitations like read/write bps and iops on io.max file. Once we set the io.max, all the blk cgroup configs are set to the throtl_grp structure by  tg_set_limit. Here the doc focusing on the cgroup v2 implementation. 

When registering the blk throttle to the disk, it sets the throtl_slice according to different disk types. That is the default window of calculating the throttling IO.


```
void blk_throtl_register(struct gendisk *disk)
{
	struct request_queue *q = disk->queue;
	struct throtl_data *td;
	int i;

	td = q->td;
	BUG_ON(!td);

	if (blk_queue_nonrot(q)) {
		td->throtl_slice = DFL_THROTL_SLICE_SSD;
		td->filtered_latency = LATENCY_FILTERED_SSD;
	} else {
		td->throtl_slice = DFL_THROTL_SLICE_HD;
		td->filtered_latency = LATENCY_FILTERED_HD;
           }
}

```


Following the tg_set_limit function, the tg_conf_updated is important function to update the throtl_grp and calculate the wait time and dispatch time of the bio.  


```
static void tg_conf_updated(struct throtl_grp *tg, bool global)
{
	struct throtl_service_queue *sq = &tg->service_queue;
	struct cgroup_subsys_state *pos_css;
	struct blkcg_gq *blkg;

	throtl_log(&tg->service_queue,
		   "limit change rbps=%llu wbps=%llu riops=%u wiops=%u",
		   tg_bps_limit(tg, READ), tg_bps_limit(tg, WRITE),
		   tg_iops_limit(tg, READ), tg_iops_limit(tg, WRITE));

	blkg_for_each_descendant_pre(blkg, pos_css,
			global ? tg->td->queue->root_blkg : tg_to_blkg(tg)) {
		struct throtl_grp *this_tg = blkg_to_tg(blkg);
		struct throtl_grp *parent_tg;

		tg_update_has_rules(this_tg);
		/* ignore root/second level */
		if (!cgroup_subsys_on_dfl(io_cgrp_subsys) || !blkg->parent ||
		    !blkg->parent->parent)
			continue;
		parent_tg = blkg_to_tg(blkg->parent);
		/*
		 * make sure all children has lower idle time threshold and
		 * higher latency target
		 */
		this_tg->idletime_threshold = min(this_tg->idletime_threshold,
				parent_tg->idletime_threshold);
		this_tg->latency_target = max(this_tg->latency_target,
				parent_tg->latency_target);
	}

	/*
	 * We're already holding queue_lock and know @tg is valid.  Let's
	 * apply the new config directly.
	 *
	 * Restart the slices for both READ and WRITES. It might happen
	 * that a group's limit are dropped suddenly and we don't want to
	 * account recently dispatched IO with new low rate.
	 */
	throtl_start_new_slice(tg, READ, false);
	throtl_start_new_slice(tg, WRITE, false);

	if (tg->flags & THROTL_TG_PENDING) {
		tg_update_disptime(tg);
		throtl_schedule_next_dispatch(sq->parent_sq, true);
	}
}

```



```
static void tg_update_disptime(struct throtl_grp *tg)
{
	struct throtl_service_queue *sq = &tg->service_queue;
	unsigned long read_wait = -1, write_wait = -1, min_wait = -1, disptime;
	struct bio *bio;

	bio = throtl_peek_queued(&sq->queued[READ]);
	if (bio)
		tg_may_dispatch(tg, bio, &read_wait);

	bio = throtl_peek_queued(&sq->queued[WRITE]);
	if (bio)
		tg_may_dispatch(tg, bio, &write_wait);

	min_wait = min(read_wait, write_wait);
	disptime = jiffies + min_wait;

	/* Update dispatch time */
	throtl_rb_erase(&tg->rb_node, tg->service_queue.parent_sq);
	tg->disptime = disptime;
	tg_service_queue_add(tg);

	/* see throtl_add_bio_tg() */
	tg->flags &= ~THROTL_TG_WAS_EMPTY;
}
```


The dispatch time use the minimum of the write wait time and read wait time calculated by the tg_may_dispatch. Then add the throtl_grp into the service queue wait to dispatch. Every update the io.max, the kernel need new window slice to throttling. 


```
static inline void throtl_start_new_slice(struct throtl_grp *tg, bool rw,
					  bool clear_carryover)
{
	tg->bytes_disp[rw] = 0;
	tg->io_disp[rw] = 0;
	tg->slice_start[rw] = jiffies;
	tg->slice_end[rw] = jiffies + tg->td->throtl_slice;
	if (clear_carryover) {
		tg->carryover_bytes[rw] = 0;
		tg->carryover_ios[rw] = 0;
	}
}
```


The global variable [jiffies](https://litux.nl/mirror/kerneldevelopment/0672327201/ch10lev1sec3.html) holds the number of ticks that have occurred since the system booted. On boot, the kernel initializes the variable to zero, and it is incremented by one during each timer interrupt. Thus, because there are HZ timer interrupts in a second, there are HZ jiffies in a second. The system uptime is therefore jiffies/HZ seconds.

It is similar for the disk IO to the cpu: dividing the disk resource into time according to the disk frequency. The “time” is not the real linear time but it is “jiffy” which the numbers of the tick from uptime. So the uptime = jiffies / HZ.

The core algorithm is that for every cgroup for this disk, kernel use sibling window (slice) to decide how much time the bio need to wait or just dispatch. It calculates the max bps or iops in this window. If the latest bio is exceed the numbers of the bps or iops, it extends the window and wait time of the window end. So it is not the fixed slice except the new slice of beginning. The details of the implement is in the tg_may_dispatch. 


```
/*
 * Returns whether one can dispatch a bio or not. Also returns approx number
 * of jiffies to wait before this bio is with-in IO rate and can be dispatched
 */
static bool tg_may_dispatch(struct throtl_grp *tg, struct bio *bio,
			    unsigned long *wait)
{
	bool rw = bio_data_dir(bio);
	unsigned long bps_wait = 0, iops_wait = 0, max_wait = 0;
	u64 bps_limit = tg_bps_limit(tg, rw);
	u32 iops_limit = tg_iops_limit(tg, rw);

	/*
 	 * Currently whole state machine of group depends on first bio
	 * queued in the group bio list. So one should not be calling
	 * this function with a different bio if there are other bios
	 * queued.
	 */
	BUG_ON(tg->service_queue.nr_queued[rw] &&
	       bio != throtl_peek_queued(&tg->service_queue.queued[rw]));

	/* If tg->bps = -1, then BW is unlimited */
	if ((bps_limit == U64_MAX && iops_limit == UINT_MAX) ||
	    tg->flags & THROTL_TG_CANCELING) {
		if (wait)
			*wait = 0;
		return true;
	}

	/*
	 * If previous slice expired, start a new one otherwise renew/extend
	 * existing slice to make sure it is at least throtl_slice interval
	 * long since now. New slice is started only for empty throttle group.
	 * If there is queued bio, that means there should be an active
	 * slice and it should be extended instead.
	 */
	if (throtl_slice_used(tg, rw) && !(tg->service_queue.nr_queued[rw]))
		throtl_start_new_slice(tg, rw, true);
	else {
		if (time_before(tg->slice_end[rw],
		    jiffies + tg->td->throtl_slice))
			throtl_extend_slice(tg, rw,
				jiffies + tg->td->throtl_slice);
	}

	bps_wait = tg_within_bps_limit(tg, bio, bps_limit);
	iops_wait = tg_within_iops_limit(tg, bio, iops_limit);
	if (bps_wait + iops_wait == 0) {
		if (wait)
			*wait = 0;
		return true;
	}

	max_wait = max(bps_wait, iops_wait);

	if (wait)
		*wait = max_wait;

	if (time_before(tg->slice_end[rw], jiffies + max_wait))
		throtl_extend_slice(tg, rw, jiffies + max_wait);

	return false;
}

```


Now we look back to the bio follow and the blk_throtl_bio is between the A and Q. 


![images/throttling](images/throttling.png)


If set to io.max, every bio goes through the _blk_throtl_bio and runs tg_may_dispatch to check whether the bio can be dispatched directly or not. If not, calculate the wait time and update the timer with the minimum wait time from the rb_tree. Finally it adds the bio into the service queue and waits for the timer to run the next dispatch loop.


![blk_throttleing](images/blk_throttling.png)



### Disk IO QoS

Both of them are controlled by the rq_qos structure:


```
struct rq_qos {
	const struct rq_qos_ops *ops;
	struct gendisk *disk;
	enum rq_qos_id id;
	struct rq_qos *next;
#ifdef CONFIG_BLK_DEBUG_FS
	struct dentry *debugfs_dir;
#endif
};

struct rq_qos_ops {
	void (*throttle)(struct rq_qos *, struct bio *);
	void (*track)(struct rq_qos *, struct request *, struct bio *);
	void (*merge)(struct rq_qos *, struct request *, struct bio *);
	void (*issue)(struct rq_qos *, struct request *);
	void (*requeue)(struct rq_qos *, struct request *);
	void (*done)(struct rq_qos *, struct request *);
	void (*done_bio)(struct rq_qos *, struct bio *);
	void (*cleanup)(struct rq_qos *, struct bio *);
	void (*queue_depth_changed)(struct rq_qos *);
	void (*exit)(struct rq_qos *);
	const struct blk_mq_debugfs_attr *debugfs_attrs;
};
```


The rq_qos is the singly linked list. There are 3 rq_qos plugins that can be used: blk-iolatency, blk-iocost and blk-wbt.


```
static struct cftype iolatency_files[] = {
	{
		.name = "latency",
		.flags = CFTYPE_NOT_ON_ROOT,
		.seq_show = iolatency_print_limit,
		.write = iolatency_set_limit,
	},
	{}
};
```



```
static struct cftype ioc_files[] = {
	{
		.name = "weight",
		.flags = CFTYPE_NOT_ON_ROOT,
		.seq_show = ioc_weight_show,
		.write = ioc_weight_write,
	},
	{
		.name = "cost.qos",
		.flags = CFTYPE_ONLY_ON_ROOT,
		.seq_show = ioc_qos_show,
		.write = ioc_qos_write,
	},
	{
		.name = "cost.model",
		.flags = CFTYPE_ONLY_ON_ROOT,
		.seq_show = ioc_cost_model_show,
		.write = ioc_cost_model_write,
	},
	{}
};
```


We have learned the block layer basic follow and we can know the place of rq_qos throttling. 


![alt_text](images/disk_qos_1.png)


IO QoS has complicated algorithms for different policy and implementation. Here is just making an entrance but not deep diving. I will make another article to talk about the disk IO QoS.
