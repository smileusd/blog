---
title: "Linux Networking Reference Guide - Quick Reference for Data Structures, Functions, and Commands"
date: 2024-03-23 09:00:00
tags:
  - Linux
  - Networking
  - Reference
  - Data Structures
  - System Calls
  - Functions
  - Debugging
categories:
  - Linux Networking Series
description: "Comprehensive quick reference guide for Linux networking data structures, kernel functions, system calls, and debugging commands. Essential resource for network developers and system administrators."
toc: true
---

Welcome to the comprehensive Linux networking reference guide! This post serves as your quick lookup resource for essential data structures, kernel functions, system calls, and debugging commands that form the foundation of Linux networking. Whether you're debugging network issues, developing network applications, or diving deep into kernel networking code, this reference will be your indispensable companion.

This guide is designed for quick lookups and serves as a practical reference for the concepts covered throughout our networking series. Keep this bookmarked for easy access during development and troubleshooting sessions.

## How to Use This Guide

This reference is organized into logical sections:

- **Data Structures**: Core kernel networking structures and their key fields
- **Function Reference**: Essential kernel functions for network programming
- **System Calls**: User-space networking APIs and their implementations
- **Debugging Commands**: Practical commands for network analysis and troubleshooting

Each section includes code snippets, parameter descriptions, and usage examples to help you quickly find the information you need.

## Core Data Structures Quick Reference

### Socket Buffer (sk_buff) - The Heart of Packet Processing

The `sk_buff` structure is the fundamental unit of packet processing in the Linux kernel:

```c
struct sk_buff {
    struct sk_buff *next, *prev;     // List pointers for queue management
    struct net_device *dev;          // Associated network device
    struct sock *sk;                 // Associated socket (if any)
    ktime_t tstamp;                 // Packet timestamp

    /* Buffer management */
    unsigned int len;               // Total data length
    unsigned int data_len;          // Length of paged data
    unsigned char *head, *data;     // Buffer start and data start
    sk_buff_data_t tail, end;       // Tail and end pointers

    /* Protocol headers */
    __u16 transport_header;         // Transport layer header offset
    __u16 network_header;           // Network layer header offset
    __u16 mac_header;              // MAC layer header offset
    __be16 protocol;               // Packet protocol type
    __u8 pkt_type;                 // Packet type classification

    refcount_t users;              // Reference count
};
```

**Essential sk_buff Macros:**
```c
skb_put(skb, len)      // Add data to end of buffer
skb_push(skb, len)     // Add space at beginning
skb_pull(skb, len)     // Remove data from beginning
skb_reserve(skb, len)  // Reserve space at head
skb_trim(skb, len)     // Trim buffer to length
```

**Header Access Functions:**
```c
struct ethhdr *eth_hdr(const struct sk_buff *skb);
struct iphdr *ip_hdr(const struct sk_buff *skb);
struct tcphdr *tcp_hdr(const struct sk_buff *skb);
struct udphdr *udp_hdr(const struct sk_buff *skb);
```

### Network Device (net_device) Structure

The `net_device` structure represents a network interface:

```c
struct net_device {
    char name[IFNAMSIZ];            // Interface name (e.g., "eth0")
    int ifindex;                    // Unique interface index
    unsigned int flags;             // Interface flags and capabilities
    unsigned int mtu;               // Maximum transmission unit
    unsigned short type;            // Hardware type (ARPHRD_ETHER, etc.)
    unsigned char *dev_addr;        // Hardware address (MAC)

    /* Operations */
    const struct net_device_ops *netdev_ops;

    /* Queuing */
    struct netdev_queue *_tx;       // Transmit queues
    unsigned int num_tx_queues;     // Number of TX queues
    unsigned int real_num_tx_queues; // Currently active TX queues

    /* Statistics */
    struct net_device_stats stats;  // Basic statistics
    atomic_long_t rx_dropped;       // Dropped received packets
    atomic_long_t tx_dropped;       // Dropped transmitted packets
};
```

**Common Interface Flags:**
```c
IFF_UP          // Interface is up and running
IFF_BROADCAST   // Broadcast capability
IFF_MULTICAST   // Multicast capability
IFF_PROMISC     // Promiscuous mode enabled
IFF_LOOPBACK    // Loopback interface
IFF_NOARP       // No ARP protocol needed
```

### Socket (sock) Structure

The core socket structure used by all protocol families:

```c
struct sock {
    struct sock_common __sk_common; // Common socket fields
    socket_lock_t sk_lock;          // Socket synchronization

    /* Queues */
    struct sk_buff_head sk_receive_queue; // Receive queue
    struct sk_buff_head sk_write_queue;   // Write queue
    struct sk_buff_head sk_error_queue;   // Error queue

    /* Buffer management */
    int sk_rcvbuf, sk_sndbuf;      // Receive and send buffer sizes
    atomic_t sk_rmem_alloc;        // Receive memory allocated
    atomic_t sk_wmem_alloc;        // Write memory allocated

    /* Socket options and state */
    unsigned long sk_flags;         // Socket flags
    struct sk_filter __rcu *sk_filter; // Socket filter (BPF)
    struct socket *sk_socket;       // BSD socket interface
    void *sk_user_data;            // User-defined data

    /* Callbacks */
    int (*sk_backlog_rcv)(struct sock *, struct sk_buff *);
    void (*sk_destruct)(struct sock *);
    void (*sk_error_report)(struct sock *);
    void (*sk_data_ready)(struct sock *);
    void (*sk_write_space)(struct sock *);
    void (*sk_state_change)(struct sock *);
};
```

**Socket Types:**
```c
SOCK_STREAM    // TCP - reliable, connection-oriented
SOCK_DGRAM     // UDP - unreliable, connectionless
SOCK_RAW       // Raw sockets - direct IP access
SOCK_SEQPACKET // Reliable sequenced packets
SOCK_PACKET    // Obsolete packet interface
```

### TCP Socket (tcp_sock) Structure

TCP-specific socket information:

```c
struct tcp_sock {
    struct inet_connection_sock inet_conn; // Inherit from inet socket

    /* Sequence numbers */
    u32 rcv_nxt;                   // Next expected sequence number
    u32 snd_nxt;                   // Next sequence number to send
    u32 snd_una;                   // First unacknowledged sequence
    u32 snd_up;                    // Urgent pointer

    /* Congestion control */
    u32 snd_cwnd;                  // Congestion window
    u32 snd_ssthresh;              // Slow start threshold
    u32 prior_cwnd;                // Prior congestion window
    u32 prr_delivered;             // Number of packets delivered in Recovery

    /* Round trip time */
    u32 srtt_us;                   // Smoothed round trip time
    u32 mdev_us;                   // Medium deviation
    u32 rttvar_us;                 // Round trip time variance
    u32 rtt_seq;                   // RTT measurement sequence number

    /* Packet tracking */
    u32 packets_out;               // Packets which are "in flight"
    u32 retrans_out;               // Retransmitted packets out
    u32 max_packets_out;           // max packets_out in last window
    u32 max_packets_seq;           // Right edge of max_packets_out flight

    /* Statistics */
    u32 total_retrans;             // Total retransmissions for entire connection
};
```

**TCP States:**
```c
TCP_ESTABLISHED  // Normal data transfer state
TCP_SYN_SENT     // Client waiting for matching connection request
TCP_SYN_RECV     // Server waiting for confirming connection request
TCP_FIN_WAIT1    // Waiting for remote TCP termination request
TCP_FIN_WAIT2    // Waiting for remote TCP termination request
TCP_TIME_WAIT    // Waiting to ensure remote received acknowledgment
TCP_CLOSE        // No connection state
TCP_CLOSE_WAIT   // Waiting for local user termination request
TCP_LAST_ACK     // Waiting for remote termination acknowledgment
TCP_LISTEN       // Waiting for incoming connection requests
TCP_CLOSING      // Waiting for remote termination acknowledgment
```

## Protocol Headers Reference

### Ethernet Header

```c
struct ethhdr {
    unsigned char h_dest[ETH_ALEN];    // Destination MAC (6 bytes)
    unsigned char h_source[ETH_ALEN];  // Source MAC (6 bytes)
    __be16 h_proto;                    // Protocol type (2 bytes)
} __attribute__((packed));
```

**Common EtherType Values:**
```c
ETH_P_IP     0x0800   // IPv4
ETH_P_IPV6   0x86DD   // IPv6
ETH_P_ARP    0x0806   // Address Resolution Protocol
ETH_P_8021Q  0x8100   // VLAN-tagged frame
ETH_P_PPP_SES 0x8864  // PPPoE session
```

### IPv4 Header

```c
struct iphdr {
    __u8 ihl:4,              // Internet Header Length
         version:4;          // Version (4 for IPv4)
    __u8 tos;               // Type of Service
    __be16 tot_len;         // Total Length
    __be16 id;              // Identification
    __be16 frag_off;        // Fragment Offset
    __u8 ttl;               // Time to Live
    __u8 protocol;          // Protocol
    __sum16 check;          // Header Checksum
    __be32 saddr;           // Source Address
    __be32 daddr;           // Destination Address
};
```

**Common IP Protocol Numbers:**
```c
IPPROTO_ICMP    1   // Internet Control Message Protocol
IPPROTO_TCP     6   // Transmission Control Protocol
IPPROTO_UDP     17  // User Datagram Protocol
IPPROTO_GRE     47  // Generic Routing Encapsulation
IPPROTO_ESP     50  // Encapsulating Security Protocol
IPPROTO_AH      51  // Authentication Header
```

### TCP Header

```c
struct tcphdr {
    __be16 source;          // Source port
    __be16 dest;            // Destination port
    __be32 seq;             // Sequence number
    __be32 ack_seq;         // Acknowledgment number
    __u16 res1:4,           // Reserved
          doff:4,           // Data offset (header length)
          fin:1,            // Finish - no more data
          syn:1,            // Synchronize sequence numbers
          rst:1,            // Reset connection
          psh:1,            // Push data to application
          ack:1,            // Acknowledgment field valid
          urg:1,            // Urgent pointer field valid
          ece:1,            // ECN-Echo
          cwr:1;            // Congestion Window Reduced
    __be16 window;          // Window size
    __sum16 check;          // Checksum
    __be16 urg_ptr;         // Urgent pointer
};
```

### UDP Header

```c
struct udphdr {
    __be16 source;          // Source port
    __be16 dest;            // Destination port
    __be16 len;             // UDP length
    __sum16 check;          // Checksum
};
```

## Essential Kernel Functions Reference

### Socket Buffer Management

**Allocation and Deallocation:**
```c
/* Allocate new sk_buff */
struct sk_buff *alloc_skb(unsigned int size, gfp_t priority);
struct sk_buff *netdev_alloc_skb(struct net_device *dev, unsigned int length);

/* Free sk_buff */
void kfree_skb(struct sk_buff *skb);        // Free with drop accounting
void consume_skb(struct sk_buff *skb);      // Free normally consumed skb

/* Clone and copy */
struct sk_buff *skb_clone(struct sk_buff *skb, gfp_t priority);
struct sk_buff *skb_copy(const struct sk_buff *skb, gfp_t priority);
```

**Buffer Manipulation:**
```c
/* Data management */
unsigned char *skb_put(struct sk_buff *skb, unsigned int len);
void *skb_put_zero(struct sk_buff *skb, unsigned int len);
unsigned char *skb_push(struct sk_buff *skb, unsigned int len);
unsigned char *skb_pull(struct sk_buff *skb, unsigned int len);
void skb_reserve(struct sk_buff *skb, int len);
void skb_trim(struct sk_buff *skb, unsigned int len);

/* Header management */
void skb_reset_mac_header(struct sk_buff *skb);
void skb_reset_network_header(struct sk_buff *skb);
void skb_reset_transport_header(struct sk_buff *skb);
void skb_set_transport_header(struct sk_buff *skb, const int offset);
```

### Network Device Operations

**Device Lookup:**
```c
struct net_device *dev_get_by_name(struct net *net, const char *name);
struct net_device *dev_get_by_index(struct net *net, int ifindex);
void dev_put(struct net_device *dev); // Release reference
```

**Packet Transmission:**
```c
int dev_queue_xmit(struct sk_buff *skb);
int dev_direct_xmit(struct sk_buff *skb);
netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb, struct net_device *dev);
```

**Packet Reception:**
```c
int netif_rx(struct sk_buff *skb);
int netif_receive_skb(struct sk_buff *skb);
gro_result_t napi_gro_receive(struct napi_struct *napi, struct sk_buff *skb);
```

### Socket Operations

**Socket Management:**
```c
struct sock *sk_alloc(struct net *net, int family, gfp_t priority,
                      struct proto *prot, int kern);
void sk_free(struct sock *sk);
void sock_hold(struct sock *sk);    // Increment reference
void sock_put(struct sock *sk);     // Decrement reference
```

**Socket State:**
```c
void sock_set_flag(struct sock *sk, enum sock_flags flag);
void sock_reset_flag(struct sock *sk, enum sock_flags flag);
bool sock_flag(const struct sock *sk, enum sock_flags flag);
```

## System Calls Reference

### Core Socket System Calls

**Socket Creation and Binding:**
```c
/* Create socket endpoint */
int socket(int domain, int type, int protocol);
// domain: AF_INET, AF_INET6, AF_UNIX, AF_PACKET
// type: SOCK_STREAM, SOCK_DGRAM, SOCK_RAW
// protocol: 0 (default), IPPROTO_TCP, IPPROTO_UDP

/* Bind socket to address */
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

/* Listen for connections (TCP) */
int listen(int sockfd, int backlog);

/* Accept connection (TCP) */
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
int accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags);

/* Connect to remote address */
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

**Data Transfer:**
```c
/* Send data */
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);

/* Receive data */
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);

/* Advanced I/O */
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```

**Common Flags:**
```c
/* Send/Receive Flags */
MSG_DONTWAIT   // Non-blocking operation
MSG_PEEK       // Peek at incoming data without removing
MSG_TRUNC      // Return real packet length
MSG_WAITALL    // Wait for full request or error
MSG_OOB        // Process out-of-band data
MSG_NOSIGNAL   // Don't send SIGPIPE on errors
```

**Socket Configuration:**
```c
/* Get/set socket options */
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);

/* Get socket/peer names */
int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

**Common Socket Options:**
```c
/* SOL_SOCKET level options */
SO_REUSEADDR    // Allow address reuse
SO_BROADCAST    // Enable broadcast
SO_KEEPALIVE    // Keep connections alive
SO_LINGER       // Linger on close if data present
SO_RCVBUF       // Receive buffer size
SO_SNDBUF       // Send buffer size
SO_RCVTIMEO     // Receive timeout
SO_SNDTIMEO     // Send timeout
SO_BINDTODEVICE // Bind socket to specific device

/* TCP level options (IPPROTO_TCP) */
TCP_NODELAY     // Disable Nagle algorithm
TCP_MAXSEG      // Maximum segment size
TCP_KEEPIDLE    // Idle time before keepalive probes
TCP_KEEPINTVL   // Interval between keepalive probes
TCP_KEEPCNT     // Number of keepalive probes
TCP_USER_TIMEOUT // Total time for unacknowledged data
```

## Essential Debugging Commands

### Network Interface Analysis

**Interface Statistics:**
```bash
# Basic interface information
ip link show                        # Show all interfaces
ip -s link show eth0               # Show interface with statistics
cat /proc/net/dev                  # Kernel interface statistics
netstat -i                         # Interface statistics (legacy)

# Detailed hardware information
ethtool eth0                       # Basic interface info
ethtool -k eth0                    # Show offload features
ethtool -S eth0                    # Show detailed NIC statistics
ethtool -g eth0                    # Show ring buffer parameters
ethtool -l eth0                    # Show channel/queue information
ethtool -c eth0                    # Show interrupt coalescing settings
```

**Queue and CPU Analysis:**
```bash
# Multi-queue information
cat /sys/class/net/eth0/queues/rx-*/rps_cpus
cat /sys/class/net/eth0/queues/tx-*/xps_cpus

# Interrupt analysis
cat /proc/interrupts | grep eth0
cat /proc/softirqs | grep NET
grep eth0 /proc/interrupts | awk '{print $1}' | tr -d ':'
```

### Socket and Connection Analysis

**Socket Information:**
```bash
# Modern socket statistics
ss -tuln                           # TCP/UDP listening sockets
ss -tulpn                          # Include process information
ss -i                              # Show internal TCP information
ss -m                              # Show socket memory usage
ss -o state established            # Show established connections with timers
ss --info sport = :80              # Detailed info for specific port

# Legacy netstat
netstat -tuln                      # TCP/UDP listening sockets
netstat -tulpn                     # Include process names
netstat -s                         # Protocol statistics
netstat -r                         # Routing table
```

**Connection Tracking:**
```bash
# Connection states
ss -o state established            # Established connections
ss -o state syn-sent               # Outgoing connection attempts
ss -o state syn-recv               # Incoming connection attempts
ss -o state time-wait              # TIME_WAIT connections

# Process and socket mapping
lsof -i :80                        # Processes using port 80
fuser -n tcp 80                    # Find process using TCP port 80
```

### Protocol-Specific Analysis

**TCP Analysis:**
```bash
# TCP connection information
cat /proc/net/tcp                  # IPv4 TCP connections
cat /proc/net/tcp6                 # IPv6 TCP connections

# TCP configuration
cat /proc/sys/net/ipv4/tcp_congestion_control
cat /proc/sys/net/ipv4/tcp_available_congestion_control
cat /proc/sys/net/ipv4/tcp_rmem    # TCP read memory
cat /proc/sys/net/ipv4/tcp_wmem    # TCP write memory

# TCP statistics
netstat -s | grep -i tcp
ss --info --memory --processes sport = :22
```

**UDP Analysis:**
```bash
# UDP socket information
cat /proc/net/udp                  # IPv4 UDP sockets
cat /proc/net/udp6                 # IPv6 UDP sockets
netstat -s | grep -i udp           # UDP statistics

# UDP buffer configuration
cat /proc/sys/net/core/rmem_default # Default receive buffer
cat /proc/sys/net/core/rmem_max     # Maximum receive buffer
cat /proc/sys/net/core/wmem_default # Default send buffer
cat /proc/sys/net/core/wmem_max     # Maximum send buffer
```

**Routing and ARP:**
```bash
# Routing information
ip route show                      # Show routing table
ip route get <IP_ADDRESS>              # Show route to destination
cat /proc/net/route                # Kernel routing table (IPv4)
cat /proc/net/ipv6_route           # Kernel routing table (IPv6)

# ARP/Neighbor information
ip neigh show                      # ARP/neighbor table
cat /proc/net/arp                  # ARP table
arp -a                             # ARP table (legacy)
```

### Packet Capture and Analysis

**tcpdump Examples:**
```bash
# Basic packet capture
tcpdump -i eth0                    # Capture on interface
tcpdump -i any                     # Capture on all interfaces
tcpdump -nn -i eth0                # No name resolution

# Protocol filtering
tcpdump -i eth0 tcp                # TCP packets only
tcpdump -i eth0 udp port 53        # DNS traffic
tcpdump -i eth0 icmp               # ICMP packets

# Host and network filtering
tcpdump -i eth0 host 192.168.1.1   # Specific host
tcpdump -i eth0 net 192.168.0.0/24 # Network range
tcpdump -i eth0 src 192.168.1.1    # Source host

# Advanced filtering
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'    # SYN packets
tcpdump -i eth0 'tcp[tcpflags] & (tcp-fin) != 0'    # FIN packets
tcpdump -i eth0 'tcp port 80 and (tcp[tcpflags] & (tcp-syn) != 0)'

# Output options
tcpdump -w capture.pcap -i eth0    # Write to file
tcpdump -r capture.pcap            # Read from file
tcpdump -X -i eth0                 # Show packet contents
tcpdump -v -i eth0                 # Verbose output
```

### System Performance Analysis

**Network Memory Usage:**
```bash
# Socket memory
cat /proc/net/sockstat             # Socket statistics
cat /proc/net/sockstat6            # IPv6 socket statistics

# Memory pressure indicators
cat /proc/sys/net/core/netdev_max_backlog
cat /proc/sys/net/core/netdev_budget
cat /proc/sys/net/core/optmem_max

# Buffer usage
grep -E "tcp|udp|raw" /proc/net/protocols
```

**CPU and Interrupt Analysis:**
```bash
# Softirq monitoring
watch -n 1 'cat /proc/softirqs | grep NET'
cat /proc/stat | grep softirq

# IRQ affinity
cat /proc/irq/*/smp_affinity_list | grep -v "0-"
for irq in /proc/irq/*/; do echo "$irq: $(cat $irq/smp_affinity_list)"; done

# Network namespace statistics
ip netns list
ip netns exec <namespace> ss -tuln
```

## Performance Tuning Parameters

### Key sysctl Parameters

**Core Network Parameters:**
```bash
# Buffer sizes
net.core.rmem_max = 134217728              # Max receive buffer
net.core.wmem_max = 134217728              # Max send buffer
net.core.rmem_default = 65536              # Default receive buffer
net.core.wmem_default = 65536              # Default send buffer

# Queue management
net.core.netdev_max_backlog = 5000         # Max packets in queue
net.core.netdev_budget = 600               # NAPI budget per softirq
net.core.dev_weight = 64                   # NAPI weight

# Connection limits
net.core.somaxconn = 65535                 # Max listen queue size
```

**TCP Parameters:**
```bash
# TCP buffers
net.ipv4.tcp_rmem = 4096 87380 134217728   # TCP read memory
net.ipv4.tcp_wmem = 4096 65536 134217728   # TCP write memory
net.ipv4.tcp_mem = 262144 349525 524288    # TCP memory pressure thresholds

# TCP behavior
net.ipv4.tcp_window_scaling = 1            # Enable window scaling
net.ipv4.tcp_timestamps = 1               # Enable timestamps
net.ipv4.tcp_sack = 1                     # Enable SACK
net.ipv4.tcp_fack = 1                     # Enable FACK
net.ipv4.tcp_congestion_control = cubic    # Congestion control algorithm

# TCP connection management
net.ipv4.tcp_fin_timeout = 30             # FIN_WAIT_2 timeout
net.ipv4.tcp_tw_reuse = 1                 # Reuse TIME_WAIT sockets
net.ipv4.tcp_max_syn_backlog = 8192       # SYN backlog size
```

This comprehensive reference guide provides the essential information needed for Linux networking development, debugging, and optimization. Keep this guide handy for quick lookups during your networking adventures!

## Series Navigation

- **Previous**: [Tools & Scripts Guide - Practical Networking Utilities](/2024/03/22/linux-networking-tools-and-scripts-guide/)
- **Next**: [Code Analysis Walkthrough - Understanding Linux Networking Implementation](/2024/03/24/linux-networking-code-analysis-walkthrough/)
- **Series Index**: [Linux Networking Deep Dive Series](/linux-networking-series/)

---

*This reference guide is part of the comprehensive Linux Networking Deep Dive series, providing essential information for network developers and system administrators.*