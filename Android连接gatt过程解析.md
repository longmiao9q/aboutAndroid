## framework层调用分析
### connectGatt

```java
    public BluetoothGatt connectGatt(Context context, boolean autoConnect,
                                     BluetoothGattCallback callback, int transport) {
        BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter();
        IBluetoothManager managerService = adapter.getBluetoothManager();
        try {
            IBluetoothGatt iGatt = managerService.getBluetoothGatt();
            if (iGatt == null) {
                // BLE is not supported
                return null;
            }
            BluetoothGatt gatt = new BluetoothGatt(context, iGatt, this, transport);
            gatt.connect(autoConnect, callback);
            return gatt;
        } catch (RemoteException e) {Log.e(TAG, "", e);}
        return null;
    }
```

### getDefaultAdapter
```java
    public static final String BLUETOOTH_MANAGER_SERVICE = "bluetooth_manager";
    
    public static synchronized BluetoothAdapter getDefaultAdapter() {
        if (sAdapter == null) {
            IBinder b = ServiceManager.getService(BLUETOOTH_MANAGER_SERVICE);
            if (b != null) {
                IBluetoothManager managerService = IBluetoothManager.Stub.asInterface(b);
                sAdapter = new BluetoothAdapter(managerService);
            } else {
                Log.e(TAG, "Bluetooth binder is null");
            }
        }
        return sAdapter;
    }
```
通过ServiceManager.getService获得一个远程对象，然后通过`asInterface`转成`IBluetoothManager`对象，这里获取到的是`BluetoothManagerService`这个服务，该服务的代码位于 `/framework/base/services/core/java/com/android/server/BluetoothManagerService.java`

### getBluetoothGatt

`IBluetoothGatt`这个对象的创建位于BluetoothManagerService中。

先看**bluetoothStateChangeHandler**这个函数，顾名思义，它是处理蓝牙状态变化的函数，它里面有一段这样的代码：

```java
                } else if (!intermediate_off) {
                    // connect to GattService
                    if (DBG) Log.d(TAG, "Bluetooth is in LE only mode");
                    if (mBluetoothGatt != null) {
                        if (DBG) Log.d(TAG, "Calling BluetoothGattServiceUp");
                        onBluetoothGattServiceUp();
                    } else {
                        if (DBG) Log.d(TAG, "Binding Bluetooth GATT service");
                        if (mContext.getPackageManager().hasSystemFeature(
                                                        PackageManager.FEATURE_BLUETOOTH_LE)) {
                            Intent i = new Intent(IBluetoothGatt.class.getName());
                            doBind(i, mConnection, Context.BIND_AUTO_CREATE | Context.BIND_IMPORTANT, UserHandle.CURRENT);
                        }
                    }
                    sendBleStateChanged(prevState, newState);
                    //Don't broadcase this as std intent
                    isStandardBroadcast = false;

                }
```

看到上面的`doBind`函数吗，此处正在绑定一个类似名叫`BluetoothGattService`的服务，我们先不管绑定的到底是什么服务，我们先看服务绑定成功后的回调

```java
        public void onServiceConnected(ComponentName className, IBinder service) {
            if (DBG) Log.d(TAG, "BluetoothServiceConnection: " + className.getClassName());
            Message msg = mHandler.obtainMessage(MESSAGE_BLUETOOTH_SERVICE_CONNECTED);
            // TBD if (className.getClassName().equals(IBluetooth.class.getName())) {
            if (className.getClassName().equals("com.android.bluetooth.btservice.AdapterService")) {
                msg.arg1 = SERVICE_IBLUETOOTH;
                // } else if (className.getClassName().equals(IBluetoothGatt.class.getName())) {
            } else if (className.getClassName().equals("com.android.bluetooth.gatt.GattService")) {
                msg.arg1 = SERVICE_IBLUETOOTHGATT;
            } else {
                Log.e(TAG, "Unknown service connected: " + className.getClassName());
                return;
            }
            msg.obj = service;
            mHandler.sendMessage(msg);
        }
```

自然到了handler里面的处理逻辑

```java
                case MESSAGE_BLUETOOTH_SERVICE_CONNECTED:
                {
                    if (DBG) Log.d(TAG,"MESSAGE_BLUETOOTH_SERVICE_CONNECTED: " + msg.arg1);

                    IBinder service = (IBinder) msg.obj;
                    synchronized(mConnection) {
                        if (msg.arg1 == SERVICE_IBLUETOOTHGATT) {
                            mBluetoothGatt = IBluetoothGatt.Stub.asInterface(service);
                            onBluetoothGattServiceUp();
                            break;
                        } // else must be SERVICE_IBLUETOOTH

                        //Remove timeout
                        mHandler.removeMessages(MESSAGE_TIMEOUT_BIND);

                        mBinding = false;
                        mBluetooth = IBluetooth.Stub.asInterface(service);

```
好了，现在知道IBluetoothGatt是个什么东西了，它也是通过binder机制获得的一个远程对象服务。这里可以揭秘，这个服务就是名为`GattService`的服务，它位于`/packages/apps/Bluetooth/src/com/android/bluetooth/gatt/GattService.java`，为什么是它？看看上面进行绑定的代码，**Intent i = new Intent(IBluetoothGatt.class.getName());**，GattService是通过App的形式进行服务的，在对应的manifest文件中，我们找到了如下的片段：

```xml
        <service
            android:process="@string/process"
            android:name = ".gatt.GattService"
            android:enabled="@bool/profile_supported_gatt">
            <intent-filter>
                <action android:name="android.bluetooth.IBluetoothGatt" />
            </intent-filter>
        </service>
```

让我们再回头看看，进行gatt的连接是调用了`BluetoothGatt中`的`connect`函数，然后又调用了`registerClient`函数，到此，连接操作好像到此就结束了，我们需要注意，蓝牙大多数的操作都是异步进行的，结果是通过回调的方式，我们在registerClient阶段传递了一个`IBluetoothGattCallback`的对象，其中有一个`onClientRegistered`这样一个回调函数

```java
            public void onClientRegistered(int status, int clientIf) {
                if (DBG) Log.d(TAG, "onClientRegistered() - status=" + status
                    + " clientIf=" + clientIf);
                if (VDBG) {
                    synchronized(mStateLock) {
                        if (mConnState != CONN_STATE_CONNECTING) {
                            Log.e(TAG, "Bad connection state: " + mConnState);
                        }
                    }
                }
                mClientIf = clientIf;
                if (status != GATT_SUCCESS) {
                    mCallback.onConnectionStateChange(BluetoothGatt.this, GATT_FAILURE,
                                                      BluetoothProfile.STATE_DISCONNECTED);
                    synchronized(mStateLock) {
                        mConnState = CONN_STATE_IDLE;
                    }
                    return;
                }
                try {
                    mService.clientConnect(mClientIf, mDevice.getAddress(),
                                           !mAutoConnect, mTransport); // autoConnect is inverse of "isDirect"
                } catch (RemoteException e) {
                    Log.e(TAG,"",e);
                }
            }
```
在这个函数中，我们才看到了连接gatt的操作。这里又出现了`mService`这个对象，根据上面的分析，这个服务就是`GattService`，让我们再一次转到该对象中，看看`clientConnect`做了什么操作

```java
    void clientConnect(int clientIf, String address, boolean isDirect, int transport) {
        enforceCallingOrSelfPermission(BLUETOOTH_PERM, "Need BLUETOOTH permission");

        if (DBG) Log.d(TAG, "clientConnect() - address=" + address + ", isDirect=" + isDirect);
        gattClientConnectNative(clientIf, address, isDirect, transport);
    }
```
`clientConnect`中调用了`gettClientConnectNative`函数，它将调用native层的函数。我们的分析到此为止。






