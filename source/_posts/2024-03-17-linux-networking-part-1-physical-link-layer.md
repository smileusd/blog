---
title: "Linux Networking Deep Dive: Part 1 - Physical & Link Layer"
date: 2024-03-17 10:00:00
categories:
  - systems-programming
  - networking
tags:
  - linux-networking
  - kernel-development
  - network-programming
  - sk_buff
  - ethernet-drivers
series: "Linux Networking Deep Dive"
part: 1
description: "Explore Linux network devices, sk_buff internals, and Ethernet driver implementation. Foundation layer for understanding kernel networking."
---

**Linux Networking Deep Dive Series:**
[Introduction]({% post_path 2024-03-16-linux-networking-deep-dive-introduction %}) |
**Part 1: Physical & Link Layer** |
[Part 2: Network Layer]({% post_path 2024-03-18-linux-networking-part-2-network-layer %}) |
[Part 3: Transport Layer]({% post_path 2024-03-19-linux-networking-part-3-transport-layer %}) |
[Part 4: Application Interface]({% post_path 2024-03-20-linux-networking-part-4-application-interface %}) |
[Part 5: Advanced Features]({% post_path 2024-03-21-linux-networking-part-5-advanced-features %}) |
[Tools Guide]({% post_path 2024-03-22-linux-networking-tools-and-scripts-guide %}) |
[Reference Guide]({% post_path 2024-03-23-linux-networking-reference-guide %}) |
[Code Analysis]({% post_path 2024-03-24-linux-networking-code-analysis-walkthrough %})

---

# Part 1: Physical & Link Layer - The Foundation

Welcome to the foundation of Linux networking! In this first installment, we dive deep into the physical and link layers - the bedrock upon which all network communication in Linux is built. Understanding these layers is crucial because every packet that flows through your system, whether it's a simple ping or high-frequency trading data, starts its journey here.

## The Big Picture

Before we dive into the specifics, let's understand where the physical and link layers fit in the Linux networking stack:

1. **Hardware**: Network interface cards (NICs) handle the physical transmission
2. **Link Layer** (our focus): Ethernet frames, device drivers, packet buffers
3. **Network Layer**: IP processing and routing (covered in Part 2)
4. **Transport Layer**: TCP/UDP protocols (covered in Part 3)
5. **Application Layer**: Sockets and system calls (covered in Part 4)

In this post, we'll explore three fundamental components that make it all work: network device abstraction, the sk_buff structure, and Ethernet driver implementation.

## Network Device Abstraction

Linux treats network interfaces as abstract devices through the `struct net_device`, providing a uniform interface regardless of whether you're dealing with Ethernet, WiFi, or even virtual interfaces.

### The net_device Structure

At the heart of every network interface is `struct net_device` defined in `include/linux/netdevice.h`:

```c
struct net_device {
    char name[IFNAMSIZ];                // Interface name (e.g., "eth0")
    int ifindex;                        // Unique interface index
    unsigned int flags;                 // Interface flags (UP/DOWN, etc.)
    unsigned int mtu;                   // Maximum transmission unit
    unsigned short type;                // Hardware type (ARPHRD_*)
    unsigned char *dev_addr;            // Hardware address (MAC)
    const struct net_device_ops *netdev_ops; // Device operations

    /* Transmit queues */
    struct netdev_queue *_tx;           // TX queue array
    unsigned int num_tx_queues;         // Number of TX queues
    unsigned int real_num_tx_queues;    // Currently active TX queues

    /* Receive handling */
    struct netdev_queue *_rx;           // RX queue array
    unsigned int num_rx_queues;         // Number of RX queues

    /* Statistics */
    struct net_device_stats stats;      // Interface statistics
    atomic_long_t rx_dropped;          // RX dropped counter
    atomic_long_t tx_dropped;          // TX dropped counter

    /* Feature capabilities */
    netdev_features_t features;         // Current feature set
    netdev_features_t hw_features;      // Hardware-supported features
};
```

### Network Device Operations

The `netdev_ops` structure defines the interface between the kernel and device drivers:

```c
struct net_device_ops {
    int (*ndo_open)(struct net_device *dev);           // Bring interface up
    int (*ndo_stop)(struct net_device *dev);           // Bring interface down
    netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
                                  struct net_device *dev); // Transmit packet
    int (*ndo_set_mac_address)(struct net_device *dev,
                              void *addr);              // Set MAC address
    void (*ndo_set_rx_mode)(struct net_device *dev);   // Set receive mode
    int (*ndo_change_mtu)(struct net_device *dev,
                         int new_mtu);                  // Change MTU
    void (*ndo_tx_timeout)(struct net_device *dev);    // Handle TX timeout
    void (*ndo_get_stats64)(struct net_device *dev,
                           struct rtnl_link_stats64 *storage); // Get statistics
};
```

### Device Registration Process

When a network driver initializes, it follows this sequence:

```c
/* 1. Allocate net_device structure */
struct net_device *dev = alloc_netdev(sizeof(struct driver_private),
                                      "eth%d", NET_NAME_UNKNOWN,
                                      setup_function);

/* 2. Initialize device-specific fields */
dev->netdev_ops = &device_netdev_ops;
dev->mtu = ETH_DATA_LEN;
dev->type = ARPHRD_ETHER;
dev->hard_header_len = ETH_HLEN;

/* 3. Set hardware address */
memcpy(dev->dev_addr, hardware_mac_address, ETH_ALEN);

/* 4. Register with kernel */
int err = register_netdev(dev);
if (err) {
    free_netdev(dev);
    return err;
}
```

## The Heart of Networking: sk_buff

The `sk_buff` (socket buffer) is arguably the most important data structure in Linux networking. Every network packet, from the moment it arrives at the hardware until it reaches an application, is represented as an sk_buff.

### sk_buff Architecture

The sk_buff is designed for efficiency, with carefully arranged fields to optimize memory access patterns:

```c
struct sk_buff {
    union {
        struct {
            struct sk_buff *next;        // Next buffer in list
            struct sk_buff *prev;        // Previous buffer in list
            struct net_device *dev;      // Associated network device
        };
        struct rb_node rbnode;           // For RB-tree storage (TCP)
    };

    struct sock *sk;                     // Associated socket
    ktime_t tstamp;                     // Packet timestamp

    /* Control buffer - 48 bytes for layer-specific data */
    char cb[48] __aligned(8);

    /* Core packet information */
    unsigned int len;                    // Total packet length
    unsigned int data_len;              // Length of paged data
    __u16 mac_len;                      // MAC header length
    __u16 hdr_len;                      // Length of cloned headers

    /* Checksum information */
    union {
        __wsum csum;                     // Checksum value
        struct {
            __u16 csum_start;            // Checksum start offset
            __u16 csum_offset;           // Checksum field offset
        };
    };

    __u8 ip_summed:2;                   // Checksum status
    __u8 cloned:1;                      // Head may be cloned
    __u8 pkt_type:3;                    // Packet classification

    /* Header pointers */
    __u16 transport_header;             // Transport layer header offset
    __u16 network_header;               // Network layer header offset
    __u16 mac_header;                  // Link layer header offset

    /* Buffer management */
    sk_buff_data_t tail;                // Tail pointer
    sk_buff_data_t end;                 // End pointer
    unsigned char *head;                // Head of buffer
    unsigned char *data;                // Data head pointer
    unsigned int truesize;              // Total buffer size
    refcount_t users;                   // Reference count
};
```

### Memory Layout and Pointers

The sk_buff uses a sophisticated pointer system to manage packet data efficiently:

```
      head                              end
       |                                |
       v                                v
   +---+--------------------------------+
   |   | headroom |  data  | tailroom  |
   +---+--------------------------------+
            ^               ^
            |               |
          data            tail

   mac_header -----> |eth|
   network_header -> |ip |
   transport_header->|tcp|
```

This design allows:
- **Efficient header addition/removal**: Simply adjust pointers
- **Zero-copy operations**: Share data between multiple sk_buffs
- **Memory management**: Reference counting for safe memory handling

### sk_buff Operations

Key operations for manipulating sk_buff data:

```c
/* Add data to packet */
unsigned char *skb_put(struct sk_buff *skb, unsigned int len);
void *skb_put_zero(struct sk_buff *skb, unsigned int len);

/* Add data to front */
unsigned char *skb_push(struct sk_buff *skb, unsigned int len);

/* Remove data from front */
unsigned char *skb_pull(struct sk_buff *skb, unsigned int len);

/* Reserve headroom */
void skb_reserve(struct sk_buff *skb, int len);

/* Clone sk_buff (share data, copy control structure) */
struct sk_buff *skb_clone(struct sk_buff *skb, gfp_t gfp_mask);

/* Copy entire sk_buff */
struct sk_buff *skb_copy(const struct sk_buff *skb, gfp_t gfp_mask);
```

### Header Management

Setting and accessing protocol headers:

```c
/* Set header pointers */
void skb_reset_mac_header(struct sk_buff *skb);
void skb_reset_network_header(struct sk_buff *skb);
void skb_reset_transport_header(struct sk_buff *skb);

/* Access protocol headers */
struct ethhdr *eth_hdr(const struct sk_buff *skb);
struct iphdr *ip_hdr(const struct sk_buff *skb);
struct tcphdr *tcp_hdr(const struct sk_buff *skb);
struct udphdr *udp_hdr(const struct sk_buff *skb);

/* Protocol type determination */
__be16 eth_type_trans(struct sk_buff *skb, struct net_device *dev);
```

## Ethernet Driver Implementation

Let's examine how a typical Ethernet driver works, using the Intel e1000e driver as our example.

### Driver Initialization

```c
static int e1000_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
{
    struct net_device *netdev;
    struct e1000_adapter *adapter;
    struct e1000_hw *hw;
    int err;

    /* Allocate network device */
    netdev = alloc_etherdev(sizeof(struct e1000_adapter));
    if (!netdev) {
        err = -ENOMEM;
        goto err_alloc_etherdev;
    }

    SET_NETDEV_DEV(netdev, &pdev->dev);

    pci_set_drvdata(pdev, netdev);
    adapter = netdev_priv(netdev);
    adapter->netdev = netdev;
    adapter->pdev = pdev;

    /* Setup network device operations */
    netdev->netdev_ops = &e1000e_netdev_ops;
    e1000e_set_ethtool_ops(netdev);
    netdev->watchdog_timeo = 5 * HZ;
    netdev->min_mtu = ETH_ZLEN - ETH_HLEN;
    netdev->max_mtu = MAX_JUMBO_FRAME_SIZE - (ETH_HLEN + ETH_FCS_LEN);

    /* Initialize hardware */
    hw = &adapter->hw;
    err = e1000e_setup_init_funcs(hw, true);
    if (err)
        goto err_init;

    /* Register network device */
    err = register_netdev(netdev);
    if (err)
        goto err_register;

    return 0;
}
```

### Transmit Path

The transmit path shows how packets flow from the kernel to the hardware:

```c
static netdev_tx_t e1000_xmit_frame(struct sk_buff *skb,
                                    struct net_device *netdev)
{
    struct e1000_adapter *adapter = netdev_priv(netdev);
    struct e1000_ring *tx_ring = adapter->tx_ring;
    unsigned int first;
    unsigned int tx_flags = 0;
    unsigned int len = skb_headlen(skb);
    unsigned int nr_frags;
    unsigned int mss;
    int count = 0;
    int tso;

    /* Check for available descriptors */
    if (e1000_maybe_stop_tx(tx_ring, count + 2)) {
        /* Not enough descriptors available */
        return NETDEV_TX_BUSY;
    }

    /* Handle TSO (TCP Segmentation Offload) */
    mss = skb_shinfo(skb)->gso_size;
    if (mss) {
        tso = e1000_tso(tx_ring, skb, &tx_flags);
        if (tso < 0) {
            dev_kfree_skb_any(skb);
            return NETDEV_TX_OK;
        }
    }

    /* Handle checksum offload */
    if (skb->ip_summed == CHECKSUM_PARTIAL) {
        if (e1000_tx_csum(tx_ring, skb, &tx_flags) < 0) {
            dev_kfree_skb_any(skb);
            return NETDEV_TX_OK;
        }
    }

    /* Map sk_buff to DMA */
    first = tx_ring->next_to_use;
    if (e1000_tx_map(tx_ring, skb, first, adapter->tx_fifo_limit,
                     nr_frags, mss)) {
        /* DMA mapping failed */
        dev_kfree_skb_any(skb);
        tx_ring->buffer_info[first].time_stamp = 0;
        tx_ring->next_to_use = first;
        return NETDEV_TX_OK;
    }

    /* Update statistics */
    netdev->stats.tx_packets++;
    netdev->stats.tx_bytes += skb->len;

    return NETDEV_TX_OK;
}
```

### Receive Path with NAPI

Modern drivers use NAPI (New API) for efficient packet reception:

```c
static int e1000_clean_rx_irq(struct e1000_adapter *adapter,
                              struct e1000_rx_ring *rx_ring,
                              int budget)
{
    struct net_device *netdev = adapter->netdev;
    struct e1000_rx_desc *rx_desc, *next_rxd;
    struct e1000_rx_buffer *buffer_info, *next_buffer;
    u32 length;
    unsigned int i;
    int cleaned_count = 0;
    bool cleaned = false;
    unsigned int total_rx_bytes = 0, total_rx_packets = 0;

    i = rx_ring->next_to_clean;
    rx_desc = E1000_RX_DESC(*rx_ring, i);

    while (rx_desc->status & E1000_RXD_STAT_DD) {
        struct sk_buff *skb;

        /* Don't exceed budget */
        if (cleaned_count >= budget)
            break;

        cleaned_count++;
        cleaned = true;

        buffer_info = &rx_ring->buffer_info[i];

        /* Allocate new sk_buff */
        skb = netdev_alloc_skb_ip_align(netdev, length);
        if (!skb) {
            /* Allocation failed - drop packet */
            adapter->alloc_rx_buff_failed++;
            break;
        }

        /* Copy data from DMA buffer to sk_buff */
        skb_put(skb, length);
        memcpy(skb->data, buffer_info->skb->data, length);

        /* Set up sk_buff fields */
        skb->protocol = eth_type_trans(skb, netdev);

        /* Handle hardware checksum */
        e1000_receive_skb(adapter, netdev, skb, staterr,
                         rx_desc->special);

        /* Update statistics */
        total_rx_bytes += skb->len;
        total_rx_packets++;

        /* Submit packet to network stack */
        napi_gro_receive(&adapter->napi, skb);

        /* Move to next descriptor */
        rx_desc->status = 0;

        i++;
        if (i == rx_ring->count)
            i = 0;

        rx_desc = E1000_RX_DESC(*rx_ring, i);
    }

    rx_ring->next_to_clean = i;

    /* Update statistics */
    u64_stats_update_begin(&rx_ring->syncp);
    rx_ring->stats.packets += total_rx_packets;
    rx_ring->stats.bytes += total_rx_bytes;
    u64_stats_update_end(&rx_ring->syncp);

    return cleaned;
}
```

### Interrupt Handling

Network drivers must efficiently handle interrupts:

```c
static irqreturn_t e1000_intr(int irq, void *data)
{
    struct net_device *netdev = data;
    struct e1000_adapter *adapter = netdev_priv(netdev);
    struct e1000_hw *hw = &adapter->hw;
    u32 icr = er32(ICR);

    if (unlikely((!icr)))
        return IRQ_NONE;  /* Not our interrupt */

    /* Disable interrupts */
    if (unlikely(icr & (E1000_ICR_RXSEQ | E1000_ICR_LSC))) {
        hw->mac.get_link_status = true;
        /* Link status change interrupt */
        schedule_work(&adapter->watchdog_task);
    }

    /* Schedule NAPI for packet processing */
    if (icr & E1000_IMS_RXT0) {
        /* Received packet interrupt */
        if (napi_schedule_prep(&adapter->napi)) {
            adapter->total_rx_bytes = 0;
            adapter->total_rx_packets = 0;
            __napi_schedule(&adapter->napi);
        }
    }

    return IRQ_HANDLED;
}
```

## DMA and Memory Management

Modern network drivers use DMA (Direct Memory Access) for efficient data transfer:

```c
/* DMA mapping for transmission */
static int e1000_tx_map(struct e1000_ring *tx_ring, struct sk_buff *skb,
                       unsigned int first, unsigned int max_per_txd,
                       unsigned int nr_frags, unsigned int mss)
{
    struct e1000_buffer *buffer_info;
    unsigned int len = skb_headlen(skb);
    unsigned int offset = 0, size, count = 0, i;
    unsigned int f, bytecount, segs;

    /* Map main skb data */
    buffer_info = &tx_ring->buffer_info[first];
    buffer_info->length = len;
    buffer_info->time_stamp = jiffies;
    buffer_info->next_to_watch = first;
    buffer_info->skb = skb;

    buffer_info->dma = dma_map_single(&adapter->pdev->dev,
                                     skb->data, len,
                                     DMA_TO_DEVICE);
    if (dma_mapping_error(&adapter->pdev->dev, buffer_info->dma))
        goto dma_error;

    /* Map fragment pages if present */
    for (f = 0; f < nr_frags; f++) {
        const struct skb_frag_struct *frag;

        frag = &skb_shinfo(skb)->frags[f];
        len = skb_frag_size(frag);
        offset = 0;

        while (len) {
            i++;
            if (i == tx_ring->count)
                i = 0;

            buffer_info = &tx_ring->buffer_info[i];
            size = min(len, max_per_txd);

            buffer_info->length = size;
            buffer_info->time_stamp = jiffies;
            buffer_info->dma = skb_frag_dma_map(&adapter->pdev->dev,
                                               frag, offset, size,
                                               DMA_TO_DEVICE);

            len -= size;
            offset += size;
            count++;
        }
    }

    return count;

dma_error:
    dev_err(&adapter->pdev->dev, "TX DMA map failed\n");
    /* Unmap any mapped buffers */
    buffer_info->dma = 0;
    if (count)
        count--;

    while (count--) {
        if (i == 0)
            i += tx_ring->count;
        i--;
        buffer_info = &tx_ring->buffer_info[i];
        e1000_unmap_and_free_tx_resource(tx_ring, buffer_info);
    }

    return 0;
}
```

## Performance Considerations

### Multi-Queue Support

Modern NICs support multiple transmit and receive queues for improved performance:

```c
/* Initialize multiple TX queues */
static int e1000e_setup_tx_resources(struct e1000_adapter *adapter)
{
    int err = 0, i;

    for (i = 0; i < adapter->num_tx_queues; i++) {
        err = e1000_setup_tx_resources(adapter->tx_ring[i]);
        if (err) {
            e_err("Allocation for TX Queue %u failed\n", i);
            for (i--; i >= 0; i--)
                e1000_free_tx_resources(adapter->tx_ring[i]);
            break;
        }
    }

    return err;
}

/* CPU affinity for interrupt handling */
static void e1000_setup_msix(struct e1000_adapter *adapter)
{
    struct net_device *netdev = adapter->netdev;
    int vector = 0;

    /* Set up RX interrupts */
    for (i = 0; i < adapter->num_rx_queues; i++) {
        struct e1000_ring *rx_ring = adapter->rx_ring[i];
        rx_ring->ims_val = E1000_IMS_RXT0;
        adapter->msix_entries[vector].entry = vector;
        adapter->msix_entries[vector].vector = 0;
        vector++;
    }
}
```

### Cache Optimization

Drivers optimize for CPU cache efficiency:

```c
/* Prefetch next descriptors */
static void e1000_rx_desc_prefetch(struct e1000_ring *rx_ring, int cleaned_count)
{
    unsigned int i = rx_ring->next_to_use;

    if (cleaned_count >= E1000_RX_BUFFER_WRITE) {
        /* Prefetch next cache line of descriptors */
        prefetch(&rx_ring->desc[i]);
    }
}

/* Align sk_buff data for optimal cache usage */
skb = netdev_alloc_skb_ip_align(netdev, length);
```

## Debugging and Monitoring

### Interface Statistics

Monitor interface health through statistics:

```bash
# View interface statistics
cat /proc/net/dev
ip -s link show eth0
ethtool -S eth0

# Monitor packet drops
netstat -i
dropwatch -l kas
```

### Driver-Level Debugging

```c
/* Enable detailed logging */
#define e_dbg(format, arg...) \
    netdev_dbg(adapter->netdev, format, ##arg)

/* Performance counters */
struct e1000_stats {
    char stat_string[ETH_GSTRING_LEN];
    int stat_offset;
};

#define E1000_STAT(_name, _stat) { \
    .stat_string = _name, \
    .stat_offset = offsetof(struct e1000_adapter, _stat) \
}

static const struct e1000_stats e1000_gstrings_stats[] = {
    E1000_STAT("rx_packets", stats.gprc),
    E1000_STAT("tx_packets", stats.gptc),
    E1000_STAT("rx_bytes", stats.gorc),
    E1000_STAT("tx_bytes", stats.gotc),
    E1000_STAT("rx_errors", stats.rxerrc),
    E1000_STAT("tx_errors", stats.txerrc),
};
```

## Try It Yourself

### Exercise 1: Monitor sk_buff Allocation
Create a simple module that tracks sk_buff allocations:

```bash
# Enable sk_buff tracing
echo 1 > /sys/kernel/debug/tracing/events/skb/kfree_skb/enable
echo 1 > /sys/kernel/debug/tracing/events/skb/consume_skb/enable

# Monitor trace output
cat /sys/kernel/debug/tracing/trace_pipe
```

### Exercise 2: Analyze Network Interface
Examine your network interface characteristics:

```bash
# Check interface capabilities
ethtool eth0

# View queue information
ls /sys/class/net/eth0/queues/

# Monitor real-time statistics
watch -n 1 'cat /proc/net/dev'
```

### Exercise 3: Packet Analysis
Use the provided tools to analyze packet flow:

```bash
# From our tools directory
sudo ./packet-tracer.sh -i eth0 -d 30 -f "icmp"

# Analyze the results
less /tmp/packet-trace-*/kernel_trace.txt
```

## Common Issues and Troubleshooting

### Ring Buffer Overruns
Monitor and tune ring buffer sizes:

```bash
# Check current ring buffer sizes
ethtool -g eth0

# Increase ring buffer size
ethtool -G eth0 rx 4096 tx 4096
```

### Interrupt Coalescing
Balance latency vs. throughput:

```bash
# Check current interrupt coalescing
ethtool -c eth0

# Reduce interrupt rate for high throughput
ethtool -C eth0 adaptive-rx on rx-usecs 50
```

### Memory Allocation Failures
Monitor sk_buff allocation failures:

```bash
# Check allocation statistics
cat /proc/net/sockstat
cat /proc/slabinfo | grep skbuff

# Monitor memory pressure
echo 1 > /proc/sys/vm/oom_dump_tasks
```

## What's Next

You now have a solid foundation in Linux networking's physical and link layers. You understand:

- How network devices are abstracted and managed in the kernel
- The sk_buff structure that carries every packet through the system
- How Ethernet drivers efficiently move packets between hardware and kernel
- Performance considerations and optimization techniques

In **Part 2**, we'll build on this foundation to explore the network layer, where we'll see how IP packets are processed, how routing decisions are made, and how the netfilter framework enables packet filtering and modification.

The journey from hardware to application is complex, but understanding these fundamental building blocks gives you the tools to debug performance issues, optimize network applications, and truly understand how Linux networking works under the hood.

---

**Next:** [Part 2: Network Layer & IP Processing →]({% post_path 2024-03-18-linux-networking-part-2-network-layer %})

**Series Navigation:**
[Introduction]({% post_path 2024-03-16-linux-networking-deep-dive-introduction %}) |
**Part 1: Physical & Link Layer** |
[Part 2: Network Layer]({% post_path 2024-03-18-linux-networking-part-2-network-layer %}) |
[Part 3: Transport Layer]({% post_path 2024-03-19-linux-networking-part-3-transport-layer %}) |
[Part 4: Application Interface]({% post_path 2024-03-20-linux-networking-part-4-application-interface %}) |
[Part 5: Advanced Features]({% post_path 2024-03-21-linux-networking-part-5-advanced-features %}) |
[Tools Guide]({% post_path 2024-03-22-linux-networking-tools-and-scripts-guide %}) |
[Reference Guide]({% post_path 2024-03-23-linux-networking-reference-guide %}) |
[Code Analysis]({% post_path 2024-03-24-linux-networking-code-analysis-walkthrough %})