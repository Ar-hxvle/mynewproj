----------------------+ |     NFC应用层         | +-----------------------+ |     NFC协议栈         |  +-----------------------+ |     GKI抽象层         |   +-----------------------+ |     HAL硬件抽象层      | +-----------------------+ |     物理硬件层   从上面几层介绍nfc的开关打开关闭流程









已深度思考（用时 39 秒）



我们有一个分层的NFC架构，从应用层到物理硬件层。现在需要介绍NFC开关（打开和关闭）的流程，流程会涉及每一层。

 根据分层结构，从上到下依次是：

1. NFC应用层
2. NFC协议栈
3. GKI抽象层
4. HAL硬件抽象层
5. 物理硬件层

 开关流程应该是：

  打开流程：从应用层发起，逐层向下直到物理硬件层。

  关闭流程：同样从应用层发起，逐层向下。

 我们分别描述打开和关闭流程，每层需要做什么。

 注意：GKI（Generic Kernel Interface）是一个通用的内核接口，用于任务调度、时间管理、缓冲区管理等。在NFC协议栈中，GKI层通常用于提供操作系统抽象，以便协议栈可以在不同的操作系统上运行。

 假设我们有一个典型的NFC开关控制流程：

 打开流程（Enable NFC）：

1. 应用层：用户通过设置界面等触发打开NFC，调用NFC API（如enable()）。
2. NFC协议栈：接收到使能请求，进行协议栈的初始化，包括初始化各个模块（如LLCP、NDEF、RF等），然后通过GKI层向下发送初始化命令。
3. GKI抽象层：主要负责协议栈与底层之间的通信，这里可能将初始化命令传递给HAL层，同时管理任务调度等。
4. HAL硬件抽象层：接收到初始化命令后，进行硬件相关的初始化，比如配置NFC控制器（如通过I2C等接口发送初始化指令），设置工作模式等。
5. 物理硬件层：NFC芯片上电，初始化射频电路，开始侦听射频场。

 关闭流程（Disable NFC）：

1. 应用层：用户触发关闭NFC，调用disable()。
2. NFC协议栈：接收到关闭请求，开始释放协议栈资源（如停止所有NFC活动，关闭RF发现等），然后通知下层进行关闭。
3. GKI抽象层：传递关闭命令给HAL层。
4. HAL硬件抽象层：执行硬件关闭操作，比如让NFC控制器进入低功耗模式或完全断电。
5. 物理硬件层：NFC芯片断电，射频电路关闭。

 然而，实际的流程可能更复杂，比如在打开过程中可能涉及固件加载等步骤，但基本流程如此。

 下面详细描述每一层在开关过程中的主要职责：

 打开流程：

1. 应用层：调用NFC服务的enable接口。
2. NFC协议栈：

​        \- 初始化内部状态机、数据结构、事件处理等。

​        \- 加载配置参数。

​        \- 通过GKI发送初始化命令给HAL层。

3. GKI抽象层：将来自协议栈的命令放入HAL层的任务队列（如果使用任务机制），或者直接调用HAL层的接口。
4. HAL硬件抽象层：

​        \- 初始化与NFC控制器的通信接口（如I2C、SPI、UART等）。

​        \- 下载固件到NFC控制器（如果需要）。

​        \- 配置NFC控制器的工作参数（射频参数、协议参数等）。

​        \- 给NFC控制器上电，并启动射频场（如果需要立即进入发现模式）。

5. 物理硬件层：NFC控制器上电，初始化射频电路，开始产生射频场（如果是主动模式）或者侦听射频场（被动模式）。

 关闭流程：

1. 应用层：调用NFC服务的disable接口。
2. NFC协议栈：

​        \- 停止发现流程（如果正在运行）。

​        \- 断开所有已连接的设备。

​        \- 释放协议栈资源（如内存、定时器等）。

​        \- 通过GKI发送关闭命令给HAL层。

3. GKI抽象层：传递关闭命令给HAL层。
4. HAL硬件抽象层：

​        \- 发送关闭命令给NFC控制器，使其停止射频活动。

​        \- 将NFC控制器设置为低功耗模式或完全断电。

​        \- 关闭与NFC控制器的通信接口（可选）。

5. 物理硬件层：NFC控制器断电，射频电路关闭，不再产生或侦听射频场。

 注意：在关闭过程中，协议栈可能需要处理一些清理工作，比如通知上层应用NFC已被关闭，保存状态等。

 以上就是一个大致的流程。实际实现中，每一层可能有更详细的操作，但分层的思想就是每一层只关注自己的职责，下层为上层提供服务。

### NFC开关流程详解（从应用层到物理硬件层）

以下是NFC开关（打开/关闭）的完整流程，按分层架构逐层说明：

------

#### **1. NFC应用层**

- **打开流程**：
  1. 用户触发操作（如点击系统设置中的"NFC开关"）。
  2. 应用层调用系统服务接口（如Android的`NfcAdapter.enable()`）。
  3. 初始化NFC服务配置（读写模式、HCE设置等）。
- **关闭流程**：
  1. 用户关闭NFC开关。
  2. 应用层调用`NfcAdapter.disable()`。
  3. 通知所有NFC相关应用停止活动（如支付APP释放资源）。

------

#### **2. NFC协议栈**

- **打开流程**：
  1. 初始化协议栈核心模块（LLCP、NDEF、ISO-DEP等）。
  2. 启动发现流程（Discovery）：
     - 配置侦听模式（Tag读写、P2P、HCE）。
     - 激活RF发现任务（轮询不同类型标签）。
  3. 分配通信资源（如RF通道、缓冲区）。
- **关闭流程**：
  1. 停止发现流程，结束所有RF轮询。
  2. 释放协议栈资源（断开活跃连接、清空缓冲区）。
  3. 注销所有已注册的AID（HCE应用）。

------

#### **3. GKI抽象层（Generic Kernel Interface）**

- **打开流程**：
  1. 初始化GKI任务调度器（创建NFC主任务线程）。
  2. 启动定时器管理（用于RF轮询、超时控制）。
  3. 为协议栈分配内存池（动态内存管理）。
- **关闭流程**：
  1. 停止所有GKI定时器。
  2. 销毁NFC相关任务线程。
  3. 释放GKI内存池。

------

#### **4. HAL硬件抽象层**

- **打开流程**：
  1. 加载NFC控制器固件（通过I²C/SPI写入芯片）。
  2. 配置硬件参数：
     - 设置射频模式（NFC-A/B/F、ISO 14443/15693）。
     - 调整RF场强度（输出功率校准）。
  3. 发送**预初始化命令序列**（如PN51x的`CORE_RESET_CMD` + `CORE_INIT_CMD`）。
  4. 使能RF天线电路（激活天线供电）。
- **关闭流程**：
  1. 发送**硬件休眠命令**（如`CORE_RESET_CMD`进入Sleep模式）。
  2. 关闭RF天线电源。
  3. 复位寄存器状态（清理硬件缓存）。

------

#### **5. 物理硬件层**

- **打开流程**：
  1. 主控通过总线（I²C/SPI）唤醒NFC芯片。
  2. 电源管理IC（PMIC）为RF天线供电。
  3. 射频电路启动：
     - 13.56MHz振荡器工作。
     - 调制解调器激活，产生电磁场。
  4. 芯片进入**侦听/发射状态**（检测场强、接收调制信号）。
- **关闭流程**：
  1. 射频电路断电（振荡器停振）。
  2. 天线回路断开（场强消失）。
  3. 芯片进入低功耗模式（电流降至μA级）。

------

### **完整流程示例（打开NFC）**

plaintext



复制



下载

```
用户点击开关 → 应用层调用enable() → 协议栈启动发现流程 → GKI初始化任务调度 → 
HAL加载固件+配置RF参数 → 物理层上电并激活射频场
```

### **关键注意事项**

- **错误处理**：任一层的失败（如HAL固件加载失败）会逐层向上返回错误码（如`NFC_STATUS_FAILED`）。
- **功耗控制**：关闭时物理层必须彻底断电，否则导致待机漏电。
- **并发安全**：协议栈需处理"关闭过程中收到标签"的冲突（强制终止交互）。

通过这种分层设计，NFC开关流程实现了**高内聚低耦合**，确保硬件操作与上层逻辑解耦，提升系统稳定性和可移植性。









结合底层源码，分析哈每一个关键的函数，写出一篇nfc开关打开流程的文档









已深度思考（用时 73 秒）



结合底层源码分析NFC开关打开流程（以Android开源项目AOSP的NFC栈为例）

本文将基于AOSP的NFC实现（如NXP或Broadcom的HAL），从应用层到硬件层逐层分析NFC打开流程的关键函数。源码路径以`packages/apps/Nfc`和`hardware/libhardware/include/hardware/nfc.h`等为主。

\---

\### **1. 应用层（NfcService）**

**关键函数**：`NfcAdapter.enable()`  

\- **调用路径**：  

  `NfcAdapter.enable()` → `NfcAdapterService.enable()` → `NfcService.enableInternal()`  

\- **源码分析**：  

  \```java

  // packages/apps/Nfc/src/com/android/nfc/NfcService.java

  void enableInternal() {

​      synchronized (this) {

​          if (mState == NfcAdapter.STATE_ON) return; // 已开启则跳过

​          mDeviceHost.initialize(); // 初始化硬件（向下调用JNI）

​          startRouting();           // 启动路由

​          mState = NfcAdapter.STATE_ON;

​          // 广播状态变化：NfcAdapter.ACTION_ADAPTER_STATE_CHANGED

​      }

  }

  \```

  **作用**：状态检查 → 触发硬件初始化 → 启动协议栈路由。

\---

\### **2. NFC协议栈（libnfc-nci）**

**关键函数**：`nfaEnable()`  

\- **调用路径**：  

  `NfcService.enableInternal()` → `NativeNfcManager.initialize()` (JNI) → `nfaEnable()`  

\- **源码分析**（C++层）：  

  \```cpp

  // system/nfc/src/nfa/nfa_api.cc

  tNFA_STATUS nfaEnable() {

​      nfa_sys_register();        // 注册NFA子系统

​      nfa_dm_start();            // 启动设备管理模块

​      nfa_ee_start();            // 启动执行环境（SE安全元件）

​      nfa_p2p_start();           // 启动P2P模块

​      // ... 其他模块初始化

​      nfa_dm_enable();           // 核心：使能设备管理

​      return NFA_STATUS_OK;

  }

  \```

  **关键子函数** `nfa_dm_enable()`：  

  \```cpp

  // system/nfc/src/nfa/dm/nfa_dm_main.cc

  void nfa_dm_enable() {

​      nfa_sys_enable_subsystems(); // 激活所有子系统

​      nfa_dm_discover_start();     // 启动发现流程（RF轮询）

  }

  \```

\---

\### **3. GKI抽象层（Generic Kernel Interface）**

**关键函数**：`GKI_init()` 和 `GKI_create_task()`  

\- **调用路径**：  

  `nfa_sys_register()` → `GKI_init()` → `GKI_create_task()`  

\- **源码分析**：  

  \```cpp

  // system/nfc/gki/common/gki.cc

  void GKI_init() {

​      pthread_mutex_init(&gki_cb.mutex, NULL); // 初始化互斥锁

​      // 创建定时器线程

​      pthread_create(&timer_thread, NULL, gki_timer_thread, NULL);

  }

  \```

  **作用**：创建NFC任务线程（如`nfa_sys_thread`）和定时器，为协议栈提供并发基础。

\---

\### **4. HAL硬件抽象层（Nfc Hardware Abstraction Layer）**

**关键函数**：`open()` 和 `coreInitialization()`  

\- **调用路径**：  

  `NfcService.enableInternal()` → `DeviceHost.initialize()` → `HalDeviceContext.open()`  

\- **源码分析**（HAL接口）：  

  \```cpp

  // hardware/libhardware/include/hardware/nfc.h

  struct nfc_nci_device {

​      int (*open)(struct nfc_nci_device* dev, nfc_event_callback_t event_cb, 

​                  nfc_data_callback_t data_cb); // 打开设备

  };

  \```

  **实现示例**（NXP PN5xx系列）：  

  \```cpp

  // vendor/nxp/opensource/libnfc-nci/src/hal/pn5xx/pn54x.cc

  static int pn54x_open() {

​      open_hw_device();         // 打开/dev/pn54x设备节点

​      nfc_start_initialize();   // 启动初始化序列

​      // ...

  }

  \```

  **核心初始化序列**：  

1. 发送复位命令：`NFA_ResetNFCC() → phNxpNciHal_core_initialized()`  
2. 配置射频参数：`phNxpNciHal_send_ext_cmd()` 发送自定义RF设置（如CE模式、低功耗模式）

\---

\### **5. 物理硬件层（驱动与芯片）**

**关键操作**：  

\- **Linux驱动**：  

  \```c

  // kernel/drivers/nfc/pn54x/pn54x.c

  static int pn54x_dev_open(struct inode *inode, struct file *filp) {

​      pn54x_enable_irq();        // 使能中断

​      pn54x_power_on();          // 芯片上电（拉高VDD）

  }

  \```

\- **芯片初始化序列**（通过I²C/SPI）：  

1. 复位芯片：`0x20 0x00 0x01 0x01`（CORE_RESET_CMD）  
2. 读取版本：`0x00 0x01 0x02`（CORE_INIT_CMD）  
3. 配置RF寄存器：`0x2F 0x02 0x00 0x01`（RF_SET_LISTEN_MODE_ROUTING）  

\---

\### **完整函数调用链（打开流程）**

\```plaintext

Java层:

  NfcService.enableInternal()

​    → NativeNfcManager.initialize() [JNI]

JNI层 (com_android_nfc_NativeNfcManager.cpp):

  nfcManager_doInitialize()

​    → nfaEnable()

NFA层 (nfa_api.cc):

  nfaEnable()

​    → nfa_sys_register()       // 注册子系统

​    → nfa_dm_enable()          // 启动设备管理

​      → nfa_dm_discover_start() // 启动发现流程

HAL层 (pn54x.cc):

  phNxpNciHal_initialize()

​    → phNxpNciHal_core_initialized()

​      → phNxpNciHal_send_ext_cmd() // 发送CORE_RESET_CMD/CORE_INIT_CMD

驱动层 (pn54x.c):

  pn54x_dev_write()           // 将HAL命令写入I²C

​    → i2c_master_send()       // Linux I²C驱动传输

\```

\---

\### **关键日志与调试**

\- **开启NFC的logcat过滤**：  

  \```bash

  adb logcat | grep -E "NfcService|NxpNciHal|libnfc"

  \```

\- **典型成功日志**：  

  \```

  NfcService: Enabling NFC

  NxpNciHal: phNxpNciHal_init: enter

  libnfc  : nfa_dm_discover_start: state=0

  NxpNciHal: Write successful (4 bytes): 20 00 01 01

  NxpNciHal: Response: 40 00 01 01

  NfcService: NFC is ON

  \```

\---

\### **异常处理流程**

\- **固件加载失败**：  

  HAL层检测到超时或无响应 → 返回`NFC_STATUS_FAILED` → JNI层抛异常 → 应用层显示"无法打开NFC"。

\- **RF冲突**：  

  若打开时检测到已有激活的标签，协议栈会强制发送`Deselect`命令终止交互。

\---

通过源码分析可见，NFC打开流程是典型的**跨层协作**：  

1. **应用层**处理用户意图和状态广播。  
2. **协议栈**管理多模块协同与RF发现。  
3. **HAL层**屏蔽硬件差异，实现标准化命令序列。  
4. **驱动层**直接操纵硬件寄存器与总线。  

这种分层设计使得更换NFC芯片（如NXP换成ST）时，只需重写HAL层实现，无需修改上层逻辑。

## NFC开关打开流程源码级分析（基于Android NFC协议栈）

本文基于AOSP（Android Open Source Project）的NFC协议栈源码，结合Linux内核驱动，深入分析NFC打开流程的关键函数调用。源码路径以`/system/nfc/`和`/kernel/drivers/nfc/`为主。

------

### **1. 应用层（NfcService）**

**关键函数**：`NfcAdapter.enable()`
**调用链**：

java



复制



下载

```
// frameworks/base/core/java/android/nfc/NfcAdapter.java
public boolean enable() {
    try {
        return sService.enable(); // 调用NfcService
    } catch (RemoteException e) { ... }
}

// packages/apps/Nfc/src/com/android/nfc/NfcService.java
public void enable() {
    synchronized (this) {
        mDeviceHost.initialize(); // 关键：初始化硬件
    }
}
```

**关键操作**：

- 状态检查（确保NFC未开启）
- 调用`NativeNfcManager.initialize()`（JNI桥接）

------

### **2. NFC协议栈（libnfc-nci）**

**关键函数**：`nfaEnable()`
**调用链**（C++层）：

cpp



复制



下载

```
// system/nfc/src/nfa/nfa_api.cc
tNFA_STATUS NFA_Enable() {
    nfa_sys_register();     // 注册NFA子系统
    nfa_dm_start();         // 启动设备管理模块
    nfa_ee_start();         // 初始化执行环境（eSE）
    nfa_p2p_start();        // 启动P2P模块
    return nfa_dm_enable(); // 核心使能函数
}

// system/nfc/src/nfa/dm/nfa_dm_main.cc
tNFA_STATUS nfa_dm_enable() {
    nfa_sys_enable_subsystems();  // 激活子系统
    nfa_dm_discover_start();      // 启动发现流程
}
```

**关键操作**：

- 初始化LLCP（逻辑链路控制协议）
- 配置RF发现参数（`NFA_DM_DISC_PARAMS`）
- 启动定时器（`GKI_start_timer()`）用于RF轮询

------

### **3. GKI抽象层（任务调度）**

**关键函数**：`GKI_create_task()`
**源码实现**：

cpp



复制



下载

```
// system/nfc/gki/common/gki.cc
uint8_t GKI_create_task(task_entry r, uint8_t task_id, const char* name) {
    pthread_create(&thread, NULL, r, NULL); // 创建POSIX线程
}
```

**任务初始化**：

cpp



复制



下载

```
// system/nfc/src/nfa/sys/nfa_sys.c
void nfa_sys_register() {
    GKI_create_task(nfa_sys_thread, NFA_TASK, "NFA_TASK");
}
```

**作用**：创建NFC主任务线程，处理协议栈事件循环

------

### **4. HAL硬件抽象层（HAL实现）**

**关键函数**：`hal_open()` 和 `hal_core_initialized()`
**调用链**：

cpp



复制



下载

```
// hardware/nxp/nfc/halimpl/src/hal.c
int hal_open() {
    phNxpNciHal_open(); // 打开NCI控制器
}

// hardware/nxp/nfc/halimpl/src/phNxpNciHal.cc
int phNxpNciHal_open() {
    dev = open("/dev/pn54x", O_RDWR); // 打开Linux设备节点
    phTmlNfc_Init();                 // 传输层初始化
    phNxpNciHal_core_initialized();  // 核心初始化
}
```

**硬件初始化序列**：

cpp



复制



下载

```
// 发送复位命令序列
static void phNxpNciHal_reset() {
    static uint8_t cmd_reset[] = {0x20, 0x00, 0x01, 0x01}; // CORE_RESET_CMD
    write(dev, cmd_reset, sizeof(cmd_reset));
    
    // 等待复位完成响应
    read(dev, resp, sizeof(resp)); // 预期响应: 0x40 0x00 0x03 0x01 0xXX 0xXX
}
```

------

### **5. 物理硬件层（Linux驱动）**

**关键驱动函数**：`pn54x_dev_write()`
**驱动源码**：

c



复制



下载

```
// kernel/drivers/nfc/pn54x/pn54x.c
static ssize_t pn54x_dev_write(struct file *filp, const char __user *buf,
                               size_t count, loff_t *offset)
{
    struct pn54x_dev *dev = filp->private_data;
    
    // 将用户空间数据复制到内核
    copy_from_user(dev->write_buf, buf, count);
    
    // 通过I²C/SPI发送到NFC芯片
    i2c_master_send(dev->client, dev->write_buf, count);
}
```

**硬件操作序列**：

c



复制



下载

```
// 上电序列
static void pn54x_enable(struct pn54x_dev *dev) {
    gpio_set_value(dev->ven_gpio, 1);  // 使能芯片电源
    msleep(10);
    gpio_set_value(dev->firm_gpio, 0); // 加载主固件
    msleep(50);
    
    // 启动RF振荡器
    reg_write(dev->client, RF_REGISTER, 0x01); 
}
```

------

### **完整函数调用栈**

plaintext



复制



下载

```
1. Java: NfcAdapter.enable()
2. Java: NfcService.enableInternal()
3. JNI: NativeNfcManager_doInitialize()
4. C++: nfaEnable()
   ├─ nfa_sys_register()
   ├─ nfa_dm_start()
   └─ nfa_dm_enable()
       └─ nfa_dm_discover_start()
5. HAL: phNxpNciHal_open()
   ├─ open("/dev/pn54x")
   ├─ phTmlNfc_Init()
   └─ phNxpNciHal_core_initialized()
       ├─ phNxpNciHal_reset()      // 发送0x20000101
       └─ phNxpNciHal_init()       // 发送0x20000200
6. Kernel: pn54x_dev_write()
   ├─ i2c_master_send(CORE_RESET_CMD)
   └─ i2c_master_send(CORE_INIT_CMD)
7. Hardware: PN54X芯片
   ├─ 复位数字逻辑
   ├─ 加载固件
   └─ 激活RF场(13.56MHz)
```

------

### **关键调试日志**

在logcat中过滤`NfcService`和`NxpNciHal`：

log



复制



下载

```
// 成功打开流程
D/NfcService: enableInternal()
I/NxpNciHal: phNxpNciHal_open: Opening HAL layer
D/NxpNciHal: phNxpNciHal_core_initialized
I/NxpNciHal: Write: len=4 > 20 00 01 01  // CORE_RESET_CMD
I/NxpNciHal: Read: len=6  < 40 00 03 01 00 00  // 复位完成
I/NfcAdaptation: Nfc HAL initialized
D/nfa_sys: nfa_sys_enable_subsystems
D/nfa_dm: Starting RF discovery...
```

------

### **错误处理机制**

1. **超时检测**（HAL层）：

   cpp

   

   复制

   

   下载

   ```
   // phNxpNciHal.cc
   if (!wait_for_response(NCI_RESET_TIMEOUT)) {
       LOG(ERROR) << "Reset timeout!";
       return NFC_STATUS_FAILED;
   }
   ```

2. **硬件状态检查**（驱动层）：

   c

   

   复制

   

   下载

   ```
   // pn54x.c
   if (gpio_get_value(dev->irq_gpio) == 0) {
       dev_err(&client->dev, "IRQ pin not active");
       return -EIO;
   }
   ```

3. **电源管理**：

   c

   

   复制

   

   下载

   ```
   // 错误时关闭电源
   pn54x_disable(dev);
   gpio_set_value(dev->ven_gpio, 0);
   ```

------

### **功耗优化设计**

1. **硬件休眠模式**：

   cpp

   

   复制

   

   下载

   ```
   // HAL发送休眠命令
   static uint8_t cmd_sleep[] = {0x2F, 0x03, 0x01, 0x01};
   write(dev, cmd_sleep, sizeof(cmd_sleep));
   ```

2. **RF场动态调节**：

   c

   

   复制

   

   下载

   ```
   // 根据距离调整功率
   reg_write(dev->client, RF_POWER_CTL, 
             (distance < 5cm) ? LOW_POWER : HIGH_POWER);
   ```

------

通过源码分析可见，NFC打开流程是严格的**分层协作**：

1. **应用层**处理用户意图和状态管理
2. **协议栈**协调多模块工作流
3. **HAL层**实现标准化硬件命令
4. **驱动层**直接控制物理硬件

关键设计亮点：

- **异步事件机制**：通过GKI任务调度处理RF发现事件
- **硬件抽象**：HAL层使不同NFC芯片（NXP/STM/Broadcom）接口统一
- **安全隔离**：JNI层实现Java/Native边界检查
- **功耗控制**：多层协同的低功耗设计（协议栈超时机制 + HAL休眠命令 + 驱动断电）