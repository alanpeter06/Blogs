Using TLS to secure a NVMe TCP Connection

To secure remote storage access in NVMe over Fabrics, traditional approaches rely on specialized interconnects like RDMA or Fibre Channel, which can be complex and costly to deploy. The NVMe Specification \[[1](https://nvmexpress.org/specifications/)\] introduces two mechanisms to address this by enabling secure, low-latency access over standard TCP/IP networks: (1) NVMe In-Band Authentication, supported by Oracle UEK7, which uses a challenge-response protocol with a shared secret for secure connections (see Blog - NVMe In-band Authentication \[[2](https://blogs.oracle.com/linux/post/nvme-inband-authentication)\]), and (2) Fabric Secure Channel, implemented as NVMe-TLS in the Oracle UEK8 kernel \[[3](https://blogs.oracle.com/linux/post/oracle-linux-now-includes-the-latest-linux-kernel-with-uek-8)\], which encrypts NVMe-TCP traffic to ensure data protection. These solutions leverage existing Ethernet infrastructure, simplifying deployment while maintaining robust security, and will be described by this blog.

### NVMe-TCP

NVMe over TCP (NVMe-TCP) which is part of the NVMe over Fabrics family, is a protocol that allows the NVMe (Non-Volatile Memory Express) storage protocol to function over standard TCP/IP networks. While NVMe is typically used for direct communication between the host system and storage devices via the PCIe bus, NVMe-TCP enables this protocol to work over conventional IP-based networks, such as Ethernet. See the blog - NVMe over TCP \[[4](https://blogs.oracle.com/linux/post/nvme-over-tcp)\].

### Linux In-Kernel TLS

TLS (Transport Layer Security) is a cryptographic protocol designed to provide secure communication over a computer network. It ensures that data transmitted between two parties (such as a client and server) remains private and integral, protecting it from eavesdropping, tampering, and forgery.

Here are the key features of TLS:

1. **Encryption**: TLS encrypts the data sent between parties, making it unreadable to anyone who intercepts it. This ensures confidentiality.
2. **Authentication**: It verifies the identity of the parties involved in the communication (like websites or servers), ensuring that you’re communicating with the intended entity and not an imposter.
3. **Data Integrity**: TLS uses hash functions to ensure that the data has not been altered in transit.
4. **Key Exchange**: It allows the secure exchange of keys between parties so that they can encrypt and decrypt messages safely.

Linux In-Kernel TLS (KTLS) refers to the implementation of Transport Layer Security (TLS) directly within the Linux kernel, rather than relying on user-space libraries like OpenSSL to handle the TLS encryption and decryption. This allows TLS processing to happen more efficiently and at a lower level in the system, reducing the overhead associated with data transmission and improving performance. Currently, KTLS primarily supports securing TCP connection and is based on TLS version 1.3.

### Linux NVMe-TLS

Linux NVMe-TLS leverages KTLS to secure NVMe over TCP by encrypting data transmitted via the NVMe-TCP protocol, delivering NVMe-TLS functionality. KTLS manages encryption, while NVMe-TCP handles the underlying transport. Currently, NVMe-TLS can only be used for NVMe-TCP and can not be used for other protocols like NVMe-FC or RDMA based configurations.

NVMe-TLS uses pre-shared keys (PSKs) that require a NVMe host and target to be configured with the same PSK in order to communicate over TLS. A PSK is a shared secret which was previously shared between the two parties. NVMe-TLS defines TLS Concatenation that uses NVMe In-Band Authentication to generate a PSK so PSKs don't have to be "manually" created and shared, but that support has recently been added to the upstream kernel, but is not currently available in UEK8.

The Oracle UEK8 kernel by default supports NVMe-TLS, but if using a non-UEK8 (v6.12 or greater) kernel, the kernel requires CONFIG_NVME_TCP_TLS and CONFIG_NVME_TARGET_TCP_TLS to be enabled when built.

### User Space Tools

* **nvme-cli**: The NVMe Command Line Interface is the primary tool for managing NVMe-TLS. Commands like `nvme gen-tls-key` and `nvme connect --tls` are used to create keys and establish TLS-secured connections.
* **tlshd Daemon**: The TLS handshake daemon (`tlshd`) handles the TLS handshake in user space, passing control to kTLS once complete. It requires configuration in `/etc/tlshd.conf`. To provide that capability, the ktls-utils package should be installed or built. See https://github.com/oracle/ktls-utils/blob/main/INSTALL. This package provides a TLS handshake user agent that listens for kernel requests and then materializes a user space socket endpoint on which to perform these handshakes. The resulting negotiated session parameters are passed back to the kernel via standard KTLS socket options.

### Trying out NVMe-TLS on a Loopback Connection

#### Setting Up a NVMe-TCP Target to use TLS

Setting up a NVMe-TLS target is very similar to setting up a NVMe-TCP target. NVMe-TLS introduces a new port attribute 'addr_tsas', when set to "tls1.3" will require a connecting host to undergo a TLS Handshake.

For this loopback example we will use "127.0.0.1" for the loopback target address 'add_traddr'. For more information on setting up a NVMe-TCP Target see the blog - NVMe over TCP \[[4](https://blogs.oracle.com/linux/post/nvme-over-tcp)\].

```
# modprobe nvmet-tcp
# mkdir /sys/kernel/config/nvmet/ports/10
# echo -n "127.0.0.1" > /sys/kernel/config/nvmet/ports/10/addr_traddr
# echo -n ipv4 > /sys/kernel/config/nvmet/ports/10/addr_adrfam
# echo -n tcp > /sys/kernel/config/nvmet/ports/10/addr_trtype
# echo -n 4420 > /sys/kernel/config/nvmet/ports/10/addr_trsvcid
# echo tls1.3 > /sys/kernel/config/nvmet/ports/10/addr_tsas # <===== NEW
# mkdir /sys/kernel/config/nvmet/subsystems/nqn.test
# echo 1 > /sys/kernel/config/nvmet/subsystems/nqn.test/attr_allow_any_host
# mkdir /sys/kernel/config/nvmet/subsystems/nqn.test/namespaces/1
# echo "/dev/nvme0n1" > /sys/kernel/config/nvmet/subsystems/nqn.test/namespaces/1/device_path
# echo 1 > /sys/kernel/config/nvmet/subsystems/nqn.test/namespaces/1/enable
# ln -s /sys/kernel/config/nvmet/subsystems/nqn.test /sys/kernel/config/nvmet/ports/10/subsystems/
```

### Connecting to a NVMe-TLS Target

**Setting up a .nvme keyring**

The .nvme keyring stores the NVMe-TLS PSKs. Add "keyrings=.nvme" to the \[authenticate\] section in /etc/tlshd.conf .section:

```
[authenticate]
keyrings=.nvme
```

**Creating Pre-Shared Keys (PSKs)**

For each subsystem, a PSK needs to be created and inserted into the .nvme keyring for both the target and host. A separate PSK should also be created and inserted for discovery. For this loopback example, since the target and host are the same system, the PSKs only need to be inserted once otherwise they need to be inserted on both the target and host systems.

```
# modprobe nvme-tcp
# nvme gen-tls-key --subsysnqn=nqn.test
NVMeTLSkey-1:01:LFKHx3GLh/WxQ3Qix/hiXhgJO/Ixkvwbye6I1Ys3uH5JBiAx:
# nvme gen-tls-key --subsysnqn=nqn.2014-08.org.nvmexpress.discovery
NVMeTLSkey-1:01:ajd8YaWR461D0eOU53vfKuI+sPtTZ8Fi1GLeXh7f/ZgMPniF:
```

We now have two PSKs, one for the NVMe subsystem, and one for discovery. These PSKs need to be "manually" shared to the target and host. The NVMe Specification does not define how the sharing is done.

**Inserting PSKs into the .nvme keyring and starting the TLS Handshake Deamon**

```
# nvme check-tls-key --subsysnqn=nqn.test -i \
-d NVMeTLSkey-1:01:LFKHx3GLh/WxQ3Qix/hiXhgJO/Ixkvwbye6I1Ys3uH5JBiAx:
Inserted TLS key 0d732a6a
# nvme check-tls-key --subsysnqn=nqn.2014-08.org.nvmexpress.discovery -i \
-d NVMeTLSkey-1:01:ajd8YaWR461D0eOU53vfKuI+sPtTZ8Fi1GLeXh7f/ZgMPniF:
Inserted TLS key 059aaa37
# systemctl start tlshd.service
# keyctl show
Session Keyring
 828015744 --alswrv      0     0  keyring: _ses
 983090618 --alswrv      0 65534   \_ keyring: _uid.0
  38284286 ---lswrv      0     0   \_ keyring: .nvme
 225651306 --als-rv      0     0       \_ psk: NVMe0R01 nqn.2014-08.org.nvmexpress:uuid:00000000-0000-0000-0000-000000000000 nqn.test
  94022199 --als-rv      0     0       \_ psk: NVMe0R01 nqn.2014-08.org.nvmexpress:uuid:00000000-0000-0000-0000-000000000000 nqn.2014-08.org.nvmexpress.discovery
# 
```

**Discovering and Connecting to Targets**

```
# nvme discover -t tcp -a 127.0.0.1 -s 4420 --tls

Discovery Log Number of Records 2, Generation counter 2
=====Discovery Log Entry 0======
trtype:  tcp
adrfam:  ipv4
subtype: current discovery subsystem
treq:    required, sq flow control disable supported
portid:  10
trsvcid: 4420
subnqn:  nqn.2014-08.org.nvmexpress.discovery
traddr:  127.0.0.1
eflags:  none
sectype: tls13
=====Discovery Log Entry 1======
trtype:  tcp
adrfam:  ipv4
subtype: nvme subsystem
treq:    required, sq flow control disable supported
portid:  10
trsvcid: 4420
subnqn:  nqn.test
traddr:  127.0.0.1
eflags:  none
sectype: tls13
# nvme connect -t tcp -a 127.0.0.1 -s 4420 -n  nqn.test --tls
connecting to device: nvme1
# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   40G  0 disk 
├─sda1        8:1    0    1G  0 part /boot
└─sda2        8:2    0   39G  0 part 
  └─ol-root 252:0    0   39G  0 lvm  /
sr0          11:0    1 1024M  0 rom  
nvme1n1     259:0    0    1G  0 disk 
nvme0n1     259:1    0    1G  0 disk 
```

### In Summary

In conclusion, NVMe-TLS provides a robust and efficient solution for securing NVMe over TCP connections, leveraging the Linux kernel's KTLS to deliver encryption, authentication, and data integrity over standard TCP/IP networks. By integrating with existing Ethernet infrastructure, NVMe-TLS simplifies deployment compared to traditional RDMA or Fibre Channel setups, while maintaining high security standards through pre-shared keys (PSKs) and tools like nvme-cli and tlshd. The loopback example demonstrates the straightforward configuration process, making NVMe-TLS an accessible option for securing remote storage access. As support for features like TLS Concatenation with NVMe In-Band Authentication continues to evolve, NVMe-TLS is poised to become a cornerstone for secure, high-performance storage solutions in modern data centers.

### Reference

\[1\] NVM Express Base Specification, Revision 2.0c, October 4, 2022 - https://nvmexpress.org/specifications/

\[2\] NVMe In-band Authentication, Alan Adamson - https://blogs.oracle.com/linux/post/nvme-inband-authentication

\[3\] Oracle Linux Now Includes the Latest Linux Kernel with UEK 8, Gursewak Sokhi - https://blogs.oracle.com/linux/post/oracle-linux-now-includes-the-latest-linux-kernel-with-uek-8

\[4\] NVMe over TCP, Alan Adamson - https://blogs.oracle.com/linux/post/nvme-over-tcp
