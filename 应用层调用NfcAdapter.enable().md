 步骤1：应用层调用`NfcAdapter.enable()`

  在应用代码中，我们可能会这样写：

​      NfcAdapter nfcAdapter = NfcAdapter.getDefaultAdapter(context);

​      nfcAdapter.enable();

 步骤2：`NfcAdapter.enable()`的实现

  在`NfcAdapter`类中，`enable()`方法会通过Binder调用到系统服务（即NfcService）中的对应方法。

 步骤3：获取NfcService的Binder代理

  在`NfcAdapter`中，有一个`INfcAdapter`类型的成员变量`mService`，它实际上是一个Binder代理对象，指向系统服务端NfcService中的`NfcAdapterService`（即INfcAdapter.Stub的子类）。

 步骤4：通过Binder代理发起调用

  当我们调用`nfcAdapter.enable()`时，实际上会调用`mService.enable()`，这个`mService`就是Binder代理对象（即`INfcAdapter.Stub.Proxy`）。

 步骤5：Binder代理的`enable()`方法

  在Binder代理的`enable()`方法中，会进行以下操作：

​      \- 将调用参数打包到Parcel对象中

​      \- 通过Binder驱动发送事务（transaction）到服务端

​      \- 等待并读取返回结果

 步骤6：服务端（NfcService）处理调用

  服务端收到事务请求后，会调用`NfcAdapterService`（继承自`INfcAdapter.Stub`）的`onTransact`方法，根据事务代码（code）找到对应的`enable()`方法并执行。

 步骤7：服务端执行真正的enable逻辑

  在`NfcAdapterService.enable()`方法中，会进行权限检查，然后调用`NfcService`中真正实现enable功能的方法。

 下面我们详细看一下代码流程：

1. 应用层调用：

​    `NfcAdapter.enable()`

2. `NfcAdapter`中的`enable`方法：

​    \```java

​    public boolean enable() {

​        try {

​            return mService.enable();

​        } catch (RemoteException e) {

​            attemptDeadServiceRecovery(e);

​            // 尝试恢复后再次调用？实际上这里可能还是失败，所以返回false

​            return false;

​        }

​    }

​    \```

3. `mService`是`INfcAdapter`接口，它的实际类型是`INfcAdapter.Stub.Proxy`。
4. 在`INfcAdapter.Stub.Proxy`中的`enable`方法：

​    \```java

​    @Override

​    public boolean enable() throws RemoteException {

​        Parcel _data = Parcel.obtain();

​        Parcel _reply = Parcel.obtain();

​        boolean _result;

​        try {

​            _data.writeInterfaceToken(DESCRIPTOR);

​            // 发起transact调用

​            boolean _status = mRemote.transact(Stub.TRANSACTION_enable, _data, _reply, 0);

​            if (!_status) {

​                // 如果transact返回false，可能是服务端没有实现该方法，或者Binder出错

​            }

​            _reply.readException();

​            _result = (_reply.readInt() != 0);

​        } finally {

​            _reply.recycle();

​            _data.recycle();

​        }

​        return _result;

​    }

​    \```

5. 服务端（NfcService进程）的`INfcAdapter.Stub`子类（通常命名为`NfcAdapterService`）会处理这个事务。在`NfcAdapterService`中，我们重写了`onTransact`方法（实际上在Stub中已经实现了，根据事务代码分发到具体方法）和具体的enable方法。
6. `NfcAdapterService`中的`enable`方法：

​    \```java

​    private class NfcAdapterService extends INfcAdapter.Stub {

​        // ... 其他方法

​        @Override

​        public boolean enable() throws RemoteException {

​            // 1. 权限检查

​            mContext.enforceCallingPermission(android.Manifest.permission.NFC, "NFC");

​            // 2. 记录调用者信息（用于后续的权限控制）

​            int uid = Binder.getCallingUid();

​            long token = Binder.clearCallingIdentity();

​            try {

​                // 3. 将调用切换到主线程（因为NFC操作需要在主线程进行）

​                // 这里通常是通过Handler发送消息到主线程，然后调用NfcService的enableInternal方法

​                // 注意：这里可能不会等待结果，而是直接返回true，然后异步执行enable操作

​                // 具体实现可能如下：

​                sendMessage(NfcService.MSG_ENABLE, uid);

​                return true;

​            } finally {

​                Binder.restoreCallingIdentity(token);

​            }

​        }

​    }

​    \```

7. 在`NfcService`中，有一个`Handler`（运行在主线程）处理`MSG_ENABLE`消息，最终调用`enableInternal`方法，该方法会执行一系列操作来启动NFC。
8. 在`NfcService`的`enableInternal`方法中，会调用底层设备（通过`DeviceHost`接口）的`enable`方法，并更新NFC状态。

 注意：以上代码是简化流程，实际代码中可能包含更多的错误处理和状态检查。

 总结：

   应用调用NfcAdapter.enable() -> 通过Binder调用到NfcService进程中的NfcAdapterService.enable() -> 经过权限检查和线程切换后，调用到NfcService.enableInternal() -> 最终调用到底层NFC控制器的enable方法。