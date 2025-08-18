nfcinjector类主要作用是提供依赖注入，帮助解决NFC栈中的循环依赖问题，并集中管理NFC服务所需的各种依赖和工具。包含了许多NFC服务所需的组件，例如上下文、主循环、事件日志、路由表解析器、屏幕状态助手、解锁管理器、设备配置门面、NFC分发器、振动效果、备份管理器、功能标志、统计工具、前台工具、诊断工具、服务注册器、看门狗、锁屏管理器等。



看门狗通常用于监控系统运行状态，如果系统出现故障（如死锁、无响应等），看门狗会触发重启等恢复机制。



Binder是Android特有的进程间通信（IPC）机制，它允许不同进程之间进行高效的方法调用。在Android系统中，系统服务（如NFC服务）运行在独立的进程（通常是system_server进程）中，而应用程序运行在自己的进程。当应用程序需要调用系统服务提供的功能时，就需要通过Binder进行跨进程通信。

1. **NFC系统服务的注册与获取**：在系统启动时，NFC服务会向ServiceManager注册自己。应用程序通过`getSystemService(Context.NFC_SERVICE)`获取NFC服务的代理对象（Binder代理），然后通过这个代理对象调用NFC服务的功能。
2. **NFC服务接口定义**：NFC服务会通过AIDL（Android Interface Definition Language）定义接口。这些接口描述了服务提供的方法，例如启用/禁用NFC、发送数据等。
3. **Binder在NFC服务内部的使用**：在NFC服务内部，Binder也被用于跨线程通信，例如将来自客户端的调用分发到主线程处理。

shouldEnableNfc（）

1. **`getNfcOnSetting()`**  

   \- 检查用户设置中NFC开关是否打开（通常对应“设置”中的NFC开关状态）。

   \- 实现：读取`Settings.Global.NFC_ON`或`Settings.Secure.NFC_ON`的值（不同Android版本可能不同）。

2. **`!mNfcInjector.isSatelliteModeOn()`**  

   \- 检查设备是否处于卫星模式（飞行模式的高级形式），在卫星模式下需要关闭NFC。

   \- 实现：通过`NfcInjector`调用`isSatelliteModeOn()`方法，该方法内部会检查系统设置中卫星模式是否开启且是否影响NFC。

3. **`!isNfcUserRestricted()`**  

   \- 检查当前用户是否被限制使用NFC（例如多用户场景下，子用户可能被禁止使用NFC）。

   \- 实现：通常通过`UserManager.DISALLOW_OUTGOING_BEAM`等用户策略判断。

4. **`allowOemEnable()`**  

   \- 设备制造商(OEM)的特殊开关。某些设备可能有硬件级别的特殊限制（如地区锁、硬件故障时强制禁用）。

   \- 实现：由设备制造商实现，可能读取系统属性或硬件状态。

- **OEM 定制支持**：

  - 通过 `INfcOemExtensionCallback` 允许厂商扩展功能（如自定义 NFC 启动流程）。

    

在 Android NFC 开发中，"异步任务"（AsyncTask）是一种**用于在后台线程执行 NFC 操作**的机制，目的是避免阻塞主线程（UI 线程），确保应用流畅响应。





注意：这个任务类被设计为在同一个后台线程上串行执行，因此不会出现多个任务同时修改mState的情况。每个任务执行时，mState要么是STATE_OFF，要么是STATE_ON，

​       在执行过程中可能会临时变为STATE_TURNING_OFF或STATE_TURNING_ON，但执行结束后必须回到STATE_ON或STATE_OFF。

 问题：什么是异步task？

 回答：异步任务（AsyncTask）是Android提供的一个轻量级的异步处理类，它可以在后台线程中执行耗时操作，然后将结果返回到主线程（UI线程）进行更新。

​       这样避免了在主线程中执行耗时操作导致的应用无响应（ANR）问题。

 在这个例子中，EnableDisableTask就是一个异步任务，它用于在后台执行NFC开启/禁用等操作，因为这些操作可能涉及硬件访问，比较耗时。

 使用AsyncTask的好处是：

   \- 简化了线程管理（内部使用了线程池）

   \- 提供了与主线程通信的机制（如onPreExecute, onPostExecute等方法，虽然在这个例子中没有使用onPostExecute，但可以在其他地方重写）

   \- 任务默认是串行执行的（在较新版本中，可以通过executeOnExecutor来改变执行方式），这有助于避免并发问题。





在启动过程中：

 \- GKI_init()初始化内核环境。

 \- GKI_create_task()创建协议栈任务（如NFCA_TASK）。

 \- 然后，当NFCA_TASK任务运行时，它会初始化协议栈，并调用HAL层的初始化函数。

 因此，HAL层的启动是由协议栈任务（NFCA_TASK）触发的，而不是由GKI直接创建任务。





分析：

AsyncTask 的基本概念

`AsyncTask` 是一个用于在后台线程执行任务并在主线程更新 UI 的便捷类。它有三个泛型参数：

1. `Params`：执行任务时传入的参数类型。
2. `Progress`：后台任务执行过程中更新进度的单位类型。
3. `Result`：后台任务执行完毕后的结果类型。

以及四个主要方法：

1. `onPreExecute()`：在后台任务开始前在主线程执行，用于初始化（例如显示进度条）。
2. `doInBackground(Params...)`：在后台线程执行，用于执行耗时操作。此方法中不能更新UI。
3. `onProgressUpdate(Progress...)`：在`doInBackground`中调用`publishProgress`后，在主线程执行，用于更新进度。
4. `onPostExecute(Result)`：在后台任务完成后在主线程执行，用于处理结果（例如更新UI）。



`enableInternal()`方法用于启用NFC。它处理状态检查、初始化NFC硬件、更新状态、应用路由配置等。返回一个布尔值表示启用是否成功。

详细步骤处理：

​	1.日志和状态检查：如果当前状态已经是`STATE_ON`，直接返回true。如果`mAlwaysOnState`（始终开启状态）为`STATE_ON`，但当前不在默认模式，则调用`disableAlwaysOnInternal()`。

​	2.准备启用：记录NFC状态变更的日志（使用`NfcStatsLog`）。更新状态为`STATE_TURNING_ON`（正在开启）。启动一个看门狗线程（`WatchDogThread`）来监视初始化过程，防止超时。

​	3.初始化NFC硬件：更新默认SWP（Single Wire Protocol）到eUICC（嵌入式UICC）的配置。获取一个唤醒锁（`mRoutingWakeLock`）以确保在初始化过程中设备保持唤醒状态。在`try`块中：根据条件判断是否需要进行完整的初始化（`mDeviceHost.initialize()`）：如果支持始终开启（`mIsAlwaysOnSupported`）且当前不是恢复过程（`mIsRecovering`）且`mAlwaysOnState`既不是`STATE_ON`也不是`STATE_TURNING_OFF`，则跳过初始化（因为可能已经初始化）。否则，调用`initialize()`进行初始化。如果初始化失败，记录错误，更新状态为`STATE_OFF`，返回false。在`finally`块中释放唤醒锁。无论初始化成功与否，都会取消看门狗线程。

​	4.初始化后设置：读取系统属性`skipNdefRead`（是否跳过NDEF读取）并记录NCI版本。重置`mPendingPowerStateUpdate`为false。同步块内：清除对象映射（`mObjectMap.clear()`）。 更新状态为`STATE_ON`（启用成功）。调用`onPreferredPaymentChanged()`通知首选支付服务已加载。

​	5.屏幕状态处理：根据是否处于配置模式（`mInProvisionMode`）获取屏幕状态。   \- 如果支持锁屏轮询（`mNfcUnlockManager.isLockscreenPollingEnabled()`），则组合屏幕状态掩码。如果锁屏轮询启用，则应用路由（`applyRouting(false)`）。调用`mDeviceHost.doSetScreenState()`设置屏幕状态。

​	6.恢复保存的技术： 调用`restoreSavedTech()`（可能是恢复之前保存的NFC技术路由？）。

​	7.应用路由配置：如果不支持始终开启，或者当前始终开启状态不是正在切换（即不是`STATE_TURNING_ON`或`STATE_TURNING_OFF`），则调用`applyRouting(true)`开始轮询。

​	8.卡模拟管理：如果设备支持HCE（主机卡模拟），则通知卡模拟管理器NFC已启用。

​	9.恢复过程处理：如果当前处于恢复过程（`mIsRecovering`），则注册全局广播接收器，并将恢复标志置为false。

​	10.省电模式处理：如果省电模式启用，则关闭省电模式并重置标志。

```java
if (mState == NfcAdapter.STATE_ON) return true;  // 已启用则跳过
if (mAlwaysOnState == NfcAdapter.STATE_ON && !isAlwaysOnInDefaultMode()) {
    disableAlwaysOnInternal();  // 处理 AlwaysOn 模式异常
}
updateState(NfcAdapter.STATE_TURNING_ON);  // 标记「启用中」状态
WatchDogThread watchDog = new WatchDogThread("enableInternal", INIT_WATCHDOG_MS);
watchDog.start();  // 防止初始化卡死的超时监控
try {
    mRoutingWakeLock.acquire();  // 持有唤醒锁防止休眠
    if (!mDeviceHost.initialize()) {  // 调用底层 HAL 初始化硬件
        updateState(NfcAdapter.STATE_OFF); // 失败回滚状态
        return false;
    }
} finally {
    mRoutingWakeLock.release();  // 确保释放锁
    watchDog.cancel();  // 停止看门狗
}
nci_version = getNciVersion();  // 获取 NFC 控制器版本
synchronized (NfcService.this) {
    mObjectMap.clear();  // 清理缓存对象
    updateState(NfcAdapter.STATE_ON);  // 正式标记为启用状态
    onPreferredPaymentChanged(NfcAdapter.PREFERRED_PAYMENT_LOADED); // 支付服务回调
}
mScreenState = mScreenStateHelper.checkScreenState(...);  // 计算屏幕状态
mDeviceHost.doSetScreenState(screen_state_mask, mIsWlcEnabled); // 配置 NFC 射频状态
applyRouting(true);  // 应用路由策略（启动轮询）
mScreenState = mScreenStateHelper。checkScreenState（.。。）;  计算屏幕状态
mDeviceHost  中。doSetScreenState（screen_state_mask， mIsWlcEnabled）;  配置 NFC 射频状态
applyRouting(true);   // 应用路由策略（启动轮询）
if (mIsHceCapable) {
    mCardEmulationManager.onNfcEnabled();  // 启用 HCE 主机卡模拟
}
if (mIsRecovering) {
    registerGlobalBroadcastsReceiver();  // 注册全局广播（恢复场景）
    mIsRecovering = false; 
}
if (mIsPowerSavingModeEnabled) {
    mDeviceHost.setPowerSavingMode(false); // 禁用省电模式以保障性能
}
```

### **注意事项**

1. **AlwaysOn 模式处理**

   - 当系统支持常开 NFC（如 AOSP 的 `config_nfcAlwaysOn`）时需特殊处理
   - 通过 `disableAlwaysOnInternal()` 确保状态一致性

2. **eSIM 集成**

   ```
   mCardEmulationManager.updateForDefaultSwpToEuicc();
   ```

   - 处理 eSIM（嵌入式 SIM）的 SWP（单线协议）路由切换

3. **安全 NFC 实现**

   - `mIsSecureNfcEnabled` 控制安全状态
   - 日志区分普通/安全模式：`NFC_STATE_CHANGED__STATE__ON_LOCKED`

4. **调试支持**

   - `DBG` 标记控制详细日志
   - `NfcProperties.skipNdefRead()` 支持跳过 NDEF 读取（测试用途）





applyRouting负责根据系统状态动态调整NFC的轮询和路由行为，确保NFC在正确的时间以正确的配置工作。它通过条件检查、延迟更新、硬件抽象层调用等机制，平衡了功能需求、性能优化和系统稳定性。

流程：

1. **检查NFC是否可用**：如果NFC未启用或正在关闭，则直接返回。
2. **OEM扩展回调处理**：如果存在OEM扩展回调并且回调结果要求跳过路由设置，则返回。
3. **特殊模式处理**：在配置模式（Provision Mode）下刷新标签分发器。
4. **轮询暂停检查**：如果轮询被暂停，则直接返回。
5. **特殊场景处理**：当屏幕解锁且有标签存在时，延迟处理路由更新。
6. **启动看门狗线程**：设置超时监控，防止操作卡死。
7. **计算并应用新的发现参数**：根据屏幕状态计算新的发现参数，如果有变化则更新硬件配置。
8. **清理工作**：取消看门狗线程。

```java
if (!isNfcEnabledOrShuttingDown()) return; // NFC 不可用时退出
if (mNfcOemExtensionCallback != null && receiveOemCallbackResult(ACTION_ON_APPLY_ROUTING)) {
    Log.d(TAG, "applyRouting: skip due to oem callback");
    return; // OEM 定制逻辑跳过路由更新
}
if (mPollingPaused) return; // 轮询暂停时跳过
if (mScreenState == ScreenStateHelper.SCREEN_STATE_ON_UNLOCKED && isTagPresent()) {
    Log.d(TAG, "applyRouting: Not updating, tag connected");
    mHandler.sendMessageDelayed(mHandler.obtainMessage(MSG_RESUME_POLLING),
            APPLY_ROUTING_RETRY_TIMEOUT_MS); // 延迟 200ms 重试
    return;
}
WatchDogThread watchDog = new WatchDogThread("applyRouting", ROUTING_WATCHDOG_MS);
try {
    watchDog.start(); // 启动超时监控（约1秒）
    NfcDiscoveryParameters newParams = computeDiscoveryParameters(mScreenState);
    
    if (force || !newParams.equals(mCurrentDiscoveryParameters)) {
        if (newParams.shouldEnableDiscovery()) {
            boolean shouldRestart = mCurrentDiscoveryParameters.shouldEnableDiscovery();
            mDeviceHost.enableDiscovery(newParams, shouldRestart); // 更新硬件配置
        } else {
            mDeviceHost.disableDiscovery(); // 完全禁用发现
        }
        mCurrentDiscoveryParameters = newParams; // 缓存新参数
    }
} finally {
    watchDog.cancel(); // 确保看门狗终止
}
```

### **关键技术点**

#### 1. **路由参数对象** `NfcDiscoveryParameters`

包含以下核心配置：

```
class NfcDiscoveryParameters {
    boolean enableDiscovery;      // 是否启用发现
    boolean enableLowPowerPolling;// 低功耗轮询模式
    boolean enableReaderMode;     // 读卡器模式
    boolean enableHostRouting;    // 主机路由
    boolean enableP2p;            // 点对点模式
    int techMask;                 // 技术掩码（NFC-A/B/F等）
}
```

#### 2. **屏幕状态与路由关系**

| 屏幕状态                    | 典型路由行为             |
| :-------------------------- | :----------------------- |
| `SCREEN_STATE_OFF_UNLOCKED` | 禁用或低功耗轮询         |
| `SCREEN_STATE_ON_LOCKED`    | 启用基础轮询（排除支付） |
| `SCREEN_STATE_ON_UNLOCKED`  | 全功能启用               |

#### 3. **硬件交互优化**

- **增量更新**：通过 `shouldRestart` 参数避免全量重启

  ```
  enableDiscovery(newParams, shouldRestart) // true=重启 false=热更新
  ```

- **配置缓存**：`mCurrentDiscoveryParameters` 避免重复配置

- **看门狗保护**：`ROUTING_WATCHDOG_MS` 防止硬件无响应

#### 4. **OEM 定制扩展**

```
receiveOemCallbackResult(ACTION_ON_APPLY_ROUTING)
```

- 允许设备厂商通过回调覆盖默认路由行为
- 典型用例：特定区域禁用某些技术

