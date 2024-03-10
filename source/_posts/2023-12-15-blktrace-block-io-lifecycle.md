---
title: blktrace block io lifecycle
update: 2023-12-29 14:42:45
tags: linux
categories: system
---

### Question: trace the life of IO by blktrace/blkparse

[blktrace](https://linux.die.net/man/8/blktrace)

[blkparse](https://linux.die.net/man/1/blkparse)

Investigate the blktrace implementation to understand the trace hook of the blk IO stack?



### blktrace to parse the block io

`blktrace` explores the Linux kernel tracepoint infrastructure to track requests in-flight through the block I/O stack. It traces everything that goes through to block devices, while observing timing information. 

First we using `dd` command to make a write IO (the read IO is the same). 

```shell
// write to a file with direct io by dd command
# dd if=/dev/zero of=testfile bs=16k count=1024000 oflag=direct
1024000+0 records in
1024000+0 records out
16777216000 bytes (17 GB, 16 GiB) copied, 111.433 s, 151 MB/s
```

Using`blktrace` and `blkparse` analize results:

```shell
# blktrace -d /dev/vda -o res
# blkparse -i res
device  
252,0   10        3     0.000238760       0  C  WS 49724200 + 32 [0]
252,0    0       17     0.000261770 3418552  A  WS 48720712 + 32 <- (253,0) 48718664
252,0    0       18     0.000262000 3418552  A  WS 49724232 + 32 <- (252,3) 48720712
252,0    0       19     0.000262192 3418552  Q  WS 49724232 + 32 [dd]
252,0    0       20     0.000263132 3418552  G  WS 49724232 + 32 [dd]
252,0    0       21     0.000263335 3418552  P   N [dd]
252,0    0       22     0.000263494 3418552  U   N [dd] 1
252,0    0       23     0.000263719 3418552  I  WS 49724232 + 32 [dd]
252,0    0       24     0.000264296 3418552  D  WS 49724232 + 32 [dd]
252,0   10        4     0.000341566       0  C  WS 49724232 + 32 [0]
252,0    0       25     0.000366914 3418552  A  WS 48720744 + 32 <- (253,0) 48718696
252,0    0       26     0.000367235 3418552  A  WS 49724264 + 32 <- (252,3) 48720744
252,0    0       27     0.000367410 3418552  Q  WS 49724264 + 32 [dd]
252,0    0       28     0.000368423 3418552  G  WS 49724264 + 32 [dd]
252,0    0       29     0.000368612 3418552  P   N [dd]
252,0    0       30     0.000368797 3418552  U   N [dd] 1
252,0    0       31     0.000369036 3418552  I  WS 49724264 + 32 [dd]
252,0    0       32     0.000369730 3418552  D  WS 49724264 + 32 [dd]
252,0   10        5     0.000453876       0  C  WS 49724264 + 32 [0]
...
252,0   10    91033     2.279081418 3405085  C  WS 50399976 + 32 [0]
252,0   10   232460     5.618988454 3405085  C  WS 51349952 + 32 [0]
252,0   10   318363     8.424729857 3405085  C  WM 39888904 + 16 [0]
252,0   10   318364     8.424732534 3405085  C  WM 267886593 + 1 [0]
252,0   10   318365     8.424734981 3405085  C  WM 267886600 + 16 [0]
252,0   10   318366     8.424737948 3405085  C  WM 267925376 + 32 [0]
252,0   10   318367     8.424750755 3405085  C  WM 267924512 + 32 [0]

...
Total (res):
 Reads Queued:           0,        0KiB  Writes Queued:       70567,     1128MiB
 Read Dispatches:        3,        0KiB  Write Dispatches:    70559,     1128MiB
 Reads Requeued:         0               Writes Requeued:         0
 Reads Completed:        3,        0KiB  Writes Completed:    70563,     1128MiB
 Read Merges:            0,        0KiB  Write Merges:            6,       24KiB
 IO unplugs:         70545               Timer unplugs:           0


# tracing 267886608
252,0    6       71     8.424115456   585  A  WM 267886608 + 8 <- (252,3) 266883088
252,0    6       72     8.424115630   585  Q  WM 267886608 + 8 [xfsaild/dm-0]
252,0    6       73     8.424115962   585  M  WM 267886608 + 8 [xfsaild/dm-0]

# tracing 4555007
252,0    9        2     8.423915195 3374227  A WFSM 4555007 + 31 <- (252,3) 3551487
252,0    9        3     8.423915869 3374227  Q WFSM 4555007 + 31 [kworker/9:4]
252,0    9        4     8.423918485 3374227  G WFSM 4555007 + 31 [kworker/9:4]
252,0    9        5     8.423927401   240  D WSM 4555007 + 31 [kworker/9:1H]
252,0   10   318350     8.424051685     0  C WSM 4555007 + 31 [0]
252,0   10   318353     8.424169747     0  C WSM 4555007 [0]
```

![*Screenshot 2023-12-25 at 13.37.09*](/Users/tashen/Library/Application Support/typora-user-images/Screenshot 2023-12-25 at 13.37.09.png)

#### ACTION IDENTIFIERS

```
The following table shows the various actions which may be
       output:

       A      IO was remapped to a different device

       B      IO bounced

       C      IO completion

       D      IO issued to driver

       F      IO front merged with request on queue

       G      Get request

       I      IO inserted onto request queue

       M      IO back merged with request on queue

       P      Plug request

       Q      IO handled by request queue code

       S      Sleep request

       T      Unplug due to timeout

       U      Unplug request

       X      Split
```

The result is long and there only paste few of them. I try to trace an different sectors and behaviors of block layer.

`blkparse` command still hard to know the total metrics and statistics, so we can use `btt` to see the statistic of whole table and picture, this is the part of results:

```shell
# blkparse -q -i res   -d sda.blk.bin
# btt -i sda.blk.bin

==================== All Devices ====================

            ALL           MIN           AVG           MAX           N
--------------- ------------- ------------- ------------- -----------

Q2Q               0.000001441   0.000119382   0.802839654       70564
Q2G               0.000000497   0.000000797   0.000025123       70559
G2I               0.000000406   0.000000654   0.000128249       70558
Q2M               0.000000133   0.000000271   0.000000631           6
I2D               0.000000411   0.000000663   0.000017267       70558
M2D               0.000031518   0.000071988   0.000116699           6
D2C               0.000064651   0.000079818   0.002640604       70565
Q2C               0.000066315   0.000081939   0.002643331       70565

==================== Device Overhead ====================

       DEV |       Q2G       G2I       Q2M       I2D       D2C
---------- | --------- --------- --------- --------- ---------
 (252,  0) |   0.9723%   0.7983%   0.0000%   0.8096%  97.4121%
---------- | --------- --------- --------- --------- ---------
   Overall |   0.9723%   0.7983%   0.0000%   0.8096%  97.4121%

==================== Device Merge Information ====================

       DEV |       #Q       #D   Ratio |   BLKmin   BLKavg   BLKmax    Total
---------- | -------- -------- ------- | -------- -------- -------- --------
 (252,  0) |    70565    70559     1.0 |        1       31       32  2257605
```

```java
 Q------->G------------>I--------->M------------------->D----------------------------->C
 |-Q time-|-Insert time-|
 |--------- merge time ------------|-merge with other IO|
 |----------------scheduler time time-------------------|---driver,adapter,storagetime--|
 
 |----------------------- await time in iostat output ----------------------------------|
```

Q2Q — time between requests sent to the block layer 

Q2G — time from a block I/O is queued to the time it gets a request allocated for it

G2I — time from a request is allocated to the time it is inserted into the device's queue 

Q2M — time from a block I/O is queued to the time it gets merged with an existing request 

I2D — time from a request is inserted into the device's queue to the time it is actually issued to the device (time the I/O is “idle” on the request queue)

M2D — time from  a block I/O is merged with an exiting request until the request is issued to the device 

D2C — service time of the request by the device (time the I/O is “active” in the driver and on the device)

Q2C — total time spent in the block layer  for a request

Actually the blktrace's implement is using various linux kernel tracepoint to trace different phase of IO. Here are some of the key tracepoints used by `blktrace`:

1. `block_rq_insert`: This tracepoint is hit when a request is inserted into the request queue.(Q)
2. `block_rq_issue`: This tracepoint is hit when a request is issued to the device.(I)
3. `block_rq_complete`: This tracepoint is hit when a request is completed.(C)
4. `block_bio_queue`: This tracepoint is hit when a bio is queued.(Q)
5. `block_bio_backmerge`: This tracepoint is hit when a bio is being merged with the last bio in the request.
6. `block_bio_frontmerge`: This tracepoint is hit when a bio is being merged with the first bio in the request.
7. `block_bio_bounce`: This tracepoint is hit when a bio is bounced.
8. `block_getrq`: This tracepoint is hit when get_request() is called to allocate a request.
9. `block_sleeprq`: This tracepoint is hit when a process is going to sleep waiting for a request to be available.
10. `block_plug`: This tracepoint is hit when a plug is inserted into the request queue.
11. `block_unplug`: This tracepoint is hit when a plug is removed from the request queue.

### Main data structure in block layer

To deepdiv the block layer, i read the code and finding as followes:

```c
/*
 * main unit of I/O for the block layer and lower layers (ie drivers and
 * stacking drivers)
 */
struct bio {
	struct bio		*bi_next;	/* request queue link */
	struct block_device	*bi_bdev;
  unsigned long       bi_rw; /* read or write */
  ...
	struct bio_vec		*bi_io_vec;	/* the actual vec list */
  struct bvec_iter	bi_iter;

};

/**
 * struct bio_vec - a contiguous range of physical memory addresses
 * @bv_page:   First page associated with the address range.
 * @bv_len:    Number of bytes in the address range.
 * @bv_offset: Start of the address range relative to the start of @bv_page.
 *
 * The following holds for a bvec if n * PAGE_SIZE < bv_offset + bv_len:
 *
 *   nth_page(@bv_page, n) == @bv_page + n
 *
 * This holds because page_is_mergeable() checks the above property.
 */
struct bio_vec {
	struct page	*bv_page;
	unsigned int	bv_len;
	unsigned int	bv_offset;
};

struct bvec_iter {
	sector_t		bi_sector;	/* device address in 512 byte
						   sectors */
	unsigned int		bi_size;	/* residual I/O count */

	unsigned int		bi_idx;		/* current index into bvl_vec */

	unsigned int            bi_bvec_done;	/* number of bytes completed in
						   current bvec */
} __packed;

struct block_device {
	sector_t		bd_start_sect;
	sector_t		bd_nr_sectors;
	struct gendisk *	bd_disk;
	struct request_queue *	bd_queue;
  bool			bd_has_submit_bio;
  ...
} __randomize_layout;

/*
 * Try to put the fields that are referenced together in the same cacheline.
 *
 * If you modify this structure, make sure to update blk_rq_init() and
 * especially blk_mq_rq_ctx_init() to take care of the added fields.
 */
struct request {
	struct request_queue *q;
  struct bio *bio;
	struct bio *biotail;
	union {
		struct list_head queuelist;
		struct request *rq_next;
	};
	enum mq_rq_state state;
	/*
	 * The hash is used inside the scheduler, and killed once the
	 * request reaches the dispatch list. The ipi_list is only used
	 * to queue the request for softirq completion, which is long
	 * after the request has been unhashed (and even removed from
	 * the dispatch list).
	 */
	union {
		struct hlist_node hash;	/* merge hash */
		struct llist_node ipi_list;
	};

	/*
	 * The rb_node is only used inside the io scheduler, requests
	 * are pruned when moved to the dispatch queue. So let the
	 * completion_data share space with the rb_node.
	 */
	union {
		struct rb_node rb_node;	/* sort/lookup */
		struct bio_vec special_vec;
		void *completion_data;
	};
  /*
	 * completion callback.
	 */
	rq_end_io_fn *end_io;
}

struct request_queue {
	struct request		*last_merge;
	struct elevator_queue	*elevator;
	struct rq_qos		*rq_qos;
	const struct blk_mq_ops	*mq_ops;
	struct queue_limits	limits;
	/*
	 * for flush operations
	 */
	struct blk_flush_queue	*fq;
}

/**
 * struct blk_mq_hw_ctx - State for a hardware queue facing the hardware
 * block device
 */
struct blk_mq_hw_ctx {
	/**
	 * @queue: Pointer to the request queue that owns this hardware context.
	 */
	struct request_queue	*queue;
  	/**
	 * @dispatch_busy: Number used by blk_mq_update_dispatch_busy() to
	 * decide if the hw_queue is busy using Exponential Weighted Moving
	 * Average algorithm.
	 */
	unsigned int		dispatch_busy;
}

```

![bio逻辑架构](https://s3.bmp.ovh/imgs/2021/09/09b73ed1c3ac78a1.webp)

### code flow in block layer

![img](https://pic2.zhimg.com/80/v2-5b898fe018554749f56c270e77261a4d_1440w.webp)



Bio ->  Request -> plug request list -> staging request queue in sheduler -> hardware request queue

![Screenshot 2023-12-25 at 13.54.30](/Users/tashen/Library/Application Support/typora-user-images/Screenshot 2023-12-25 at 13.54.30.png)

![Screenshot 2023-12-25 at 13.57.23](/Users/tashen/Library/Application Support/typora-user-images/Screenshot 2023-12-25 at 13.57.23.png)

These pictures records the total main code and function flow from the filessystem to block layer and to driver layder. It also prints the tracepoint mapping to the acation in output of the blktrace  where they trace from. By these tracepoint, we can understand the block io process flow more clearly. 
