---
title: Why the cgroup v1 can not throttle the buffer io but cgroup v2 can
update: 2024-05-06 14:42:45
tags: linux
categories: system
---

## Why the cgroup v1 can not throttle the buffer io but cgroup v2 can


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


When call the write syscall with buffer io, vfs call the generic_perform_write function and run the interface write_begin and write_end in proper order. Between them, the raw_copy_from_user is the real data write from the user space into the kernel space of memory area. In the write_end stage, mark_buffer_dirty function marks the memory page and inode is dirty and wait to writeout: 


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


Here the bio has to bind to the IO cgroup, if the css is empty, it will bind to the root cgroup.

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

So in cgroup v2, before entering the block layer, the filesystem layer has decided which inode, which memory cgroup and which IO cgroup bind to this bio. Then it is transfer to direct IO after calling submit_bio. 

