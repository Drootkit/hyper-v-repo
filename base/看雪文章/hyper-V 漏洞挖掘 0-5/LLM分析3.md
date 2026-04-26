## 文章总结：Hyper-V安全从0到1(3)

### 基础信息

| 项目 | 内容 |
|------|------|
| **文章标题** | Hyper-V安全从0到1(3) |
| **作者/来源** | 看雪安全社区 / ifyou |
| **发布时间** | 2017-11-9 |
| **技术领域** | Hyper-V虚拟化、VMBus、数据传输、Linux内核驱动 |
| **原文链接** | https://bbs.kanxue.com/thread-222656.htm |

---

### 总体摘要

本文是Hyper-V安全研究系列的第三篇，**核心聚焦于虚拟机与宿主机之间的数据传输机制**。文章从攻击面分析的角度出发，详细剖析了数据从虚拟机内核层→Hypervisor层→Windows内核层→Windows应用层的完整流向。

**关键要点**：
- 数据传输通过**VMBus**和**环形内存(buffer_ring)**实现
- 虚拟机发送数据使用**Hypercall**通知宿主机；接收数据通过**中断机制**触发回调
- Windows内核层是主要攻击面，涉及`vmbusr`和`vmbkmclr`模块的数据分发
- 用户态通过**命名管道**与内核通信

---

### 核心内容详解

#### 1. 数据传输流程概览

数据流向框架：
- **虚拟机→宿主机**：数据填入`buffer_ring` → VMBus调用Hypercall → Hypervisor触发VM-Exit → 宿主机处理
- **宿主机→虚拟机**：宿主机写入`buffer_ring` → Hypervisor触发虚拟机中断 → 中断处理程序调用回调函数 → 虚拟机读取数据

#### 2. 虚拟机接收数据流程（Linux内核）

```
中断触发 → vmbus_isr() → tasklet_schedule(hv_context.event_dpc) 
→ vmbus_on_event() → process_chn_event() 
→ channel->onchannel_callback() [即synthvid_receive()]
→ vmbus_recvpacket() → hv_ringbuffer_read()
```

以`hyperv_fb`（虚拟显示设备）驱动为例，展示了从注册回调函数`synthvid_receive`到实际读取数据的完整调用链。

#### 3. 虚拟机发送数据流程

```
synthvid_send() → vmbus_sendpacket() → vmbus_sendpacket_ctl()
→ hv_ringbuffer_write() [写入环形缓冲区]
→ vmbus_setevent() [通知宿主机]
```

#### 4. 通知宿主机的两种方式

| 方式 | 实现机制 | 适用设备 |
|------|----------|----------|
| **monitorpage** | 在共享内存页的`trigger_group.pending`置位 | 网卡、硬盘设备 |
| **hypercall** | 执行`vmcall`指令触发VM-Exit | 集成服务、键盘、鼠标、动态内存、视频设备 |

**hypercall实现细节**：
- `hv_do_hypercall()`通过内联汇编调用`hypercall_page`
- `hypercall_page`由MSR寄存器`HV_X64_MSR_HYPERCALL`初始化
- 页面内容包含`vmcall`指令（0x0f01c1），执行后陷入Hypervisor层

---

### 安全研究价值

1. **攻击面识别**：Windows内核层的`vmbusr`和`vmbkmclr`模块解析虚拟机数据，是潜在的漏洞挖掘目标
2. **逆向辅助**：理解数据流向有助于定位关键函数和结构体成员
3. **漏洞复现**：掌握通信机制有助于构造PoC和触发漏洞条件
