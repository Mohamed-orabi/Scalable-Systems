# One Server, 100 Clients — How Sockets Handle This

## The Complete Guide to Connection Multiplexing

---

# Chapter 1: The Short Answer

**No, the same socket is NOT shared.** Each client gets its OWN dedicated
socket. The server has ONE listening socket that CREATES a new socket
for each client that connects.

```
WRONG mental model:
  "One socket shared by 100 clients"
  
  Client A ──┐
  Client B ──┤──► ONE socket ──► Server
  Client C ──┘

CORRECT mental model:
  "One LISTENING socket + one DEDICATED socket per client"

  Client A ──► Socket A ──┐
  Client B ──► Socket B ──┤──► Server process
  Client C ──► Socket C ──┘
                          │
  Listening socket ───────┘ (creates A, B, C on accept())
```


---

# Chapter 2: The Analogy — A Restaurant

```
THE LISTENING SOCKET = the restaurant's FRONT DOOR

  The front door doesn't serve food.
  It doesn't take orders.
  It ONLY does one thing: when a customer arrives,
  it assigns them to a TABLE.

  The front door stays open for the NEXT customer.
  It never "belongs to" any one customer.


A CONNECTED SOCKET = a TABLE in the restaurant

  Each customer gets their own table.
  The waiter takes orders and brings food AT THAT TABLE.
  Customer A's conversation doesn't interfere with Customer B's.
  
  Table A: Customer A ↔ Waiter (private conversation)
  Table B: Customer B ↔ Waiter (different private conversation)
  Table C: Customer C ↔ Waiter (yet another)


THE FLOW:

  1. Customer arrives at the FRONT DOOR (client calls connect())
  2. Front door says: "Table 4 is available" (accept() returns new fd)
  3. Customer sits at Table 4 (client connected to dedicated socket)
  4. Front door is ready for the NEXT customer (listen continues)
  5. Orders and food flow between customer and Table 4 (read/write)
  6. Customer leaves (close()) → Table 4 freed for reuse
```


---

# Chapter 3: The Code — How It Actually Works

```rust
use std::net::TcpListener;
use std::io::{Read, Write};
use std::thread;

fn main() {
    // ═══════════════════════════════════════════════════════════
    //  Step 1: Create the LISTENING socket
    //  This is the "front door." It does NOT handle any client data.
    //  Kernel: socket() + bind() + listen()
    // ═══════════════════════════════════════════════════════════
    
    let listener = TcpListener::bind("0.0.0.0:8080").unwrap();
    // listener fd = 3 (the listening socket)
    
    println!("Server listening on port 8080...");
    println!("This socket (fd 3) will NEVER read or write client data.");
    println!("It ONLY creates new sockets via accept().\n");

    // ═══════════════════════════════════════════════════════════
    //  Step 2: Accept loop — create a NEW socket for each client
    //  Each accept() returns a BRAND NEW socket dedicated to ONE client.
    // ═══════════════════════════════════════════════════════════

    for (client_number, stream) in listener.incoming().enumerate() {
        let mut stream = stream.unwrap();
        
        // 'stream' is a NEW socket — fd 4, 5, 6, 7, 8...
        // Each client gets their OWN fd, their OWN socket.
        
        let client_addr = stream.peer_addr().unwrap();
        println!("Client {} connected from {} (new socket created)",
            client_number, client_addr);

        // ═══════════════════════════════════════════════════════
        //  Step 3: Handle this client on their dedicated socket
        //  Often done in a separate thread (so we don't block
        //  the accept loop for other clients).
        // ═══════════════════════════════════════════════════════
        
        thread::spawn(move || {
            // This thread handles ONE client on ONE socket.
            // Other clients have their own threads and sockets.
            
            let mut buffer = [0u8; 1024];
            
            // Read from THIS client's socket
            let n = stream.read(&mut buffer).unwrap();
            println!("Client {}: received {:?}",
                client_number,
                String::from_utf8_lossy(&buffer[..n]));
            
            // Write back to THIS client's socket
            stream.write_all(b"Hello from server!").unwrap();
            
            // When 'stream' is dropped, THIS client's socket is closed.
            // The listening socket and other clients are UNAFFECTED.
        });
        
        // The accept loop continues IMMEDIATELY.
        // It doesn't wait for the thread to finish.
        // The NEXT client can connect right away.
    }
}
```


---

# Chapter 4: The Kernel's File Descriptor Table

After 5 clients connect, the server's fd table looks like this:

```
Server process fd table:

  fd 0 → struct file → stdin
  fd 1 → struct file → stdout
  fd 2 → struct file → stderr
  fd 3 → struct file → LISTENING socket ← accept() uses this
  fd 4 → struct file → CONNECTED socket ← Client A (from 10.0.0.1:50001)
  fd 5 → struct file → CONNECTED socket ← Client B (from 10.0.0.2:50002)
  fd 6 → struct file → CONNECTED socket ← Client C (from 10.0.0.3:50003)
  fd 7 → struct file → CONNECTED socket ← Client D (from 10.0.0.4:50004)
  fd 8 → struct file → CONNECTED socket ← Client E (from 10.0.0.5:50005)
```

**fd 3 is the listening socket.** It never sends or receives data.
Its ONLY job is to produce new connected sockets via accept().

**fd 4-8 are connected sockets.** Each is a private channel to one client.
Writing to fd 4 sends data ONLY to Client A. Reading from fd 6 receives
data ONLY from Client C. They are completely independent.


---

# Chapter 5: What accept() Actually Does in the Kernel

```
Server calls: accept(fd=3, &client_addr)

The kernel does:

Step 1: Find the listening socket
  fd 3 → struct file → struct socket → struct sock
  sock->sk_state = TCP_LISTEN
  "This is a listening socket. Good."

Step 2: Check the accept queue
  The listening socket has a queue of COMPLETED connections
  (clients that finished the TCP handshake):
  
  ┌─ Listening socket (fd 3) ──────────────────────────────┐
  │                                                         │
  │  sk_state = TCP_LISTEN                                  │
  │  sk_sport = 8080 (our port)                            │
  │  sk_saddr = 0.0.0.0 (all interfaces)                  │
  │                                                         │
  │  ACCEPT QUEUE (completed connections waiting):          │
  │  ┌─────────────────────────────────────────────────┐   │
  │  │ sock: 10.0.0.6:50006 ↔ us:8080 (ESTABLISHED)   │   │
  │  │ sock: 10.0.0.7:50007 ↔ us:8080 (ESTABLISHED)   │   │
  │  │ sock: 10.0.0.8:50008 ↔ us:8080 (ESTABLISHED)   │   │
  │  └─────────────────────────────────────────────────┘   │
  │                                                         │
  │  If queue is EMPTY: accept() blocks (process sleeps).   │
  │  If queue is NON-EMPTY: pop the first one.              │
  └─────────────────────────────────────────────────────────┘

Step 3: Pop a connection from the accept queue
  Take: sock for 10.0.0.6:50006 ↔ us:8080
  This is a FULLY ESTABLISHED connection (3-way handshake already done).

Step 4: Create a NEW struct file for this connection
  new_file = alloc_file()
  new_file->f_op = socket_file_ops
  new_file->private_data = new_socket (wrapping the popped sock)

Step 5: Allocate a new fd number
  Find lowest unused fd in the process's table → fd 9
  fd_table[9] = new_file

Step 6: Fill in the client's address
  client_addr = { IP=10.0.0.6, port=50006 }

Step 7: Return
  accept() returns fd 9.
  Your code receives a TcpStream wrapping fd 9.
```

**The key insight:** The new sock (with the established connection)
was already created by the kernel DURING the TCP handshake — before
accept() was even called. The SYN arrives, the kernel creates a new
sock, does the SYN-ACK/ACK exchange, and puts the completed sock on
the accept queue. accept() just picks it up.

```
TIMELINE:

  Time 1: Client sends SYN
  Time 2: Kernel receives SYN on listening socket
          Kernel creates a NEW struct sock for this connection
          Kernel sends SYN-ACK
  Time 3: Client sends ACK
  Time 4: Kernel receives ACK
          Connection is ESTABLISHED
          Kernel puts the new sock on the accept queue
  Time 5: Your code calls accept()
          accept() pops the sock from the queue
          Returns a new fd
          
  The 3-way handshake happens WITHOUT your code doing anything.
  accept() is just "give me the next completed connection."
```


---

# Chapter 6: The Kernel Data Structures — Listening vs Connected

```
LISTENING SOCKET (fd 3):

  struct file ──► struct socket ──► struct sock (TCP_LISTEN)
                    │
                    │ ops = inet_stream_ops
                    │
                    └──► struct sock:
                          sk_state = TCP_LISTEN
                          sk_saddr = 0.0.0.0
                          sk_sport = 8080
                          sk_daddr = (none — not connected to anyone)
                          sk_dport = (none)
                          
                          sk_receive_queue = (unused — listeners don't receive data)
                          sk_write_queue = (unused — listeners don't send data)
                          
                          icsk_accept_queue = ★ THE ACCEPT QUEUE
                            → completed connections waiting for accept()


CONNECTED SOCKET for Client A (fd 4):

  struct file ──► struct socket ──► struct sock (TCP_ESTABLISHED)
                    │
                    │ ops = inet_stream_ops (same ops, different state)
                    │
                    └──► struct sock:
                          sk_state = TCP_ESTABLISHED
                          sk_saddr = 192.168.1.10     (our IP)
                          sk_sport = 8080              (our port)
                          sk_daddr = 10.0.0.1          (Client A's IP)
                          sk_dport = 50001             (Client A's port)
                          
                          sk_receive_queue = [data from Client A]
                          sk_write_queue = [data going to Client A]


CONNECTED SOCKET for Client B (fd 5):

  struct file ──► struct socket ──► struct sock (TCP_ESTABLISHED)
                    │
                    └──► struct sock:
                          sk_state = TCP_ESTABLISHED
                          sk_saddr = 192.168.1.10     (our IP — same!)
                          sk_sport = 8080              (our port — same!)
                          sk_daddr = 10.0.0.2          (Client B's IP — different!)
                          sk_dport = 50002             (Client B's port — different!)
                          
                          sk_receive_queue = [data from Client B]
                          sk_write_queue = [data going to Client B]
```

**Notice:** Both connected sockets have the SAME source IP and source port
(192.168.1.10:8080). They're distinguished by the DESTINATION IP and port.
This is the **4-tuple**: (src_ip, src_port, dst_ip, dst_port). Every connection
has a unique 4-tuple.

```
Connection A: (192.168.1.10, 8080, 10.0.0.1, 50001) → unique!
Connection B: (192.168.1.10, 8080, 10.0.0.2, 50002) → unique!
Connection C: (192.168.1.10, 8080, 10.0.0.3, 50003) → unique!

ALL share the same server IP:port.
ALL are different because the client IP:port is different.
```


---

# Chapter 7: How the Kernel Routes Incoming Packets

When a packet arrives, the kernel must figure out: does this go to the
listening socket or to one of the connected sockets?

```
INCOMING PACKET:
  src_ip=10.0.0.2, src_port=50002, dst_ip=192.168.1.10, dst_port=8080
  Payload: "SELECT * FROM accounts"

KERNEL LOOKUP:

  Step 1: Hash the 4-tuple
    hash(10.0.0.2, 50002, 192.168.1.10, 8080) → bucket 47
  
  Step 2: Search the established connections hash table
    Bucket 47:
      sock: (192.168.1.10:8080 ↔ 10.0.0.1:50001) → no match
      sock: (192.168.1.10:8080 ↔ 10.0.0.2:50002) → ★ MATCH!
  
  Step 3: Deliver to Client B's connected socket
    Add the data to fd 5's sk_receive_queue.
    Wake up any thread blocked on read(fd=5).

The listening socket (fd 3) is NOT involved in data delivery.
It only handles NEW connection requests (SYN packets).
```

What about a SYN packet from a NEW client?

```
INCOMING SYN PACKET:
  src_ip=10.0.0.9, src_port=50009, dst_ip=192.168.1.10, dst_port=8080
  Flags: SYN

KERNEL LOOKUP:

  Step 1: Hash the 4-tuple
    hash(10.0.0.9, 50009, 192.168.1.10, 8080) → bucket 83
  
  Step 2: Search established connections → NOT FOUND (it's new!)
  
  Step 3: Search listening sockets
    Is anything listening on 192.168.1.10:8080?
    → Yes! fd 3's sock is TCP_LISTEN on port 8080.
  
  Step 4: This is a NEW connection
    Create a new struct sock for this connection.
    Send SYN-ACK.
    After handshake completes → put on accept queue.
    Next accept() call will pick it up.
```


---

# Chapter 8: Handling 100 Clients — Three Approaches

## Approach 1: One Thread Per Client (Simplest)

```rust
// What we showed earlier — spawn a thread for each client

for stream in listener.incoming() {
    let stream = stream.unwrap();
    thread::spawn(move || {
        handle_client(stream);  // each thread handles one client
    });
}

PROS:
  ✓ Simple code
  ✓ One blocked client doesn't affect others
  ✓ Fine for ~100-1000 clients

CONS:
  ✗ 10,000 clients = 10,000 threads = lots of memory (~8MB stack each)
  ✗ Context switching overhead with many threads
  ✗ Not practical for 100,000+ connections
  
Kernel state:
  100 threads, each with:
    - Their own fd pointing to a connected socket
    - Their own kernel stack (8KB)
    - Scheduler must manage all 100 threads
```

## Approach 2: epoll — One Thread, Many Sockets (Efficient)

```rust
// Pseudocode for the epoll approach (used by Tokio, nginx, etc.)

let epoll = epoll_create()?;

// Register the listening socket
epoll_ctl(epoll, EPOLL_CTL_ADD, listener_fd, EPOLLIN)?;

// Also register all connected client sockets
let mut clients: HashMap<i32, TcpStream> = HashMap::new();

loop {
    // Wait for ANY socket to be ready (readable or writable)
    let events = epoll_wait(epoll, &mut event_buf, timeout)?;
    //
    // This ONE call monitors ALL sockets simultaneously!
    // The thread sleeps until at least one socket has data.
    
    for event in events {
        if event.fd == listener_fd {
            // New client connecting!
            let (client_stream, addr) = listener.accept()?;
            epoll_ctl(epoll, EPOLL_CTL_ADD, client_fd, EPOLLIN)?;
            clients.insert(client_fd, client_stream);
            
        } else {
            // Existing client has data to read
            let stream = clients.get_mut(&event.fd).unwrap();
            let n = stream.read(&mut buffer)?;
            // Process the request...
            stream.write_all(&response)?;
        }
    }
}

PROS:
  ✓ ONE thread handles ALL clients
  ✓ Scales to 100,000+ connections
  ✓ Minimal memory (no per-client thread stack)
  ✓ No context switching overhead

CONS:
  ✗ More complex code (event-driven, non-blocking)
  ✗ Can't do blocking operations (everything must be async)
  
Kernel state:
  1 thread with:
    - All 100 client fds registered in one epoll instance
    - epoll_wait() sleeps until ANY socket has an event
    - Kernel maintains a red-black tree of monitored fds
    - Kernel maintains a ready list of fds with pending events
```

### How epoll Works in the Kernel

```
struct eventpoll {
    struct rb_root_cached rbr;    // Red-black tree of ALL monitored fds
    struct list_head rdllist;     // READY LIST: fds with pending events
    wait_queue_head_t wq;        // processes sleeping in epoll_wait()
};

When you add a socket to epoll:
  → Insert into the red-black tree (O(log N))
  → Register a callback on the socket: "When data arrives,
    add me to the ready list and wake up epoll_wait()"

When data arrives on a socket:
  → TCP puts data in sk_receive_queue (as always)
  → TCP calls the epoll callback
  → Callback adds this fd to epoll's ready list
  → Callback wakes up any process sleeping in epoll_wait()

When your code calls epoll_wait():
  → Check the ready list
  → If non-empty: return immediately with the ready fds
  → If empty: sleep until a callback wakes us up

epoll_wait() does NOT scan all 100 sockets to check which ones
have data. It just looks at the READY LIST — only sockets that
already have events. This is O(ready_count), not O(total_count).
That's why epoll scales to millions of connections.
```

## Approach 3: async/await (Tokio) — Best of Both Worlds

```rust
// Using Tokio — Rust's async runtime (built on epoll internally)

use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();
    
    loop {
        let (mut stream, addr) = listener.accept().await.unwrap();
        
        // spawn an async TASK (not a thread — much lighter!)
        tokio::spawn(async move {
            let mut buf = [0u8; 1024];
            let n = stream.read(&mut buf).await.unwrap();
            stream.write_all(b"Hello!").await.unwrap();
        });
    }
}

// Under the hood, Tokio uses epoll (Linux) or kqueue (macOS).
// The .await points are where the task YIELDS control —
// it doesn't block a thread, it just says "wake me when data arrives"
// and lets another task run on the same thread.

PROS:
  ✓ Looks like simple per-client code (easy to write)
  ✓ Scales like epoll (100,000+ connections)
  ✓ Tasks are much cheaper than threads (~few hundred bytes vs 8MB)

CONS:
  ✗ Must use async/await throughout (can't mix with blocking code)
  ✗ Slightly more complex error handling
```


---

# Chapter 9: The Complete Picture — 100 Clients Connected

```
SERVER PROCESS (one process handling 100 clients):

  fd table:
    fd 0:   stdin
    fd 1:   stdout
    fd 2:   stderr
    fd 3:   LISTENING socket (port 8080) ── only accept()s, never read/write
    fd 4:   Client 1  (10.0.0.1:50001 ↔ us:8080)  ── private channel
    fd 5:   Client 2  (10.0.0.2:50002 ↔ us:8080)  ── private channel
    fd 6:   Client 3  (10.0.0.3:50003 ↔ us:8080)  ── private channel
    ...
    fd 103: Client 100 (10.0.0.100:50100 ↔ us:8080)

  In the kernel:

  ┌─ Socket Hash Table ────────────────────────────────────────┐
  │                                                             │
  │  LISTEN table:                                              │
  │    (*, 8080) → listening sock (fd 3)                        │
  │                                                             │
  │  ESTABLISHED table:                                         │
  │    (us:8080, 10.0.0.1:50001)   → sock for fd 4             │
  │    (us:8080, 10.0.0.2:50002)   → sock for fd 5             │
  │    (us:8080, 10.0.0.3:50003)   → sock for fd 6             │
  │    ...                                                      │
  │    (us:8080, 10.0.0.100:50100) → sock for fd 103           │
  │                                                             │
  │  Each connected sock has its OWN:                           │
  │    - sk_receive_queue (data waiting to be read)             │
  │    - sk_write_queue (data waiting to be sent)               │
  │    - TCP sequence numbers                                   │
  │    - Retransmission timers                                  │
  │    - Congestion window                                      │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘

  Incoming packet routing:

    Packet from 10.0.0.42:50042 with data "GET /accounts"
      → kernel hashes (us:8080, 10.0.0.42:50042)
      → finds fd 45's sock
      → adds data to fd 45's receive queue
      → wakes up the thread/task handling fd 45

    Packet from 10.0.0.99:50099 with data "INSERT INTO..."
      → kernel hashes (us:8080, 10.0.0.99:50099)
      → finds fd 102's sock
      → adds data to fd 102's receive queue
      → wakes up the thread/task handling fd 102

    SYN from 10.0.0.101:50101 (new client!)
      → no established connection found
      → falls through to listening socket (fd 3)
      → kernel does SYN-ACK
      → after handshake: new sock on accept queue
      → next accept() → fd 104
```


---

# Chapter 10: Connection to Your Storage Engine

If you build a network API for FerrisDB:

```
FerrisDB Server Architecture:

  ┌─────────────────────────────────────────────────────────┐
  │  NETWORK LAYER (sockets)                                 │
  │                                                         │
  │  Listening socket (port 5432) ─── accept loop           │
  │     │                                                   │
  │     ├── Client 1 socket ──→ parse query ──→ execute ──┐ │
  │     ├── Client 2 socket ──→ parse query ──→ execute ──┤ │
  │     ├── Client 3 socket ──→ parse query ──→ execute ──┤ │
  │     └── ...                                            │ │
  │                                                        │ │
  ├────────────────────────────────────────────────────────│─┤
  │  STORAGE ENGINE (what you built in Phases 1-3)         │ │
  │                                                        ▼ │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ Transaction Manager                              │     │
  │  │   begin() / commit() / abort()                   │     │
  │  ├─────────────────────────────────────────────────┤     │
  │  │ B+ Tree ←──── search / insert / delete           │     │
  │  ├─────────────────────────────────────────────────┤     │
  │  │ Buffer Pool ←── fetch_page / flush_page          │     │
  │  ├─────────────────────────────────────────────────┤     │
  │  │ WAL ←── append / flush                           │     │
  │  ├─────────────────────────────────────────────────┤     │
  │  │ Page Manager ←── read_page / write_page          │     │
  │  └─────────────────────────────────────────────────┘     │
  │                         │                                │
  │                    ┌────┴────┐                           │
  │                    │  Disk   │                           │
  │                    └─────────┘                           │
  └─────────────────────────────────────────────────────────┘

  Each client gets:
    - Their own socket (private network channel)
    - Their own transaction (private database session)
    - Shared access to the same buffer pool and B+ tree
      (concurrency controlled by locks/MVCC — Phase 3)
      
  The socket layer and the storage layer are INDEPENDENT:
    - Sockets handle network bytes
    - Storage engine handles database pages
    - They meet in the middle: "Client 42 sent 'GET key=1005',
      the engine searched the B+ tree, found the record,
      and we send the result back through Client 42's socket."
```

**The parallel is exact:**

```
Socket accept queue             ←→  Buffer pool's frame allocation
  "Pending connections waiting       "Pages waiting to be loaded
   for the server to handle"          into memory"

One socket per client           ←→  One transaction per client
  "Private communication channel"    "Private database session"

sk_receive_queue                ←→  (client's query bytes)
  "Data the client sent us"          "The SQL or key-value command"

sk_write_queue                  ←→  (query result bytes)
  "Data we're sending back"          "The records we found"

epoll (monitoring 100 sockets)  ←→  (thread pool processing queries)
  "Wait for ANY client to send       "Execute the next available
   a request, then handle it"         query from any client"
```
