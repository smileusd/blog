---
title: "Linux Networking Deep Dive: Part 2 - Network Layer & IP Processing"
date: 2024-03-18 10:00:00
categories:
  - systems-programming
  - networking
tags:
  - linux-networking
  - ip-processing
  - routing
  - netfilter
  - network-layer
series: "Linux Networking Deep Dive"
part: 2
description: "Deep dive into IP packet processing, routing decisions, and netfilter framework. Learn how Linux handles network layer protocols."
---

## Series Navigation

This is **Part 2** of the Linux Networking Deep Dive series:

- [Part 1: Foundation - Physical & Link Layer](../2024-03-16-linux-networking-deep-dive-introduction/) ← Previous
- **Part 2: Network Layer & IP Processing** ← You are here
- Part 3: Transport Layer & Socket Processing → Coming Next

## Introduction

In [Part 1](../2024-03-16-linux-networking-deep-dive-introduction/), we explored the physical and link layer foundations of Linux networking, including network device registration, sk_buff management, and the transition from link layer to network layer. Now we dive into the network layer, where Linux makes critical routing decisions and applies powerful packet filtering and manipulation capabilities.

The network layer is where packets begin their journey through the IP protocol stack, determining whether they should be delivered locally, forwarded to another host, or processed by specialized services. This layer implements the core Internet Protocol (IPv4/IPv6) processing, sophisticated routing algorithms, and the netfilter framework that powers iptables, NAT, and firewall functionality.

## IP Packet Processing Pipeline

### The Journey Begins: ip_rcv()

Every IPv4 packet's journey through the Linux network stack begins in the `ip_rcv()` function, located in `net/ipv4/ip_input.c`. This is where the link layer hands off packets to the network layer for processing.

```c
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt,
           struct net_device *orig_dev)
{
    struct net *net = dev_net(dev);

    skb = ip_rcv_core(skb, net);
    if (skb == NULL)
        return NET_RX_DROP;

    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
                   net, NULL, skb, dev, NULL,
                   ip_rcv_finish);
}
```

This function demonstrates several key design principles:

- **Network Namespace Awareness**: Uses `dev_net(dev)` to operate within the correct network namespace
- **Two-Stage Processing**: Core validation followed by netfilter hook processing
- **Early Exit Strategy**: Invalid packets are dropped before expensive netfilter processing

### Core IP Validation

The `ip_rcv_core()` function performs essential sanity checks that protect the system from malformed packets:

#### Promiscuous Mode Filtering
```c
if (skb->pkt_type == PACKET_OTHERHOST) {
    dev_core_stats_rx_otherhost_dropped_inc(skb->dev);
    drop_reason = SKB_DROP_REASON_OTHERHOST;
    goto drop;
}
```

This check ensures that packets captured in promiscuous mode but not destined for this host are immediately dropped, preventing unnecessary processing.

#### Header Length Validation
```c
if (!pskb_may_pull(skb, sizeof(struct iphdr)))
    goto inhdr_error;

iph = ip_hdr(skb);

if (iph->version != 4)
    goto inhdr_error;

if (iph->ihl < 5)  // Minimum header length (20 bytes)
    goto inhdr_error;
```

These checks ensure that the packet contains a valid IPv4 header with the minimum required 20 bytes.

#### Packet Length and Checksum Verification
```c
len = ntohs(iph->tot_len);
if (skb->len < len) {
    drop_reason = SKB_DROP_REASON_PKT_TOO_SMALL;
    __IP_INC_STATS(net, IPSTATS_MIB_INTRUNCATEDPKTS);
    goto drop;
}

if (ip_fast_csum((u8 *)iph, iph->ihl)) {
    drop_reason = SKB_DROP_REASON_IP_CSUM;
    __IP_INC_STATS(net, IPSTATS_MIB_INHDRERRORS);
    goto drop;
}
```

The kernel performs comprehensive validation including length consistency and header checksum verification using optimized assembly instructions.

### Drop Reasons and Statistics

Linux maintains detailed statistics about why packets are dropped, which is invaluable for troubleshooting:

| Drop Reason | SNMP Counter | Common Cause |
|-------------|--------------|--------------|
| `SKB_DROP_REASON_OTHERHOST` | - | Promiscuous mode filter |
| `SKB_DROP_REASON_PKT_TOO_SMALL` | `IPSTATS_MIB_INTRUNCATEDPKTS` | Packet smaller than header indicates |
| `SKB_DROP_REASON_IP_CSUM` | `IPSTATS_MIB_INHDRERRORS` | Invalid IP header checksum |
| `SKB_DROP_REASON_IP_INHDR` | `IPSTATS_MIB_INHDRERRORS` | Malformed IP header |

## Routing System Deep Dive

After successful validation, packets enter the routing subsystem through the `ip_rcv_finish()` function. This is where Linux determines the packet's ultimate destination.

### Route Lookup Optimization

Modern Linux implements several optimizations to minimize routing lookup overhead:

#### 1. Route Hint Optimization
```c
if (ip_can_use_hint(skb, iph, hint)) {
    err = ip_route_use_hint(skb, iph->daddr, iph->saddr, iph->tos,
                            dev, hint);
    if (unlikely(err))
        goto drop_error;
}
```

For packet streams, the kernel can reuse routing decisions from previous packets in the same flow, significantly reducing CPU overhead.

#### 2. Early Demux for Performance
```c
if (READ_ONCE(net->ipv4.sysctl_ip_early_demux) &&
    !skb_dst(skb) &&
    !skb->sk &&
    !ip_is_fragment(iph)) {
    // Protocol-specific early demux
}
```

Early demux allows established TCP connections to bypass full routing lookup by associating packets directly with their sockets.

### FIB Trie Structure

Linux uses a compressed trie (Patricia tree) for efficient route lookup, implemented in `net/ipv4/fib_trie.c`:

```c
struct key_vector {
    t_key key;                // Routing prefix
    unsigned char pos;        // Bit position
    unsigned char bits;       // Number of bits
    unsigned char slen;       // Suffix length
    union {
        struct hlist_head leaf;     // Terminal nodes (routes)
        struct key_vector __rcu *tnode[0]; // Internal nodes
    };
};
```

This structure provides several advantages:
- **O(log n) lookup time**: Efficient for large routing tables
- **Memory efficient**: Compressed representation reduces memory usage
- **Cache-friendly**: Locality of reference improves performance
- **Lock-free reads**: RCU protection allows concurrent access

### Route Lookup Process

The core lookup function `fib_lookup()` implements a layered approach:

```c
int fib_lookup(struct net *net, struct flowi4 *flp,
               struct fib_result *res, unsigned int flags)
{
    struct fib_table *tb;

    // Check local table first
    tb = fib_get_table(net, RT_TABLE_LOCAL);
    if (tb && !__fib_lookup(net, flp, res, flags))
        return 0;

    // Check main table
    tb = fib_get_table(net, RT_TABLE_MAIN);
    if (tb && !__fib_lookup(net, flp, res, flags))
        return 0;

    // Apply policy routing rules
    return fib_rules_lookup(net, flp, res);
}
```

This approach checks tables in priority order:
1. **Local table**: Routes to local interfaces and addresses
2. **Main table**: Normal routing entries
3. **Policy routing**: Rule-based routing for advanced configurations

### Policy Routing and Multiple Tables

Linux supports up to 255 routing tables, enabling sophisticated policy-based routing:

```bash
# Default routing rule priority
0:    from all lookup local     # Local routes
32766: from all lookup main      # Main routing table
32767: from all lookup default   # Default routes
```

Policy rules can make routing decisions based on:
- **Source address**: Route packets from specific networks differently
- **Type of Service**: Prioritize traffic based on TOS field
- **Input interface**: Apply different policies per interface
- **Packet marks**: Route based on netfilter marks

Example policy routing configuration:
```bash
# Route traffic from 192.168.1.0/24 via different gateway
ip rule add from 192.168.1.0/24 table 100
ip route add default via 10.0.1.1 table 100

# Route high-priority traffic differently
ip rule add tos 0x10 table 200
ip route add default via 10.0.2.1 table 200
```

## Netfilter Framework Architecture

The netfilter framework is the cornerstone of Linux packet filtering, providing hooks at strategic points in the packet processing pipeline.

### The Five Netfilter Hooks

Netfilter defines five hook points in the IPv4 stack, each serving specific purposes:

#### 1. NF_INET_PRE_ROUTING
**Location**: After packet arrival, before routing decision
**Use Cases**:
- **DNAT (Destination NAT)**: Redirect packets to different destinations
- **Connection Tracking**: Establish connection state
- **Early packet filtering**: Drop malicious packets before routing overhead

```c
return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
               net, NULL, skb, dev, NULL,
               ip_rcv_finish);
```

#### 2. NF_INET_LOCAL_IN
**Location**: After routing, for packets destined locally
**Use Cases**:
- **Local packet filtering**: Control access to local services
- **Service protection**: Implement rate limiting and access controls
- **Logging and auditing**: Monitor incoming connections

#### 3. NF_INET_FORWARD
**Location**: After routing, for packets to be forwarded
**Use Cases**:
- **Firewall rules**: Control packet forwarding between networks
- **Bandwidth management**: Shape traffic flowing through the system
- **VPN processing**: Handle tunneled traffic

#### 4. NF_INET_LOCAL_OUT
**Location**: For locally generated packets, before routing
**Use Cases**:
- **Outbound filtering**: Control local application traffic
- **SNAT (Source NAT)**: Modify source addresses for outbound packets
- **Traffic shaping**: Apply QoS to outbound traffic

#### 5. NF_INET_POST_ROUTING
**Location**: After routing, just before packet transmission
**Use Cases**:
- **Final NAT**: Last chance for address translation
- **Traffic accounting**: Count and classify outbound traffic
- **QoS marking**: Set DSCP/TOS values for traffic prioritization

### Hook Processing and Priorities

Hooks are processed in priority order, with well-defined standard priorities:

```c
#define NF_IP_PRI_RAW_BEFORE_DEFRAG -450
#define NF_IP_PRI_CONNTRACK_DEFRAG  -400
#define NF_IP_PRI_RAW               -300
#define NF_IP_PRI_CONNTRACK         -200
#define NF_IP_PRI_MANGLE            -150
#define NF_IP_PRI_NAT_DST           -100
#define NF_IP_PRI_FILTER              0
#define NF_IP_PRI_NAT_SRC            100
```

This ordering ensures that:
- **Connection tracking** establishes state before filtering
- **Destination NAT** occurs before routing decisions
- **Source NAT** happens after routing is complete
- **Raw table** can bypass connection tracking when needed

### Connection Tracking System

Connection tracking (conntrack) maintains state information for network connections, enabling stateful packet filtering:

```c
struct nf_conn {
    struct nf_conntrack         ct_general;
    spinlock_t                  lock;
    u16                         cpu;
    struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];
    unsigned long               status;
    u32                         timeout;
    // ... additional fields
};
```

#### Connection States

Different protocols maintain different state information:

**TCP States**: `SYN_SENT`, `SYN_RECV`, `ESTABLISHED`, `FIN_WAIT`, `CLOSE_WAIT`, `TIME_WAIT`, etc.

**UDP States**: `NEW`, `ESTABLISHED`, `UNREPLIED`

**ICMP States**: `NEW`, `ESTABLISHED` (for request/reply pairs)

Connection tracking enables powerful filtering rules:
```bash
# Allow established connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Drop invalid packets
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```

### NAT Implementation

Network Address Translation integrates deeply with connection tracking:

#### Source NAT (SNAT)
```bash
# Basic SNAT for private network
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -j SNAT --to-source 203.0.113.1

# MASQUERADE for dynamic IP addresses
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

#### Destination NAT (DNAT)
```bash
# Port forwarding to internal server
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:8080

# Load balancing across multiple servers
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100-192.168.1.103
```

## ICMP Processing and Network Diagnostics

The Internet Control Message Protocol (ICMP) provides essential diagnostic and error reporting capabilities.

### ICMP Message Processing

ICMP processing begins in `icmp_rcv()`, which validates and dispatches messages:

```c
int icmp_rcv(struct sk_buff *skb)
{
    struct icmphdr *icmph;

    // Security policy check
    if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb))
        goto drop;

    // Header validation
    if (!pskb_may_pull(skb, sizeof(*icmph)))
        goto drop;

    icmph = icmp_hdr(skb);

    // Checksum verification
    if (skb_checksum_simple_validate(skb))
        goto drop;

    // Dispatch to appropriate handler
    return icmp_pointers[icmph->type].handler(skb);
}
```

### Message Type Handlers

Different ICMP message types have specialized handlers:

| Type | Handler | Purpose |
|------|---------|---------|
| 0 (Echo Reply) | `ping_rcv` | Ping responses |
| 3 (Dest Unreachable) | `icmp_unreach` | Network/host/port unreachable |
| 5 (Redirect) | `icmp_redirect` | Route optimization |
| 8 (Echo Request) | `icmp_echo` | Ping requests |
| 11 (Time Exceeded) | `icmp_unreach` | TTL expiration |

### Path MTU Discovery

ICMP Type 3, Code 4 (Fragmentation Needed) implements Path MTU Discovery:

```c
static void icmp_unreach(struct sk_buff *skb)
{
    struct icmphdr *icmph = icmp_hdr(skb);

    switch (icmph->code) {
    case ICMP_FRAG_NEEDED:
        info = ntohs(icmph->un.frag.mtu);
        // Update route cache with new MTU
        dst_update_pmtu(skb_dst(skb), NULL, skb, info, true);
        break;
    }

    // Deliver error to socket layer
    icmp_socket_deliver(skb, info);
}
```

This mechanism allows applications to discover the optimal packet size for a network path, improving efficiency and reducing fragmentation.

### Rate Limiting and Security

ICMP implements sophisticated rate limiting to prevent abuse:

```c
static bool icmp_global_allow(struct net *net, int *credit)
{
    int delta = jiffies - net->ipv4.icmp_global_credit_last_update;

    if (delta > 0) {
        int new_credit = delta * net->ipv4.icmp_global_credit_rate;
        net->ipv4.icmp_global_credit = min_t(int,
            net->ipv4.icmp_global_credit + new_credit,
            net->ipv4.icmp_global_credit_cap);
    }

    if (net->ipv4.icmp_global_credit > 0) {
        net->ipv4.icmp_global_credit--;
        return true;
    }

    return false;
}
```

Configuration options for ICMP security:
```bash
# Configure ICMP rate limiting
echo 'net.ipv4.icmp_ratelimit = 1000' >> /etc/sysctl.conf

# Disable ping responses (security hardening)
echo 'net.ipv4.icmp_echo_ignore_all = 1' >> /etc/sysctl.conf

# Disable ICMP redirects (prevent attacks)
echo 'net.ipv4.conf.all.accept_redirects = 0' >> /etc/sysctl.conf
```

## Practical Examples and Exercises

### Network Layer Debugging Tools

#### 1. Route Analysis
```bash
# Show routing table
ip route show

# Test route lookup for specific destination
ip route get 8.8.8.8

# Show policy routing rules
ip rule show

# Monitor routing decisions
ip monitor route
```

#### 2. Packet Processing Analysis
```bash
# View IP processing statistics
cat /proc/net/snmp | grep "Ip:"

# Monitor packet drops
watch 'cat /proc/net/snmp | grep "Ip:" | grep -E "(InDiscards|InErrors)"'

# Check netfilter statistics
cat /proc/net/netfilter/nf_log
```

#### 3. Connection Tracking Monitoring
```bash
# View active connections
cat /proc/net/nf_conntrack

# Monitor connection events
conntrack -E

# Show connection tracking statistics
cat /proc/net/stat/nf_conntrack
```

### Performance Tuning Examples

#### 1. Routing Performance
```bash
# Enable early demux for better performance
echo 'net.ipv4.ip_early_demux = 1' >> /etc/sysctl.conf

# Tune FIB hash table size
echo 'net.ipv4.fib_multipath_hash_policy = 1' >> /etc/sysctl.conf

# Optimize route cache
echo 'net.ipv4.route.gc_thresh = 32768' >> /etc/sysctl.conf
```

#### 2. Netfilter Optimization
```bash
# Increase connection tracking table size
echo 'net.netfilter.nf_conntrack_max = 262144' >> /etc/sysctl.conf

# Optimize hash table buckets
echo 'net.netfilter.nf_conntrack_buckets = 65536' >> /etc/sysctl.conf

# Tune connection timeouts
echo 'net.netfilter.nf_conntrack_tcp_timeout_established = 432000' >> /etc/sysctl.conf
```

### Troubleshooting Network Layer Issues

#### Common Problems and Solutions

**1. Packet Drops at IP Layer**
```bash
# Check for header errors
netstat -s | grep -i "ip.*error"

# Monitor specific drop reasons
echo 1 > /sys/kernel/debug/tracing/events/skb/kfree_skb/enable
cat /sys/kernel/debug/tracing/trace
```

**2. Routing Issues**
```bash
# Verify route lookup
traceroute -n destination_ip

# Check routing table consistency
ip route show table all | grep destination_network

# Test policy routing
ip route get destination_ip from source_ip
```

**3. Netfilter Problems**
```bash
# Enable netfilter packet tracing
iptables -t raw -A PREROUTING -j TRACE

# Monitor dropped packets
iptables -L -n -v | grep DROP

# Check connection tracking issues
dmesg | grep conntrack
```

## What's Next: Transport Layer Preview

In Part 3, we'll explore the transport layer where Linux implements TCP and UDP protocols. We'll cover:

- **TCP State Management**: Connection establishment, data transfer, and termination
- **Socket Layer Architecture**: How applications interact with the network stack
- **Buffer Management**: Socket buffers, congestion control, and flow control
- **Performance Optimization**: TCP optimizations, zero-copy techniques, and offloading
- **UDP Processing**: Connectionless protocol handling and multicast support

The transport layer builds upon the network layer foundation we've covered here, using the routing decisions and connection tracking state to efficiently deliver data to applications.

## Key Takeaways

The network layer represents a sophisticated balance of performance, security, and flexibility:

- **IP Processing Pipeline**: Efficient validation and processing with detailed error tracking
- **Routing System**: Scalable lookup algorithms with policy-based routing capabilities
- **Netfilter Framework**: Powerful packet filtering and manipulation with minimal overhead
- **ICMP Integration**: Essential diagnostics and error reporting with security protections

Understanding these subsystems is crucial for:
- **Network Performance**: Optimizing packet processing pipelines
- **Security Implementation**: Effective firewall and NAT configurations
- **Troubleshooting**: Diagnosing connectivity and routing issues
- **System Design**: Making informed networking architecture decisions

The Linux network layer's careful design enables it to scale from embedded devices to high-performance routers and servers, while maintaining the flexibility needed for modern networking requirements including containers, virtualization, and cloud environments.

---

*This post is part of the Linux Networking Deep Dive series. Stay tuned for Part 3 where we'll explore the transport layer and socket processing!*