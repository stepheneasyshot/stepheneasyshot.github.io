---
layout: post
description: > 
  本文介绍了Android平台上连接手环等BLE设备一般的通信流程
image: 
  path: /assets/img/blog/blogs_ble_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_ble_cover.png
    960w:  /assets/img/blog/blogs_ble_cover.png
    480w:  /assets/img/blog/blogs_ble_cover.png
accent_image: /assets/img/blog/blogs_ble_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android进阶】BLE通信一般流程
在当今的物联网（IoT）时代，我们的智能手机早已不仅仅是通讯工具，更是连接和控制身边智能硬件的**中枢**。无论是追踪你运动数据的智能手环，监测你睡眠质量的智能床垫，还是让你告别钥匙的智能门锁，它们与 Android 设备之间高效、可靠的通信桥梁，正是**低功耗蓝牙 (Bluetooth Low Energy, BLE)** 技术。

**什么是 BLE？**

**BLE 低功耗蓝牙**（通常也被称为 Bluetooth Smart）是专为物联网应用设计的无线通信协议。它继承了传统蓝牙的可靠性，但在**功耗上做到了极致优化** 。不同于传统蓝牙需要持续、高带宽的数据流，BLE 的核心在于快速连接、发送少量数据后迅速休眠，这使得它成为电池供电的小型设备的首选。对于 Android 开发者来说，熟练掌握如何利用 Android 系统提供的 API 与这些 BLE 设备进行交互，是构建现代移动应用的关键技能。

**丰富的应用场景**

BLE 连接技术已经渗透到我们生活的方方面面，带来了无限的创新可能：

1.  **健康与运动追踪：** 智能手表、心率带、运动传感器等将实时数据传输到你的 Android App，进行分析和可视化。
2.  **智能家居与控制：** 通过手机控制智能灯泡、温度计、门锁、以及各种传感器。
3.  **资产定位与寻物：** 利用 iBeacon 或 Eddystone 等技术实现室内导航、商家推送，以及通过小巧的蓝牙标签寻找丢失的物品。

本篇博客将介绍Android设备和BLE设备通信的几个关键流程，**扫描**、**连接**、**发现服务**，并最终实现与 BLE 设备的**高效数据交互**。

核心概念先行：

* 手机 (Central / 中心设备)： 主动扫描并连接其他设备的设备，在BLE协议中称为 **中心设备** 。手机、电脑等通常扮演这个角色。
* BLE设备 (Peripheral / 外围设备)： 被动广播自身信息，等待被连接的设备，称为**外围设备**。如智能手环、心率带、防丢器等。
* GATT (Generic Attribute Profile)： 这是连接建立后，双方通信所遵循的核心协议。它定义了一个 **服务-特征值** 的数据结构。
    * 服务： 一个独立的功能模块。例如，一个“心率服务”。
    * 特征： 服务下的具体数据点。它是实际读写操作的对象。例如，在“心率服务”下，会有 **“心率测量特征”** （用于读取心率数据）和 **“心率位置特征”** （用于写入或通知佩戴位置）。
    * 属性： 特征、服务等都被称为属性，每个属性都有一个唯一的标识符。

## 1. 寻找设备 广播与扫描
这个阶段的目标是让手机发现BLE设备的存在
### 1.1 外围设备广播
外围设备会周期性地（例如每秒100次）向周围环境 **发送广播数据包**（Advertising Packets）。这些数据包包含少量信息，这就是广播报文。

广播报文内容：
* 设备地址： 类似MAC地址，是设备的唯一标识符。
* 设备名称： 可读的名称，如 “MI_Band”。
* 发射功率： 用于粗略的距离估算。
* 服务UUIDs： 设备所支持的主要服务的列表。这是手机判断设备类型的关键（例如，看到“心率服务”的UUID，就知道这是个心率设备）。
* 制造商特定数据： 设备厂商可以自定义放入一些数据，如电池电量、硬件版本等。

### 1.2 中心设备扫描
手机（通常作为中央设备 Central）处于**扫描**状态，App通过操作系统提供的蓝牙API启动蓝牙扫描，手机的蓝牙芯片会监听来自周围 BLE 设备的广播数据包，在同样的广播通道上监听这些广播报文。

手机收到数据包后，会将其解析并**回调**给应用程序，提取出上述广播的信息，并在App的扫描结果列表中显示出来（例如，显示“发现设备：MI_Band”）。

此时，手机知道了BLE设备的存在和基本信息，但双方还未建立正式连接。这是设备间建立联系的第一步。

### 1.3 广播细节
在 **手机(Observer)** 跟 **设备B（Advertiser）** 建立连接之前，设备B需要先进行广播，即设备B不断发送如下广播信号，**t** 为广播间隔。

每发送一次广播包，我们称其为一次广播事件（advertising event），因此t也称为广播事件间隔。广播事件是有一个持续时间的，蓝牙芯片只有在广播事件期间才打开射频模块，这个时候功耗比较高，其余时间蓝牙芯片都处于idle状态，因此平均功耗非常低，以Nordic nRF52810为例，每1秒钟发一次广播，平均功耗不到11uA。

每一个广播事件包含三个广播包，即分别在37/38/39三个射频通道上同时广播相同的信息，即真正的广播事件是下面这个样子的。

![广播事件](/assets/img/blog/blogs_ble_advertise.png)

设备B不断发送广播信号给手机（Observer），如果手机不开启扫描窗口，手机是收不到设备B的广播的，如下图所示，不仅手机要开启射频接收窗口，而且 **只有手机的射频接收窗口跟广播发送的发射窗口匹配成功** ，而且 **广播射频通道和手机扫描射频通道是同一个通道** ，手机才能收到设备B的广播信号。

也就是说，如果设备B在37通道发送广播包，而手机在扫描38通道，那么即使他们俩的射频窗口匹配，两者也是无法进行通信的。

由于这种匹配成功是一个概率事件，因此手机扫到设备B也是一个概率事件，也就是说，手机有时会很快扫到设备B，比如只需要一个广播事件，手机有时又会很慢才能扫到设备B，比如需要10个广播事件甚至更多。
#### 避免信道干扰
为了避免与其他无线设备（主要是Wi-Fi）的干扰，BLE广播事件的三个广播包分别在 **37/38/39** 三个射频通道上广播，这三个通道分别避开了Wi-Fi信道 **1/11/6** 的下边缘和上边缘。

* 频率冲突：Wi-Fi主要工作在2.4GHz频段，而这个频段也正是蓝牙（包括BLE）的工作频段。Wi-Fi将其频段划分为多个信道，例如常用的1, 6, 11信道。
    * Wi-Fi信道1 的中心频率是 **2.412 GHz**
    * Wi-Fi信道6 的中心频率是 **2.437 GHz**
    * Wi-Fi信道11 的中心频率是 **2.462 GHz**

* BLE信道的频率：
    * BLE信道37： **2.402 GHz**
    * BLE信道38： **2.426 GHz**
    * BLE信道39： **2.480 GHz**

你可能会注意到，BLE总共有40个物理信道（0-39）。37、38、39用于广播，那么剩下的0-36信道用于什么呢？
* 数据信道：在BLE连接建立之后，通信双方会使用 **自适应跳频技术** ，在0-36这37个信道上进行数据传输。
* 跳频的意义：连接后使用跳频，是为了在数据传输阶段也能 **动态地避开瞬时干扰** 。主从设备会共同协商，跳过那些信噪比差、干扰大的信道，从而在连接状态下也能维持一个稳定、高效的数据链路。

## 2. 建立连接
当用户在手机 App 上选择一个设备后，就会开始连接过程。

手机请求：
* **动作:** 手机 App 调用系统 API，向选定的 BLE 设备**发送连接请求**（Connection Request）。
* **信息:** 请求中会包含**连接参数**，如连接间隔（Connection Interval）、从机延迟（Slave Latency）和超时时间（Supervision Timeout），这些参数定义了连接后数据交换的频率和容错能力。即双方会协商一套通信参数，如连接间隔（设备多久通信一次，影响功耗和响应速度）、从机延迟等。

BLE 设备响应：
* **动作:** BLE 设备从广播状态切换到**连接**状态，并**接受**连接请求。
* **结果:** 双方建立了一个**双向的、独占的**连接。从此刻起，设备停止广播，并且只有这个手机能与它通信。双方进入**链路层**（Link Layer）的数据交换阶段。在连接状态下，通信会从之前的3个广播通道切换到37个数据通道，并通过一种自适应跳频技术来避免无线干扰，保证通信稳定。

## 3. 服务发现
连接建立后，手机需要知道设备上有什么功能。

BLE 使用 **GATT (Generic Attribute Profile)** 规范来组织数据。数据被组织成服务和特性，每个特性都有一个 **UUID** 标识。

手机作为 **GATT 客户端**，会自动向BLE设备发送一个 **服务发现请求** 。

BLE设备会将其内部的所有服务，以及每个服务下的所有特征，像一个文件目录一样完整地返回给手机。

手机端的蓝牙协议栈会解析这个“目录”，并建立一个本地的GATT数据库。手机 App 知道了设备的全部功能结构（UUID 和属性），就可以通过查询这个数据库来知道可以对设备进行哪些操作。

## 4. 配对绑定
**安全校验和配对（Bonding/Pairing）** 在手机和 BLE 设备之间的通信中**不是必须的**。是否需要配对，完全取决于 **BLE 设备上的特性（Characteristic）** 的配置。

很多 BLE 服务中的特性（例如，电池电量、设备名称、简单的通知开关）被配置为可以进行未加密或无需身份验证的读取和写入操作。对于这些特性，手机可以直接在连接建立后进行服务发现，然后直接进行读写，不需要经过配对（Pairing）或绑定（Bonding）的复杂流程。

**配对** 是用于建立加密和身份验证的链路，它在以下情况是必须的：
* 敏感数据传输： 当传输的数据具有隐私性或敏感性时（如医疗数据、GPS 位置、个人运动记录）。
* 控制关键功能： 当手机需要控制设备的关键、有影响的功能时（如智能门锁的开关、支付授权、修改固件设置）。
* 需要用户身份确认： 设备的某个特性被配置为需要加密或身份验证权限才能访问。当手机尝试读写这个受保护的特性时，BLE 栈（Stack）会强制触发配对流程。

### 配对/绑定 流程
配对和绑定是为了建立**信任**关系，并交换**安全密钥**，以便在后续连接中能进行**加密通信**。安全性和配对通常发生在服务发现之后，或在第一次读写受保护的特性时触发。如访问加密、身份验证的特性时，系统会自动触发配对流程。

通信双方协商安全级别和配对方式（如 **Just Works**、**Passkey Entry**、**Numeric Comparison** 等）。配对过程中，手机可能会弹出配对请求框，显示一个随机生成的6位数字码。用户需要在设备上确认这个数字码，或者简单地点击 “配对” 按钮。然后双方会交换密钥，建立信任关系。

配对成功后，双方交换 **长期密钥 (LTK)** ，并将其存储起来（称为**绑定**）。绑定后，后续连接可以直接使用 **LTK** 建立加密链路，无需重复配对。

## 5. 通信 - 基于GATT的数据交互
所有通信都是通过 **对特征的读写** 操作完成的。特征有不同的属性来控制其行为：

* Read： 手机可以主动读取特征的值。例如，读取一次当前的电池电量
* Write： 手机可以向特征写入数据（控制设备）。例如，向设备发送一个“寻找手环”的命令，手环收到之后就会震动
* Notify / Indicate： 这是BLE通信中最重要、最节能的模式。设备可以在数据变化时，主动向手机推送数据，而手机无需不停地询问。
    * Notify 通知： 不可靠通知，手机不回复确认。
    * Indicate 指示： 可靠通知，设备会等待手机的确认后再发送下一条。

这个流程是所有基于 BLE 的应用通信的基础。在 Android 平台，可以在 Android 的 `BluetoothLeScanner` 类中找到扫描相关的 API，在 `BluetoothGatt` 类中找到连接、服务发现和数据读写相关的 API。

## Android项目实践
### 1.扫描设备
连接BLE设备，首先需要扫描。Android SDK中使用 `BluetoothLeScanner` 对象来执行扫描的动作：

```java
private BluetoothLeScanner bluetoothLeScanner;

bluetoothLeScanner = bluetoothAdapter.getBluetoothLeScanner();

bluetoothLeScanner.startScan(scanCallback);
```

由于扫描是一个异步的过程，所以这里需要传入一个回调接口，我们在回调接口去获取扫描的结果：

```java
public ScanCallback scanCallback = new ScanCallback() {
        @Override
        public void onScanResult(int callbackType, ScanResult result) {
            super.onScanResult(callbackType, result);
            ScanRecord record = result.getScanRecord();
            BluetoothDevice device = result.getDevice();
            String deviceName = record.getDeviceName();
            Log.d(TAG, "record name:" + deviceName);
            Log.d(TAG, "ServiceUuids:" + record.getServiceUuids());

            if (TextUtils.isEmpty(deviceName)) {
                return;
            }

            if (deviceName.startsWith("XX")) {    //这里我们可以找出设备名以XX开头的BLE设备
              
                byte[] bytes = record.getBytes();    //这里可以获取整个广播的完整数据，包括协议头等
                for (int i = 0; i < bytes.length; i++) {
                    Log.d(TAG, "[" + i + "]:" + bytes[i]);
                }
                
                bluetoothLeScanner.stopScan(scanCallback);     //需要停止扫描
                              
            }
        }

    };
```

### 2.连接设备
在找到了“XX”名称的设备后，我们就可以发起GATT协议的连接了：

```java
/**
 * 
 * @param autoConnect Whether to directly connect to the remote device (false) or to automatically connect as soon as the remote device becomes available (true).
 * @param callback GATT callback handler that will receive asynchronous callbacks.
*/
device.connectGatt(context, false, gattCallback);
```

第二个参数 `autoConnect` 如果是false代表仅发起本次连接，如果连接不上则会反馈连接失败；如果是true则表示只要这个远程的设备可用，那么底层协议栈就会自动去连接，并且第一次连接不上，也会继续去连接。

第三个参数 `callback` 是一个 **关于GATT协议相关的回调接口** ，主要有GATT连接状态的回调、发现Service服务的回调、特征值(Characteristic)发生改变的回调、最大传输单元（MTU）改变的回调、物理层发送模式(PHY)改变回调等，如下：

```java
BluetoothGattCallback gattCallback = new BluetoothGattCallback() {

        @Override
        public void onPhyUpdate(BluetoothGatt gatt, int txPhy, int rxPhy, int status) {
            super.onPhyRead(gatt, txPhy, rxPhy, status);
            Log.d(TAG, "onPhyUpdate txPhy:" + txPhy + "; rxPhy:" + rxPhy);
        }

        @Override
        public void onMtuChanged(BluetoothGatt gatt, int mtu, int status) {
            super.onMtuChanged(gatt, mtu, status);
            Log.d(TAG, "onMtuChanged mtu:" + mtu + "; status:" + status);
        }

        @Override
        public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
            super.onConnectionStateChange(gatt, status, newState);
            Log.d(TAG, "onConnectionStateChange newState:" + newState);
            if (newState == BluetoothProfile.STATE_CONNECTED) {  //协议连接成功
                Log.d(TAG, "STATE_CONNECTED");
                bluetoothGatt = gatt;             
                bluetoothGatt.discoverServices();   //发现service服务
            } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {         //协议连接失败
                Log.d(TAG, "STATE_DISCONNECTED");
            }

        }
}
```

### 3.获取服务和特征
在GATT协议连接成功之后，就可以去发现从设备端提供了哪些Service服务，如上代码。

这是一个异步的过程，待从设备反馈了自己提供的服务之后，Android框架层会通过BluetoothGattCallback回调通知，如下：

```java
BluetoothGattCallback gattCallback = new BluetoothGattCallback() {
    @Override
    public void onServicesDiscovered(BluetoothGatt gatt, int status) {
        super.onServicesDiscovered(gatt, status);
        List<BluetoothGattService> services = gatt.getServices();
        for (BluetoothGattService service : services) {
            Log.d(TAG, "UUID:" + service.getUuid().toString());
        }

        //1.根据UUID获取到服务
        mGattService = gatt.getService(UUID.fromString("0000ff00-0000-1000-8000-00805f9b34fb"));

        if (mGattService == null) {
            Log.w(TAG, "GattService is null!");
        } else {
            Log.i(TAG, "connect GattService");
            if (writeCharacteristic == null) {
                //2.获取一个特征（Characteristic），这是从设备定义好的，我通过这个Characteristic去写从设备感兴趣的值
                writeCharacteristic = mGattService
                        .getCharacteristic(UUID.fromString("0000ff02-0000-1000-8000-00805f9b34fb"));
            }
            if (readCharacteristic == null) {
                //3.获取一个主设备需要去读的特征（Characteristic），获取从设备发送过来的数据
                readCharacteristic = mGattService
                        .getCharacteristic(UUID.fromString("0000ff01-0000-1000-8000-00805f9b34fb"));

                //4.注册特征（Characteristic）值改变的监听
                bluetoothGatt.setCharacteristicNotification(readCharacteristic, true);
                List<BluetoothGattDescriptor> descriptors = readCharacteristic.getDescriptors();
                for (BluetoothGattDescriptor descriptor : descriptors) {
                    descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
                    bluetoothGatt.writeDescriptor(descriptor);
                }
            }

        }
    }
};
```

经过上述代码中的四个步骤，两个设备间已经可以发送和接收数据了。
### 4.通过特征（Characteristic）发送数据
把需要发送的数据设置到 `writeCharacteristic`，然后再调用 `BluetoothGatt` 的写入方法，即可完成数据的发送：

```java
writeCharacteristic.setValue(datas);
bluetoothGatt.writeCharacteristic(writeCharacteristic);
```

### 5.读取数据
当从设备有数据发送到主设备之后，Android系统会回调 `BluetoothGattCallback` 的 `onCharacteristicChanged` 方法通知：

```java
@Override
        public void onCharacteristicChanged(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic) {
            super.onCharacteristicChanged(gatt, characteristic);
            UUID uuid = characteristic.getUuid();
            byte[] receiveData = characteristic.getValue();
            for (byte b : receiveData) {
                Log.d(TAG, "receiveData:" + Integer.toHexString(b));
            }
        }
```

### 三、注意事项
#### 1.自动连接属性
`connectGatt` 方法的自动连接参数设置为true之后，连接建立了，这个时候如果是断开连接，如下：

```java
bluetoothGatt.disconnect();
```

虽然在Android层面的 `BluetoothGattCallback` 接口会立刻反馈一个 `STATE_DISCONNECTED` 信号值，但是在数据链路层却还是处于连接的状态，连接并没有断开。

#### 2.开启定位功能
现在Android最新的版本，需要开启定位才能使用BLE功能。

判断定位功能是否开启：

```java
private boolean isLocationEnable(Context context) {
        LocationManager locationManager = (LocationManager) context.getSystemService(Context.LOCATION_SERVICE);
        boolean networkProvider = locationManager.isProviderEnabled(LocationManager.NETWORK_PROVIDER);
        boolean gpsProvider = locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER);
        if (networkProvider || gpsProvider) {
            return true;
        }
        return false;
    }
```

开启定位功能的方法：

```java
LocationManager locationManager = (LocationManager) context.getSystemService(Context.LOCATION_SERVICE);
            try {
                Field field = UserHandle.class.getDeclaredField("SYSTEM");
                field.setAccessible(true);
                UserHandle userHandle = (UserHandle) field.get(UserHandle.class);
                Method method = LocationManager.class.getDeclaredMethod(
                        "setLocationEnabledForUser",
                        boolean.class,
                        UserHandle.class);
                method.invoke(locationManager, true, userHandle);
            } catch (Exception e) {
            }
```

#### 3.最大传输单元（MTU）的设置
Android默认的最大传输单元(MTU)是23个字节，除去报文头占用的3个字节，实际最大只能传递20个字节。当两个设备之间传递的数据长度超过20字节的时候，数据就会被截断，导致通信异常。

只有在GATT协议连接成功之后，才可以设置MTU值，最大MTU=512，如下：

```java
bluetoothGatt.requestMtu(128);
```

#### 4.从设备广播间隔影响连接
当Android协议栈（Host）给蓝牙芯片Chip发送一个连接的指令，芯片在收到之后，会在一定的时间内去接收从设备的广播，在收到广播之后才会发送连接请求给从设备；如果从设备的广播间隔设置不合理，就会导致芯片无法在限定的时间内收到广播，导致无法发送连接请求。