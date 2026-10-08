# 🤖 eBPF Hello World with GoLang
[ebpf.io](https://ebpf.io)

## 📦 Requirements:
1. **Install** Linux VM _(via multipass)_ - _required for linux headers_
2. **Install** the required packages.
    ```
    - net-tools
    - golang-go
    - clang
    - llvm
    - linux-libc-dev
    - linux-tools-common
    - linux-tools-generic
    - libbpf-dev
    - linux-lowlatency-tools-common
    - build-essential
    - python3
    - python3-pip
    - python3-venv
    ```
    **2a**. Provision and boostrap the Workspace VM.
    ```
    $ ./_setup.sh
    ```

## 📦 Compile eBPF & Go Program
1. **SSH** to the virtual environment
    ```
    $ multipass shell ebpf-working-environment1
    ```
2. **Compile** eBPF Program _(Kernel Space)_
    ```
    $ cd /mnt/workspace/bpf
    $ make clean && make
    ```
3.  **Compile** Go Program
    ```
    $ cd /mnt/workspace/cmd
    $ make clean && make init && make build
    ```
4. **Run** the Program
    ```
    $ sudo ./helloworld
    ```

## ⏱️ Trigger kernel syscall `do_sys_openat2`
Run: `cat /etc/hosts` (this triggers `do_sys_openat2`)

### 📝 Response
```
2026/08/20 00:25:41 eBPF program loaded
2026/08/20 00:25:41 waiting for events...
2026/08/20 00:25:41 EVENT: Hello, eBPF! <--- response from syscall event
```

### 💡 What happened?
Every kernel syscall `do_sys_openat2()` a hook triggers a `hello()` function which sends event message to userspace which then reads by Go Program.

## 🗿 Kernel to Userspace Structure
```
 Linux kernel
      │
      │ do_sys_openat2()
      ▼
 ┌──────────────┐
 │ eBPF program │
 │    hello()   │
 └──────┬───────┘
        │
        │ event
        ▼
 ┌──────────────┐
 │ Ring Buffer  │
 └──────┬───────┘
        │
        │ userspace reads / rd.Read() in go userspace
        ▼
 ┌──────────────┐
 │ Go program   │
 └──────────────┘

 eBPF kernel                        Go
    │                               │
    │       ring buffer             │
    │  ┌───────────────────────┐    │
    ├─►│ event │ event │ event │───►│
    │  └───────────────────────┘    │
    │                               │
    └── produce                  consume
```

## 🗿 Trigger Flow
```
 cat
  │
  │ open()
  ▼
 Linux kernel
  │
  ▼
 do_sys_openat2()
  │
  │ kprobe fires
  ▼
 hello()
  │
  ├── reserve 32 bytes
  │
  ├── write "Hello, eBPF!"
  │
  └── submit event
  │
  ▼
 BPF ring buffer
  │
  │ Go reads
  ▼
 Go userspace
  │
  ▼
 EVENT: Hello, eBPF! <------ log.Printf()
```

## 💡 The three most important pieces
### ⚡️ Kernel eBPF: `C Code`
```
bpf_ringbuf_submit(e, 0);
```

### ⚡️ Go ring-buffer reader:
```
record, err := rd.Read()
```

### ⚡️ Go event decoder:
```
record, err := rd.Read()
binary.Read(..., &event)
```
```
KERNEL                 USERSPACE
bpf_ringbuf_submit()
	│
	▼
Ring buffer
	│
	│
	└──────────────► rd.Read()
							│
							▼
						record
```

# 🤖 Real-world Applications

## 📦 Network Tracer
Bind to Kernel `tracepoint/sock/inet_sock_set_state` and capture each connection states and calculate latency.

### ⚡️ Network Call
```
$ curl https://httpbin.org/delay/5
{
  "args": {}, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/8.5.0", 
    "X-Amzn-Trace-Id": "Root=1-6a944381-2bd25f303829a6cc75d7c0f1"
  }, 
  "origin": "13.63.92.146", 
  "url": "https://httpbin.org/delay/5"
}
```

### 📝 eBPF Trace Response
```
2026/10/08 15:56:53 eBPF NET TRACER Running...
TIME    TIME_NS_SINCE_BOOT      PID     COMM    CWD     CMD     CONN    LATENCY
10-08-2026T16:00:28.201693      17063051669693  42357   curl    /apps/workspace/ebpf-demo/ebpf-go-apps/bpf/nettrace-test        curl https://httpbin.org/delay/5        172.31.36.100:0->3.90.119.127:443[CLOSE]->[SYN_SENT]    0.000002s
10-08-2026T16:00:35.201731      17070051689027  42357   curl    /apps/workspace/ebpf-demo/ebpf-go-apps/bpf/nettrace-test        curl https://httpbin.org/delay/5        172.31.36.100:39950->3.90.119.127:443[ESTABLISHED]->[FIN_WAIT1] 7.000022s
10-08-2026T16:00:35.313948      17070161617435  42357   ebpf-demo                       172.31.36.100:39950->3.90.119.127:443[FIN_WAIT1]->[CLOSING]     7.109950s
```
Note: It shows Latency between syscall eg. `[FIN_WAIT1]` it waits around 5s.

## ⚡️ Monitor attache eBPF loaded programs.
### 👾 List loaded eBPF programs
eBPF programs loaded into the kernel.
```
$ bpftool prog show | grep "trace_tcp_state" -A3
56: tracepoint  name trace_tcp_state  tag 924dc88896b201b9  gpl
        loaded_at 2026-10-08T11:27:57+0000  uid 0
        xlated 1248B  jited 759B  memlock 4096B  map_ids 3,4
        btf_id 55
```
### 👾 List eBPF maps
Maps holds the data generated from kernel-space.
```
$ bpftool map show

# maps declared in eBPF Programs
3: lru_hash  name connections  flags 0x0
        key 8B  value 32B  max_entries 65536  memlock 6816640B
        btf_id 54
4: ringbuf  name events  flags 0x0
        key 0B  value 0B  max_entries 16777216  memlock 16855360B
        btf_id 54
```
### 👾 Dump maps data
curl event is triggered for the map to contain data.
```
$ curl https://httpbin.org/delay/5; bpftool map dump id 3
{
  "args": {}, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/8.5.0", 
    "X-Amzn-Trace-Id": "Root=1-6ac780bf-094ba3cc6721a90f24e185d4"
  }, 
  "origin": "13.63.92.146", 
  "url": "https://httpbin.org/delay/5"
}
[{
    "key": 18446617228410057216,
    "value": {
        "saddr": 1680089004,
        "daddr": 2512035638,
        "sport": 58712,
        "dport": 47873,
        "pid": 4999,
        "tgid": 4999,
        "start_ns": 1353878813428
}]
```

## 📦 System Call Tracer
Detect all system call entry by tapping from `raw_tracepoint/sys_enter`.
_Example C Program that invokes couple of kernel syscalls_
[systrace-test.c](./ebpf-go-hello-world/bpf/systrace-test/systrace-test.c)
```
make build && ./systrace-test testfile # shows pid 26873
```

### eBPF Trace Response
```
Tracing PID 26873
PID 26873 exists
2026/10/08 16:02:47 eBPF SYS TRACER Running...
TIME    TIME_NS_SINCE_BOOT      PID     COMM    CWD     CMD     SYS_CALL        SYS_CALL_ID
10-08-2026T16:02:49.249966      17204099880404  26873   systrace-test   /apps/workspace/ebpf-demo/ebpf-go-apps/bpf/systrace-test        ./systrace-test testfile        openat  257
10-08-2026T16:02:49.250048      17204099910336  26873   systrace-test   /apps/workspace/ebpf-demo/ebpf-go-apps/bpf/systrace-test        ./systrace-test testfile        fstat   5
10-08-2026T16:02:49.250085      17204099959747  26873   systrace-test   /apps/workspace/ebpf-demo/ebpf-go-apps/bpf/systrace-test        ./systrace-test testfile        read    0
10-08-2026T16:02:49.250121      17204099966823  26873   systrace-test   /apps/workspace/ebpf-demo/ebpf-go-apps/bpf/systrace-test        ./systrace-test testfile        close   3
10-08-2026T16:02:49.250151      17204099976551  26873   systrace-test   /apps/workspace/ebpf-demo/ebpf-go-apps/bpf/systrace-test        ./systrace-test testfile        clock_nanosleep 230
```
It shows the list of syscall made by the running program eg. `read()` `clock_nanosleep()`

## 📦 Malloc Tracer
Bind `uprobe/malloc` which detects when userspace allocate memory via `malloc()`
_Example C Program that allocates memory via malloc_
[malloc-test.c](./ebpf-go-hello-world/bpf/malloc-test/malloc-test.c)
```
./malloc-test 10496000
PID: 88199
Allocated: 10496000 bytes (10.01 MiB)
```

### eBPF Trace Response
```
eBPF Program=trace_malloc Section=uprobe/malloc
Arch: amd64 | libcPath: /lib/x86_64-linux-gnu/libc.so.6
TIME    TIME_NS_SINCE_BOOT      PID     COMM    CWD     CMD     SID     TGID    UID     SIZE
10-08-2026T15:59:57.039351      17031889340995  42050   malloc-test     /apps/workspace/ebpf-demo/ebpf-go-apps/bpf/malloc-test  ./malloc-test 10485760  18446744073709551599    42050   1000   malloc: 10485760 bytes: (10.00 MB)
```

The trace shows the exact amount of memory allocated and who allocated it including the PID Command.


**References:**
- https://manual.cs50.io/
- https://docs.ebpf.io/ebpf-library/libbpf/ebpf/BPF_CORE_READ/
- https://github.com/cilium/ebpf
- https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md
- https://man7.org/linux/man-pages/man7/bpf-helpers.7.html
- https://elixir.bootlin.com/linux/v7.2/source/tools/testing/selftests/bpf
- https://100go.co/#error-management