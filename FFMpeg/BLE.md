## BLE

```shell
BLE的默认MTU大小为23字节，其中3字节用于协议头，因此实际可传输的数据为20字节，如果需要传输更大数据，开发人员可以设置MTU大小来优化数据传输。
iOS没有开放的API来请求MTU更改/更新，因此连接过程中协商MTU。iOS将自动使用两个设备支持的最大值，iOS最多支持185，因此这是将支持的最大值。如果外围设备支持且MTU小于185，则将使用该值。 
```

**Central:** 中心设备,发起蓝牙连接的设备(一般指手机)
**Peripheral:** 外设,被蓝牙连接的设备(一般是运动手环/蓝牙模块)
**Service:**服务,每个设备会提供服务,一个设备有很多服务，比如手环的震动和亮起来的颜色是两个不同服务
**Characteristic**:特征,每个服务中包含很多个特征,这些特征的权限一般分为:读(read)/写(write)/通知(notice)几种,可以通过特征进行读写数据(重要角色)(中心设备写入数据的时候一定要找到可写入特征才可以写入成功)
**Descriptor:**描述者,每个特征可以对应一个或者多个描述者,用于描述特征的信息或者属性

建立中心角色
扫描外设(Discover Peripheral)
连接外设(Connect Peripheral)
扫描外设中的服务和特征
获取外设的services
获取外设的Characteristics,获取characteristics的值,获取Characteristics的Descriptor和Descriptor的值
利用特征与外设做数据交互(Explore And Interact)
订阅Characteristic的通知
断开连接(Disconnect)



问题1.调用搜索函数时，搜索不到设备的问题，返回的nil。

答：当首次调用函数搜索设备外设时，无法获取外设设备信息的原因是central的state为CBCentralManagerStateUnknown,这个状态表示手机设备的蓝牙状态为未开启。解决方法：需要在此委托方法中监听蓝牙状态的状态改变为ON时，去开启扫描操作。

问题2.外设蓝牙名称被修改后可能搜索不到的问题

答：在测试的过程中正常获取蓝牙名称是通过peripheral.name获取，但是可能存在这种情况是当修改连接过的蓝牙名称后，可能存在搜索不到的情况。解决方法：在蓝牙的广播数据中 根据@"kCBAdvDataLocalName"这个key 便可获得准确的蓝牙名称。

问题3.调用断开蓝牙的接口，手机蓝牙并没有马上与外设断开连接，而是等待5秒左右的时间后才真正断开。

答：可以与硬件开发的同事沟通，从设备收到数据后主动断开连接即可。

问题4.是否能长时间处于后台？

答：可以，后台长时间执行需要开启Background Modes。后台扫描设备跟前台扫描周围设备有一点不同：  也许是考虑到功耗的原因，在后台只能搜索特定的设备，所以必须要传Service UUID。不传的话一台设备都搜不到,而这时就需要外设在广播包中有Service UUID。

问题5.蓝牙允许连接的最大距离支持是多少？

答：iOS 蓝牙允许连接的最大距离的限制是10m。

问题6.蓝牙连接成功需要多长时间

答：正常连接周围的蓝牙外设一般时在5秒内

问题7.多台设备是否能同时连接？

答：官方文档，以及蓝牙底层协议，说明理论上可以支持到同时连接 7 个，但这 7 个能同时正常工作么？貌似不能（三个蓝牙耳机测试的结果）

问题8.如何保证发送数据的完整性

答：做 一个分包发送的操作，保证了数据的完整性。

问题9.如何实现重连机制？

答：自动重连函数被调用之后，会设置一个全局标识为_isAutoConnect=YES，然后判断手机设备的蓝牙是否开启，若开启，则重连扫描外设设备，当扫到上一次连接的蓝牙设备后就会调用连接的代理函数，并停止扫描。

如果手动杀掉APP，那么再次打开APP的时候APP是不会自动连接设备的，但是由于系统蓝牙此时还是与手表连接中的，所以需要重新扫描设备（因为在扫描的代理函数中添加了自动连接的逻辑），经过测试，当扫描到上次连接上的蓝牙外设后就会停止。
方式一: 直接扫描重连

方式二： 通过系统提供的函数retrieveConnectedPeripheralsWithServices

方式三： 通过系统提供的函数retrievePeripheralsWithIdentifiers

问题10：如何获取已经配对过的蓝牙外设？

答：系统一共提供了两个函数来获取已经配对过的蓝牙外设,NSArray *[_centralManager retrieveConnectedPeripheralsWithServices:<#(nonnull NSArray<CBUUID *> *)#>];( CBUUID指的是ServiceUUID)、[_centralManager retrievePeripheralsWithIdentifiers:<#(nonnull NSArray<NSUUID *> *)#>]; 参数是个已连接的ServiceUUID或Identifiers的数组，是个必填项，若传@[]空数组，则返回值是nil。

问题11：开发蓝牙 APP，有什么工具可以协助蓝牙测试？

答：首先测试蓝牙必须时真机，其次安装了蓝牙调试助手或LightBlue等第三方App来调试蓝牙的开发。

问题12：App作为中心设备端，连接到蓝牙设备之后，如何获取外设设备的Mac地址？

答：iOS端是无法直接获取设备的Mac地址，但是可以间接获取，但都需要和硬件工程师进行沟通。

1，将蓝牙外设广播里，提供Mac地址，这样中心设备端在扫描阶段，可以直接读取广播里的值，从而获取到外设设备的Mac地址。
2，可以在外设设备的某个服务的特征中，提供Mac地址，但是前提是要确定是读取哪个特征，UUID是多少。

问题13：为什么两个 iPhone 手机的都打开蓝牙之后，却相互搜不到彼此手机上的同个蓝牙Demo？

答：在蓝牙通信中，分为中心端和设备端。而通常手机蓝牙Demo都处在中心端状态，也就是只能接收广播，而自己没有向周围发送广播。所以两台手机之间一般是无法发现对方的（因为大家都是中心端）

问题14：蓝牙外设设备升级等，大数据传输，耗时操作，数据发送时选择CBCharacteristicWriteWithResponse，会影响总交互时间， 使用CBCharacteristicWriteWithoutResponse又回出现丢包的现象。

答：

[self.peripheral writeValue:data forCharacteristic:self.characteristic type:CBCharacteristicWriteWithResponse];
[self.peripheral writeValue:data forCharacteristic:self.characteristic type:CBCharacteristicWriteWithResponse];

 如果交互的总时间我们不能接受，可以选用CBCharacteristicWriteWithoutResponse模式每包20字节循环发，但注意每发12包延迟100毫秒（经验值，12包240字节小于250），即可加快大数据传输速率。



