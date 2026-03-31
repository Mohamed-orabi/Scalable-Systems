# Linux Sockets — The Complete Kernel Deep Dive

## From "What Is a Socket?" to Kernel Data Structures — For Beginners

---

# Part A: What Sockets Are

## Chapter 1: The Problem Sockets Solve

Two programs need to talk to each other. Maybe they're on the same
machine. Maybe they're on different machines across the internet.
They need a way to send bytes back and forth.

```
Program A (your web browser)     Program B (web server)
     in Cairo                          in London

     "Give me                    "Here is the
      the web page"              web page HTML"
          │                           │
          └─────────── ??? ───────────┘
          
          How do bytes travel between them?
```

The answer is **sockets**. A socket is an ENDPOINT for communication.
It's like a phone — each person has one, and they can talk through it.

```
Program A                              Program B
  socket A ◄──────── network ────────► socket B

  A writes bytes into socket A → bytes travel to socket B → B reads them
  B writes bytes into socket B → bytes travel to socket A → A reads them
```

**The genius of Unix sockets:** They look EXACTLY like files. You read()
from a socket the same way you read() from a file. You write() to a
socket the same way you write() to a file. A socket IS a file descriptor.

```rust
// Writing to a file:
let file = File::open("data.txt")?;
file.write_all(b"Hello")?;

// Writing to a socket (looks almost identical!):
let socket = TcpStream::connect("93.184.216.34:80")?;
socket.write_all(b"Hello")?;

// Both use fd, both use write(), both use read().
// The kernel handles the difference internally.
```


## Chapter 2: The Analogy — Phones

Think of sockets as phones:

```
YOUR PHONE                                  THEIR PHONE
(your socket)                               (their socket)

┌──────────────┐                            ┌──────────────┐
│ Phone number │   ← address (IP:port)      │ Phone number │
│ Speaker      │   ← write() sends bytes    │ Speaker      │
│ Microphone   │   ← read() receives bytes  │ Microphone   │
│ Dial button  │   ← connect()              │              │
│ Hang up      │   ← close()                │ Hang up      │
└──────────────┘                            └──────────────┘

To make a call:
  1. Create a phone          → socket()
  2. Dial a number           → connect(IP, port)
  3. Talk into the phone     → write(data)
  4. Listen to the phone     → read(data)
  5. Hang up                 → close()

To receive a call:
  1. Create a phone          → socket()
  2. Assign a phone number   → bind(IP, port)
  3. Turn on the ringer      → listen()
  4. Pick up when it rings   → accept() → returns a NEW socket for this call
  5. Talk / listen           → write() / read()
  6. Hang up                 → close()
```


## Chapter 3: Socket Types

There are three main types:

```
TYPE 1: SOCK_STREAM (TCP) — like a phone call
  → Reliable: every byte arrives, in order, no duplicates
  → Connected: you establish a connection BEFORE sending
  → Stream: bytes flow continuously (no message boundaries)
  → Used by: HTTP, SSH, databases, your storage engine's API

TYPE 2: SOCK_DGRAM (UDP) — like sending letters
  → Unreliable: letters might get lost, arrive out of order
  → Connectionless: just put an address on the letter and send
  → Datagram: each send() is a discrete message
  → Used by: DNS, video streaming, games

TYPE 3: SOCK_RAW — like having your own postal truck
  → You build the entire packet yourself (headers and all)
  → Used by: ping, traceroute, network tools
  → Requires root privileges
```

We'll focus on TCP (SOCK_STREAM) because it's what 90% of applications
use, including your storage engine if you ever add a network API.


---

# Part B: The Socket API — What Your Code Does

## Chapter 4: Server Side — Accept Connections

```rust
// A TCP server in Rust (what you'd write)

use std::net::TcpListener;
use std::io::{Read, Write};

fn main() {
    // Step 1: socket() + bind() + listen()
    // TcpListener::bind does all three in one call
    let listener = TcpListener::bind("0.0.0.0:8080").unwrap();
    //  "0.0.0.0" = listen on all network interfaces
    //  "8080"    = port number (like a phone extension)
    
    println!("Server listening on port 8080...");
    
    // Step 2: accept() — wait for a client to connect
    // This BLOCKS until someone connects
    for stream in listener.incoming() {
        let mut stream = stream.unwrap();
        
        // 'stream' is a NEW socket, dedicated to THIS client.
        // The listener continues accepting other clients.
        
        // Step 3: read() — receive data from the client
        let mut buffer = [0u8; 1024];
        let bytes_read = stream.read(&mut buffer).unwrap();
        println!("Received: {}", String::from_utf8_lossy(&buffer[..bytes_read]));
        
        // Step 4: write() — send data back to the client
        stream.write_all(b"Hello from server!").unwrap();
        
        // Step 5: close() — happens automatically when 'stream' is dropped
    }
}
```

## Chapter 5: Client Side — Connect to a Server

```rust
// A TCP client in Rust

use std::net::TcpStream;
use std::io::{Read, Write};

fn main() {
    // Step 1: socket() + connect()
    // TcpStream::connect does both
    let mut stream = TcpStream::connect("127.0.0.1:8080").unwrap();
    //  "127.0.0.1" = localhost (same machine)
    //  "8080"      = server's port number
    
    // Step 2: write() — send data to the server
    stream.write_all(b"Hello from client!").unwrap();
    
    // Step 3: read() — receive response from the server
    let mut buffer = [0u8; 1024];
    let bytes_read = stream.read(&mut buffer).unwrap();
    println!("Response: {}", String::from_utf8_lossy(&buffer[..bytes_read]));
    
    // Step 4: close() — automatic when 'stream' is dropped
}
```

## Chapter 6: The System Calls Behind Rust's API

Rust's std hides the raw syscalls. Here's what's really happening:

```
RUST CODE                          ACTUAL SYSCALLS
──────────                         ─────────────────

TcpListener::bind("0.0.0.0:8080")
  │
  ├── socket(AF_INET, SOCK_STREAM, 0)     → fd = 3
  │   "Create a TCP socket. Give me an fd."
  │
  ├── setsockopt(3, SO_REUSEADDR, 1)
  │   "Allow reusing the address if the server restarts."
  │
  ├── bind(3, {addr=0.0.0.0, port=8080})
  │   "Assign this address and port to my socket."
  │
  └── listen(3, 128)
      "Start accepting connections. Queue up to 128 pending."


listener.accept()
  │
  └── accept(3, &client_addr)              → fd = 4
      "Wait for a connection. Return a NEW fd for this client."
      "Also tell me the client's IP address."


stream.write_all(b"Hello")
  │
  └── write(4, "Hello", 5)                 → 5
      OR
      sendto(4, "Hello", 5, 0, NULL, 0)   → 5
      "Send these bytes through socket fd 4."


stream.read(&mut buf)
  │
  └── read(4, buf, 1024)                   → bytes_read
      OR
      recvfrom(4, buf, 1024, 0, NULL, NULL) → bytes_read
      "Receive bytes from socket fd 4."


drop(stream)
  │
  └── close(4)
      "Shut down and release socket fd 4."
```


---

# Part C: The Kernel's View — Socket as a File

## Chapter 7: A Socket IS a File Descriptor

This is the most important design decision in Unix networking:
**a socket is a file descriptor.** It lives in the same fd table as
regular files, pipes, and everything else.

```
Process's file descriptor table:

  fd 0 → struct file → stdin (terminal)
  fd 1 → struct file → stdout (terminal)
  fd 2 → struct file → stderr (terminal)
  fd 3 → struct file → TcpListener (listening socket)
  fd 4 → struct file → TcpStream (connected socket to client A)
  fd 5 → struct file → TcpStream (connected socket to client B)
  fd 6 → struct file → database.db (regular file)
  fd 7 → struct file → database.wal (regular file)
```

**They all live in the SAME table.** The kernel uses the same `struct file`
for sockets as it does for regular files. The difference is in the
OPERATIONS TABLE:

```
fd 6 (regular file):
  struct file {
      f_op = ext4_file_operations     ← filesystem ops
      f_inode = inode 4521             ← real file on disk
      f_pos = 8192                     ← has a position/cursor
  }

fd 4 (TCP socket):
  struct file {
      f_op = socket_file_ops           ← socket ops (different!)
      private_data = struct socket *   ← points to socket internals
      f_pos = 0                        ← meaningless for sockets
  }
```

When you call `write(4, data, 5)`:
- The VFS does the same 5 checks as always (FMODE_WRITE? etc.)
- Then calls `f_op->write_iter()`
- For a regular file: this goes to ext4
- For a socket: this goes to `sock_write_iter()` → TCP stack

**The VFS doesn't know or care that fd 4 is a socket.** It dispatches
through the operations table, and the socket's write function handles
the networking. This is the Strategy Pattern again — same interface,
different behavior.


## Chapter 8: The Kernel Socket Structures

A socket in the kernel is THREE nested structures:

```
struct file (the VFS layer — same as any fd)
  │
  │ private_data
  ▼
struct socket (the socket layer — generic for all socket types)
  │
  │ sk
  ▼
struct sock (the protocol layer — specific to TCP, UDP, etc.)
```

Let's look at each one:

### Layer 1: struct file (VFS)

```c
// The same struct file used for regular files.
// For sockets, it has:

struct file {
    struct file_operations *f_op;     // → socket_file_ops
    void *private_data;                // → struct socket
    // f_pos, f_inode, etc. are mostly unused for sockets
};

// socket_file_ops:
static const struct file_operations socket_file_ops = {
    .read_iter   = sock_read_iter,     // redirects to socket's recvmsg
    .write_iter  = sock_write_iter,    // redirects to socket's sendmsg
    .poll        = sock_poll,          // for select()/poll()/epoll
    .release     = sock_close,         // cleanup on close()
    .mmap        = sock_mmap,          // rarely used
    .fasync      = sock_fasync,        // async notification
};
```

### Layer 2: struct socket (Socket layer)

```c
struct socket {
    socket_state         state;    // SS_UNCONNECTED, SS_CONNECTING,
                                   // SS_CONNECTED, SS_DISCONNECTING
    
    unsigned long        flags;    // SOCK_ASYNC_NOSPACE, etc.
    
    struct file          *file;    // back-pointer to the struct file
    
    struct sock          *sk;      // → the protocol-specific structure
    
    const struct proto_ops *ops;   // → protocol operations table
    //
    // For TCP: ops = &inet_stream_ops
    //   .bind      = inet_bind
    //   .connect   = inet_stream_connect
    //   .accept    = inet_accept
    //   .listen    = inet_listen
    //   .sendmsg   = tcp_sendmsg
    //   .recvmsg   = tcp_recvmsg
    //   .shutdown  = inet_shutdown
};
```

### Layer 3: struct sock (Protocol layer — TCP-specific)

```c
struct sock {
    // ─── Connection identity (the "4-tuple") ───
    __be32           sk_saddr;      // source IP (my address)
    __be32           sk_daddr;      // destination IP (their address)  
    __be16           sk_sport;      // source port (my port)
    __be16           sk_dport;      // destination port (their port)
    
    // ─── State ───
    unsigned char    sk_state;      // TCP_ESTABLISHED, TCP_LISTEN, etc.
    
    // ─── Buffers ───
    struct sk_buff_head sk_receive_queue;  // ★ incoming data waits here
    struct sk_buff_head sk_write_queue;    // ★ outgoing data waits here
    
    // ─── Timers ───
    struct timer_list  sk_timer;    // retransmission timer
    
    // ─── Memory limits ───
    long             sk_rcvbuf;     // max receive buffer size
    long             sk_sndbuf;     // max send buffer size
    
    // ─── Wait queue ───
    struct socket_wq *sk_wq;       // processes sleeping on this socket
                                   // (blocked in read() waiting for data)
    
    // ─── TCP-specific (in struct tcp_sock, which extends struct sock) ───
    u32              snd_nxt;       // next sequence number to send
    u32              rcv_nxt;       // next sequence number expected
    u32              snd_wnd;       // how much the receiver can accept
    u32              rcv_wnd;       // how much WE can accept
    u32              srtt_us;       // smoothed round-trip time
    // ... many more TCP state fields
};
```


## Chapter 9: The Three Layers Visualized

```
YOUR CODE: write(fd=4, "Hello", 5)

  fd 4
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│ struct file                                                  │
│   f_op = socket_file_ops                                    │
│   private_data ──────────────────────────────────────┐      │
│                                                      │      │
│   VFS calls: f_op->write_iter()                      │      │
│   → sock_write_iter()                                │      │
│     "I'm not a file. I'm a socket. Let me redirect." │      │
└──────────────────────────────────────────────────────│──────┘
                                                       │
                                                       ▼
┌─────────────────────────────────────────────────────────────┐
│ struct socket                                                │
│   state = SS_CONNECTED                                      │
│   ops = inet_stream_ops (TCP operations)                    │
│   sk ──────────────────────────────────────────────┐        │
│                                                    │        │
│   socket layer calls: ops->sendmsg()               │        │
│   → tcp_sendmsg()                                  │        │
│     "I know this is TCP. Let me handle the protocol."│       │
└────────────────────────────────────────────────────│────────┘
                                                     │
                                                     ▼
┌─────────────────────────────────────────────────────────────┐
│ struct sock (struct tcp_sock)                                │
│   sk_saddr = 192.168.1.10 (my IP)                           │
│   sk_daddr = 93.184.216.34 (their IP)                       │
│   sk_sport = 54321 (my port)                                │
│   sk_dport = 80 (their port)                                │
│   sk_state = TCP_ESTABLISHED                                │
│                                                             │
│   sk_write_queue: [skb with "Hello"]                        │
│   → TCP adds headers, computes checksum                     │
│   → passes to IP layer → to device driver → to NIC → wire  │
└─────────────────────────────────────────────────────────────┘
```


---

# Part D: How Data Flows Through the Socket

## Chapter 10: The Send Path — write() on a Socket

```
Your code: stream.write_all(b"Hello")?;

LAYER 1: VFS (same as any write)
  ┌────────────────────────────────────────────────────────┐
  │ sys_write(fd=4, buf="Hello", count=5)                  │
  │   → fdget_pos(4) → struct file                         │
  │   → vfs_write() → 5 checks (same as file writes!)     │
  │   → f_op->write_iter() → sock_write_iter()            │
  └────────────────────────────────────────────────────────┘
          │
          ▼
LAYER 2: Socket layer
  ┌────────────────────────────────────────────────────────┐
  │ sock_write_iter():                                     │
  │   → builds a struct msghdr (message header)            │
  │   → calls socket->ops->sendmsg()                      │
  │   → inet_sendmsg()                                    │
  │   → tcp_sendmsg()                                     │
  └────────────────────────────────────────────────────────┘
          │
          ▼
LAYER 3: TCP (transport layer)
  ┌────────────────────────────────────────────────────────┐
  │ tcp_sendmsg():                                         │
  │                                                        │
  │   Step 1: Copy your data into kernel buffers (sk_buff) │
  │     skb = alloc_skb(size)                              │
  │     copy_from_user(skb->data, "Hello", 5)              │
  │     → Your "Hello" is now in a kernel buffer.          │
  │                                                        │
  │   Step 2: Add to the send queue                        │
  │     sk->sk_write_queue ← add skb                       │
  │     → Queued for sending.                              │
  │                                                        │
  │   Step 3: Build TCP header                             │
  │     ┌──────────────────────────────────────┐           │
  │     │ TCP Header (20 bytes):                │           │
  │     │   Source port: 54321                  │           │
  │     │   Dest port: 80                       │           │
  │     │   Sequence number: 1001               │           │
  │     │   Ack number: 5001                    │           │
  │     │   Flags: PSH|ACK                      │           │
  │     │   Window size: 65535                  │           │
  │     │   Checksum: 0xA3B4                    │           │
  │     └──────────────────────────────────────┘           │
  │                                                        │
  │   Step 4: Pass to IP layer                             │
  │     ip_queue_xmit(skb)                                 │
  └────────────────────────────────────────────────────────┘
          │
          ▼
LAYER 4: IP (network layer)
  ┌────────────────────────────────────────────────────────┐
  │ ip_queue_xmit():                                       │
  │                                                        │
  │   Step 1: Route lookup                                 │
  │     "Where do I send packets for 93.184.216.34?"       │
  │     → routing table → gateway 192.168.1.1, via eth0    │
  │                                                        │
  │   Step 2: Build IP header                              │
  │     ┌──────────────────────────────────────┐           │
  │     │ IP Header (20 bytes):                 │           │
  │     │   Source IP: 192.168.1.10             │           │
  │     │   Dest IP: 93.184.216.34              │           │
  │     │   Protocol: TCP (6)                   │           │
  │     │   TTL: 64                             │           │
  │     │   Total length: 45                    │           │
  │     └──────────────────────────────────────┘           │
  │                                                        │
  │   Step 3: Pass to link layer                           │
  │     dev_queue_xmit(skb)                                │
  └────────────────────────────────────────────────────────┘
          │
          ▼
LAYER 5: Network device driver
  ┌────────────────────────────────────────────────────────┐
  │ dev_queue_xmit():                                      │
  │                                                        │
  │   Step 1: Add Ethernet header                          │
  │     ┌──────────────────────────────────────┐           │
  │     │ Ethernet Header (14 bytes):           │           │
  │     │   Dest MAC: AA:BB:CC:DD:EE:FF        │           │
  │     │   Source MAC: 11:22:33:44:55:66      │           │
  │     │   Type: 0x0800 (IPv4)                │           │
  │     └──────────────────────────────────────┘           │
  │                                                        │
  │   Step 2: Queue the packet                             │
  │     Put the skb in the NIC's transmit ring buffer.     │
  │                                                        │
  │   Step 3: Tell the NIC to send                         │
  │     Write to NIC's doorbell register.                  │
  │     NIC DMAs the packet from RAM to the wire.          │
  └────────────────────────────────────────────────────────┘
          │
          ▼
     PHYSICAL WIRE / WiFi / fiber
     Your "Hello" is now electromagnetic signals
     traveling to the destination machine.


THE COMPLETE PACKET ON THE WIRE:

  ┌────────────┬────────────┬────────────┬─────────┐
  │ Ethernet   │ IP Header  │ TCP Header │ "Hello" │
  │ Header     │ (20 bytes) │ (20 bytes) │ (5 bytes)│
  │ (14 bytes) │            │            │         │
  └────────────┴────────────┴────────────┴─────────┘
  
  Total: 59 bytes on the wire for your 5-byte "Hello"!
  54 bytes of headers + 5 bytes of data.
```


## Chapter 11: The Receive Path — read() on a Socket

```
The OTHER machine sent us "World". How does it arrive?

LAYER 5: NIC hardware receives the packet
  ┌────────────────────────────────────────────────────────┐
  │ NIC sees electromagnetic signals on the wire.          │
  │ Reassembles into a frame (ethernet header + payload).  │
  │ DMAs the frame into a pre-allocated kernel buffer.     │
  │ Triggers an interrupt (or NAPI poll).                  │
  └────────────────────────────────────────────────────────┘
          │
          ▼
LAYER 4: IP layer processes the packet
  ┌────────────────────────────────────────────────────────┐
  │ ip_rcv():                                              │
  │   Strip the IP header.                                 │
  │   Verify checksum.                                     │
  │   Check: is this for us? (dest IP = our IP? yes.)      │
  │   Check: protocol field = 6 (TCP).                     │
  │   → Pass to TCP layer.                                 │
  └────────────────────────────────────────────────────────┘
          │
          ▼
LAYER 3: TCP layer processes the segment
  ┌────────────────────────────────────────────────────────┐
  │ tcp_v4_rcv():                                          │
  │                                                        │
  │   Step 1: Find the socket                              │
  │     Look up: (src_ip, src_port, dst_ip, dst_port)      │
  │     → hash table → found struct sock!                  │
  │                                                        │
  │   Step 2: Validate TCP header                          │
  │     Check sequence number, checksum, flags.            │
  │                                                        │
  │   Step 3: Process the data                             │
  │     Strip TCP header.                                  │
  │     Put the data payload ("World") into an skb.        │
  │     Add skb to sk->sk_receive_queue.                   │
  │                                                        │
  │   Step 4: Wake up anyone waiting                       │
  │     If a process is blocked in read() on this socket,  │
  │     wake it up: "Hey, data arrived!"                   │
  │     sk->sk_wq → wake_up_interruptible()               │
  │                                                        │
  │   Step 5: Send ACK back                                │
  │     "I received your bytes. My ack number is now       │
  │      their_seq + 5."                                   │
  └────────────────────────────────────────────────────────┘
          │
          ▼
YOUR CODE: stream.read(&mut buf)
  ┌────────────────────────────────────────────────────────┐
  │ sys_read(fd=4, buf, 1024)                              │
  │   → VFS → sock_read_iter() → tcp_recvmsg()            │
  │                                                        │
  │ tcp_recvmsg():                                         │
  │   Check sk->sk_receive_queue:                          │
  │     → Data is waiting! (the skb with "World")          │
  │   Copy from kernel buffer to user buffer:              │
  │     copy_to_user(buf, skb->data, 5)                    │
  │   Return 5 (bytes read).                               │
  │                                                        │
  │ If the queue was EMPTY:                                │
  │   The process SLEEPS (blocks) until data arrives.      │
  │   When the NIC interrupt handler adds data to the      │
  │   queue and calls wake_up(), the process resumes.      │
  └────────────────────────────────────────────────────────┘
```


---

# Part E: The sk_buff — The Kernel's Packet Container

## Chapter 12: What an sk_buff Is

Every packet in the kernel is wrapped in a `struct sk_buff` (socket buffer).
It's the MOST important data structure in Linux networking:

```c
struct sk_buff {
    // ─── Navigation ───
    struct sk_buff   *next, *prev;    // linked list (queue)
    struct sock      *sk;             // which socket owns this
    struct net_device *dev;           // which network interface
    
    // ─── Data pointers ───
    unsigned char    *head;       // start of allocated memory
    unsigned char    *data;       // start of current data ★
    unsigned char    *tail;       // end of current data
    unsigned char    *end;        // end of allocated memory
    
    unsigned int     len;         // total data length
    
    // ─── Protocol headers ───
    __u16            protocol;    // ETH_P_IP, ETH_P_ARP, etc.
    union {
        struct tcphdr  *th;       // TCP header pointer
        struct udphdr  *uh;       // UDP header pointer
        struct iphdr   *iph;      // IP header pointer
    } h;
};
```

**The clever part:** As the packet moves through layers, the `data` pointer
is adjusted to point to the current layer's header:

```
When building a packet (sending):

  Start: data points to payload
    head ──────────────────────────────── end
              data ──── tail
              │ "Hello" │

  After TCP adds header:
    head ──────────────────────────────── end
         data ──────────── tail
         │TCP│  "Hello"   │

  After IP adds header:
    head ──────────────────────────────── end
    data ──────────────────── tail
    │IP │TCP│   "Hello"    │

  After Ethernet adds header:
    head ──────────────────────────────── end
    data ──────────────────────── tail
    │ETH│IP │TCP│  "Hello"  │

Each layer PREPENDS its header by moving 'data' backward.
The actual data ("Hello") never moves — only the pointer changes!
This is called "headroom" and it's very efficient (no copying).


When receiving a packet (the reverse):

  Start: data points to ethernet header
    │ETH│IP │TCP│  "World"  │

  Ethernet layer strips its header (moves data forward):
    │IP │TCP│  "World"  │

  IP layer strips its header:
    │TCP│  "World"  │

  TCP layer strips its header:
    │  "World"  │    ← just the payload, ready for your read()
```


---

# Part F: Connection Establishment — The TCP Handshake

## Chapter 13: What Happens During connect() and accept()

When a client calls `connect()` and a server calls `accept()`, the
TCP three-way handshake happens:

```
CLIENT                                              SERVER
socket()                                            socket()
connect(server_ip:80) ──┐                           bind(0.0.0.0:80)
                        │                           listen(128)
                        │                           accept() ← BLOCKS
                        │
  ┌─────────────────────┴────────────────────────────────────────┐
  │                                                              │
  │  Step 1: CLIENT sends SYN                                   │
  │    Client kernel builds a TCP packet:                        │
  │      SYN flag = 1                                           │
  │      Sequence number = 1000 (random starting number)        │
  │      "I want to connect. My starting seq is 1000."          │
  │                                          ──────────►        │
  │                                                              │
  │  Step 2: SERVER sends SYN-ACK                               │
  │    Server kernel builds a TCP packet:                        │
  │      SYN flag = 1, ACK flag = 1                             │
  │      Sequence number = 5000 (server's random start)         │
  │      Ack number = 1001 (client's seq + 1)                   │
  │      "I accept. My starting seq is 5000. I expect your      │
  │       next byte to be 1001."                                │
  │    ◄──────────                                              │
  │                                                              │
  │  Step 3: CLIENT sends ACK                                   │
  │      ACK flag = 1                                           │
  │      Ack number = 5001 (server's seq + 1)                   │
  │      "Got it. I expect your next byte to be 5001."          │
  │                                          ──────────►        │
  │                                                              │
  └──────────────────────────────────────────────────────────────┘
                        │
connect() returns       │                           accept() returns
  "Connected!"          │                             new_fd for this client
                        │
                    ESTABLISHED
              Both sides can now send data.
```

### What happens in the kernel during this:

```
CLIENT calls connect():
  1. Kernel creates a SYN packet
  2. Sends it via IP → device driver → wire
  3. Sets socket state to TCP_SYN_SENT
  4. connect() BLOCKS (waits for SYN-ACK)
  5. SYN-ACK arrives → kernel sends ACK
  6. Sets socket state to TCP_ESTABLISHED
  7. connect() returns success

SERVER calls accept():
  1. accept() BLOCKS (waits for a connection)
  2. SYN arrives → kernel sends SYN-ACK
  3. Kernel creates a NEW struct sock for this connection
  4. Puts it on the "SYN received" queue
  5. ACK arrives → connection complete
  6. Moves the struct sock to the "established" queue
  7. accept() returns a NEW fd pointing to the new socket
  8. The ORIGINAL listening socket continues listening
```

**Key insight:** `accept()` creates a BRAND NEW socket. The listening
socket (fd 3) keeps listening. The new socket (fd 4) is dedicated to
this one client. If 100 clients connect, you get 100 new fds, all
separate from the listening fd.


---

# Part G: Socket Buffers — Where Data Waits

## Chapter 14: The Send and Receive Queues

Every socket has two queues of sk_buffs:

```
struct sock for a connected TCP socket:

  sk_write_queue (SEND BUFFER):
  ┌──────┬──────┬──────┐
  │ skb  │ skb  │ skb  │ → data YOU wrote but not yet ACKed by receiver
  │ 1KB  │ 500B │ 2KB  │
  └──────┴──────┴──────┘
  Total: 3.5KB of data waiting to be sent or ACKed.
  Limit: sk_sndbuf (default: 16KB - 4MB, auto-tuned)

  sk_receive_queue (RECEIVE BUFFER):
  ┌──────┬──────┐
  │ skb  │ skb  │ → data THEY sent but you haven't read() yet
  │ 800B │ 1KB  │
  └──────┴──────┘
  Total: 1.8KB of data waiting for your read().
  Limit: sk_rcvbuf (default: 16KB - 4MB, auto-tuned)
```

**What happens when you write() faster than the network can send:**

```
Your code:
  stream.write_all(&huge_data)?;   // 1MB of data

Kernel:
  Copy chunks into sk_write_queue.
  TCP sends what it can (limited by receiver's window size).
  
  If the send buffer is FULL:
    write() BLOCKS! Your process sleeps until space is freed.
    (Space is freed when the receiver ACKs received data.)
  
  This is TCP's FLOW CONTROL — it prevents you from
  overwhelming the receiver or the network.
```

**What happens when data arrives but you don't read():**

```
Network:
  Data arrives from the sender.
  TCP puts it in sk_receive_queue.
  More data arrives. Queue grows.
  
  If the receive buffer is FULL:
    TCP tells the sender: "Window size = 0. STOP SENDING."
    The sender pauses until we read() and free space.
    
  This is also flow control — it prevents the sender from
  filling our RAM with unread data.
```


---

# Part H: How the Kernel Finds the Right Socket

## Chapter 15: The Socket Hash Table

When a packet arrives from the network, the kernel must figure out
which socket it belongs to. It does this with a hash table:

```
Incoming packet:
  Source IP:   93.184.216.34
  Source Port: 80
  Dest IP:     192.168.1.10
  Dest Port:   54321

The kernel computes:
  hash = hash(src_ip, src_port, dst_ip, dst_port)
  bucket = hash % table_size
  
  Search bucket for a struct sock matching all 4 values.
  
  ┌─ Socket Hash Table ─────────────────────────────────┐
  │                                                      │
  │ bucket 0:  (empty)                                   │
  │ bucket 1:  sock A: (10.0.0.1:22 ↔ 10.0.0.2:55000) │
  │ bucket 2:  (empty)                                   │
  │ bucket 3:  sock B: (192.168.1.10:54321 ↔             │
  │                      93.184.216.34:80)   ★ MATCH!    │
  │ bucket 4:  (empty)                                   │
  │ ...                                                  │
  └──────────────────────────────────────────────────────┘

Found sock B! Deliver the packet to sock B's receive queue.
```

This is O(1) — a hash computation + a short chain walk.
The kernel can handle millions of connections because the hash table
lookup is constant time regardless of how many sockets exist.


---

# Part I: Connection to Your Storage Engine

## Chapter 16: Why This Matters for You

If you ever add a network API to your storage engine (so clients can
connect over TCP and send queries), here's how it would work:

```
CLIENT (your app)                    FERRISDB SERVER
─────────────────                    ──────────────────

TcpStream::connect(                  TcpListener::bind(
  "db-server:5432"                     "0.0.0.0:5432"
)                                    )
  │                                    │
  │ ── TCP handshake ──────────────── │
  │                                    │
  │                                  accept() → new socket
  │                                    │
write("GET account_1001")            read() → "GET account_1001"
  │                                    │
  │  bytes travel through:             │  kernel receives bytes from:
  │  tcp_sendmsg → IP → driver → wire │  wire → driver → IP → tcp_v4_rcv
  │                                    │  → sk_receive_queue → read()
  │                                    │
  │                                  Your engine:
  │                                    B+ tree search("account_1001")
  │                                    → RID(7, 15)
  │                                    → fetch_page(7) from buffer pool
  │                                    → read slot 15
  │                                    → "Mohamed|Cairo|5000.00"
  │                                    │
  │                                  write("Mohamed|Cairo|5000.00")
  │                                    │
read() → "Mohamed|Cairo|5000.00"       │
```

**The entire network stack is just another I/O path.** Instead of:
- user buffer → page cache → block I/O → disk (file write)

You have:
- user buffer → socket buffer → TCP → IP → NIC → wire (network send)

Same pattern: your data moves through layers, each adding its own
headers/processing, until it reaches the destination.

```
FILE I/O LAYERS:              NETWORK I/O LAYERS:
──────────────                ─────────────────────
VFS                           VFS (same!)
  ↓                             ↓
ext4 file_operations          socket_file_ops
  ↓                             ↓
Page cache                    TCP (sk_write_queue)
  ↓                             ↓
Block I/O layer               IP layer
  ↓                             ↓
Disk device driver            Network device driver
  ↓                             ↓
SSD hardware                  NIC hardware
  ↓                             ↓
NAND flash                    Copper/fiber/radio waves
```

Both paths start at the VFS with the same write() syscall.
Both paths end at hardware. The middle layers are different
but the architecture — layered dispatch through operations
tables — is identical. This is the power of Unix's
"everything is a file" design.
