KVM虚拟化



## KVM

Kernel-based Virtual Machine (KVM) ：基于内核的虚拟机。在 Linux 2.6.20 版本中并入主线代码。

KVM组成：`kvm.ko`（核心模块，负责提供 CPU 和内存的虚拟化功能）`kvm-intel.ko` `kvm-amd.ko`（针对特定CPU虚拟化提供支持）`/dev/kvm`（一个字符设备，用户态程序如 QEMU，通过它与 KVM 通信）

```c
// 打开 /dev/kvm
int kvm_fd = open("/dev/kvm", O_RDWR);
// 创建虚拟机 （KVM 内核模块创建一个 VM 实例）
int vm_fd = ioctl(kvm_fd, KVM_CREATE_VM, 0);
// 创建 vCPU （分配 VMCS（VT-x）或 VMCB（AMD-V）结构。）
int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, 0);
// 进入客户机模式（触发 VM Entry，客户机开始运行。）
struct kvm_run *run = mmap(..., vcpu_fd, ...);
ioctl(vcpu_fd, KVM_RUN, NULL);
/**
 *  当客户机访问 I/O、网络、磁盘时，触发 VM Exit；
 *  CPU 跳回 Root Mode，执行 KVM 代码；
 *  KVM 通知 QEMU（通过 kvm_run->exit_reason）；
 *  QEMU 模拟设备行为（如读磁盘、发网络包）；
 *  处理完成后，再次 KVM_RUN，恢复客户机。
 */
```

内存虚拟化支持：EPT（Intel），NPT（AMD）KVM 设置 EPT/NPT 表；MMU 硬件自动完成转换；极大减少 VM Exit 次数，提升性能。





## QEMU

上述KVM基于硬件级别的支持，提供了相同指令架构下高效的CPU和内存虚拟化。如果需要模拟一个完整的虚拟机，还需对硬盘、网卡、显卡等外部IO设备进行虚拟化。这一层功能由 QEMU 提供。

QEMU 根据目标架构提供多个系统模拟程序，例如 qemu-system-x86_64、qemu-system-aarch64 等。QEMU可以在用户态模拟一个完整的虚拟机，并可以使用TCG模式进行跨架构模拟（Type-2型虚拟机），这种用户态纯软件模拟的性能很差。在KVM的支持下，QEMU仅需负责设备IO的模拟，将CPU和内存提供给KVM模拟，虽然只能模拟同架构的虚拟机，但性能接近原生执行效率（QEMU/KVM是Type-1型虚拟机）。

Virtio是**半虚拟化（Para-virtualization）**设备接口规范，用于高效的模拟IO设备。Virtio前端可以视为在客户机操作系统运行的驱动程序，后端运行在宿主机的QEMU进程中。通过共享内存队列机制，Virtio 减少了传统**全虚拟化**设备因频繁触发 VM Exit 而带来的性能损耗，实现了高效的设备通信。

> 启用 KVM 加速只需在 QEMU 命令行中添加 `-accel kvm` 参数。一旦开启，开发者和用户可无需关注底层虚拟化机制，除非需要配置高级功能，例如：**嵌套虚拟化**、**大页内存支持** 或 **实时迁移** 等特性。

> 全虚拟化：GuestOS不不知道虚拟机的存在，使用为真实硬件设计的驱动程序，QEMU需要完全模拟一个硬件设备。半虚拟化：GuestOS直到虚拟机的存在，主动使用特殊的半虚拟化PV驱动程序。
>





## libvirt

上述两种技术结合的虚拟化技术被称为 `QEMU/KVM` 。libvirt是红帽RHEL推广的一套工具集，用于管理虚拟机，可以视为 `QEMU/KVM` 的前端，底层也是调用QEMU命令程序来管理虚拟机的。

主要包含两个命令：`virt-install`：创建虚拟机。`virsh`：管理虚拟机。

启动虚拟机（virsh start）关闭虚拟机（virsh shutdown）删除虚拟机（virsh undefine）

> 除了 QEMU/KVM ，libvirt 工具链也支持其他虚拟化技术作为后端，比如Xen，LXC。

[配置和管理 Linux 虚拟机 | Red Hat Enterprise Linux](https://docs.redhat.com/zh-cn/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_linux_virtual_machines/index)

[KVM介绍 | Red Hat Enterprise Linux](https://www.redhat.com/en/topics/virtualization/what-is-KVM)





## 附录：KVM技术与CPU虚拟化

在现代CPU架构中，一般提供了四种特权级别，操作系统OS运行在 Ring 0 最高权限，用户态程序运行在 Ring 3 普通权限，其余两个基本被废弃了。

2005年左右，英特尔在 Pentium 4 处理器中推出了 Intel VT-x ，引入了两个模式，Root 模式拥有整个系统的最终控制权（用于运行Hypervisor），Non-Root 模式则替代了以前的 Ring 四种模式（用于运行Guest OS）。可以在默认的Ring 0级别使用VMXON指令进入 Root 模式。

```txt
物理 CPU
│
├── VMX Root Operation (KVM / Hypervisor)
│   └── 拥有对整个系统的最终控制权（可以创建 VM、管理内存、处理中断）
│
└── VMX Non-Root Operation (Guest OS)
    ├── Guest Ring 0 → 实际运行在 Non-Root 模式下
    ├── Guest Ring 1 → 同上
    ├── Guest Ring 2 → 同上
    └── Guest Ring 3 → 同上
```

当 Guest OS 在 Non-Root 模式下执行敏感指令（如写 CR3、发 I/O 指令）时，CPU 自动触发 VM Exit，切换回 Root 模式，把控制权交给 KVM。KVM 处理完后，通过 VM Entry 返回 Guest。对 Guest OS 来说，它仍然认为自己运行在 Ring 0，一切正常。

在加载KVM模块后，Linux内核就会进入 Intel VT-x 的 Root 模式，此时不再是普通的操作系统，而是 Hypervisor （负责管理所有其他虚拟机），并且保留了 Host OS 的身份（负责运行宿主机用户态程序）。



2008年左右，英特尔在 Nehalem架构 处理器中推出了 Intel EPT，主要用于内存的虚拟化。

虚拟机的内存翻译主要有两步，第一步，Guest OS（客户机虚拟地址 -> 客户机真实地址），Hypervisor（客户机真实地址 -> 主机真实地址），在EPT出现前主要通过Hypervisor维护一个**影子页表**来实现Hypervisor步骤的映射，每次客户机修改页表操作都需要频繁的 VM Exit 维护影子页表。Intel EPT 是 Intel VT-x 的扩展功能，KVM直接创建硬件级别的EPT页表（Extended Page Tables）。让CPU硬件自动完成了Hypervisor这一步骤的翻译，无需KVM的频繁干预了。





## 附录：Docker 虚拟化技术

Docker 虽然并非传统意义上的虚拟机，但它通过容器技术实现了用户态进程的虚拟化（进程隔离）。每个容器共享宿主机的操作系统内核，相较于传统虚拟机，资源开销更小、启动更快，因而能够高效地解决环境配置不一致等问题，已成为许多传统虚拟机使用场景的理想替代方案。

每个容器都共享宿主机的操作系统内核，这是其与传统 QEMU/KVM 虚拟机最根本的区别。通过共享内核，Docker 在用户态利用 Namespaces 实现进程、网络、文件系统等环境的隔离，并借助 cgroups 进行资源限制与分配，从而在保证轻量高效的同时，实现了各容器之间的完全隔离和精细化的资源管理。

> 在 Docker 容器中执行 `uname` 命令显示的是宿主机的内核版本。这意味着，即使宿主机运行的是 Ubuntu 20（基于 Linux 5.4 内核），而容器内运行的是 Ubuntu 24（基于 Linux 6.8 内核），容器实际使用的仍是宿主机的旧版内核。容器中的 Ubuntu 24 仅提供了更新的用户态程序和库，这些程序通过系统调用与宿主机的旧内核交互。因此，如果这些程序需要 Linux 6.8 内核才引入的新功能，就可能因内核不支持而失败。

**Linux Namespaces（命名空间）**：大致分为六个命名空间，主要用于进程的隔离。

- Mount（2002年引入，用于实现挂载方面的隔离）

- UTS（用于实现独立的 hostname 和 domain name）

- IPC（用于实现独立的进程间交互IPC）

- PID（用于实现独立的进程PID）

- Network（用于实现独立的网络）

- User（2013年引入，用于实现独立的用户和用户组）

**CGroups（Control Group 控制组）**：命名空间只能进行进程间的隔离，CGroups的作用是资源限制与分配，可以防止容器过度占用CPU、内存等物理资源，确保系统资源的合理利用和多容器环境的稳定运行。

**UnionFS（联合文件系统）**：是一种支持分层结构的虚拟文件系统，也是 Docker 镜像技术的核心基础。Docker 使用的 AUFS（Advanced UnionFS）正是基于 UnionFS 发展而来的一种改进和增强版本。

> LXC（Linux Containers）是一种基于 Linux 内核的 Namespaces 和 cgroups 实现的轻量级操作系统虚拟化技术，通过这些内核特性提供了进程、资源和环境的隔离。它是 Docker 最直接的灵感来源。Docker 使用 Go 语言对 LXC 进行了封装，大幅提升了易用性，并创新性地引入了基于 UnionFS 的分层镜像系统，彻底改变了应用打包与分发的方式。
>
> 在早期版本中，Docker 确实使用 LXC 作为默认的容器运行时，但随后开发了更轻量、更安全的 libcontainer（后演变为符合标准的 runc），逐步取代了 LXC。此后，为了统一容器生态，行业成立了开放容器倡议（OCI），制定了容器运行时（runtime）和镜像格式的标准，最终确立了以 runc 为代表的标准化容器运行时架构，为 Kubernetes 等编排系统的发展奠定了坚实基础。







END