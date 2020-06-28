CoreBluetooth是iOS系统提供的关于BLE（低功耗蓝牙）交互的框架，希望这篇文章能够带你看一看CoreBluetooth框架是如何构建的，以及一些可能忽视的细节。
在收看这篇文章之前，希望你对GATT/GAP协议已经有足够的了解；如果你是BLE的初学者，可以先看看我的翻译：
[BLE技术介绍](https://zhongxunchao.github.io/wiki/%E7%BF%BB%E8%AF%91/BLE%E6%8A%80%E6%9C%AF%E4%BB%8B%E7%BB%8D/)

##一般的BLE实现
网上已经有很多iOS上BLE快速集成的方案,我找到了其中一篇：
[iOS BLE开发](https://www.jianshu.com/p/244668e2c959)
可以看到，在一般的BLE开发中，app都是作为center设备，也会用到2个重要的类：CBCentralManager和CBPeripheral.
###1. 初始化CBCentralManager并且设置代理
```
    self.centerManager = [[CBCentralManager alloc]initWithDelegate:self queue:nil];
    self.centerManager.delegate  = self;
```
###2. 开始扫描设备
```
 [self.centerManager scanForPeripheralsWithServices:nil options:@{CBCentralManagerScanOptionAllowDuplicatesKey:[NSNumber numberWithInt:1],CBCentralManagerOptionShowPowerAlertKey:[NSNumber numberWithInt:1]}];
```
###3. 获取设备列表
在CBCentralDelegate的一个方法里获取到CBPeripheral.
```
-(void)centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)peripheral advertisementData:(NSDictionary<NSString *,id> *)advertisementData RSSI:(NSNumber *)RSSI{}
```
每当发现新设备就会进入上面的代理方法，一般而言可以新设备放到一个数组，记得在适当的时机停止扫描。

###4. 连接设备
从上面获得的设备列表中选择一个CBPeripheral对象，并进行连接。
```
[self.centerManager connectPeripheral:peripheral options:nil];
```
###5. 发现设备的CBService
```
// 连接设备成功后，发现相关的CBService
- (void)centralManager:(CBCentralManager *)central didConnectPeripheral:(CBPeripheral *)peripheral {
    peripheral.delegate = self;
    [peripheral discoverServices:@[[self.class serviceUUID]]];
}
```
###6. 发现CBCharateristic
这里要注意到一个区别，我们在扫描和连接的过程中，都使用了CBCentralManager对象或者代理方法。但是当设备连接成功后，接下来的CBService和CBCharacteristic将与Centeral设备无关，而是通过CBPeripheral的代理方法继续推进。因此在第5步中我们设置了CBPhripheral的代理对象。
接下来的推进将以CBPhripheralDelegate为主。在发现CBService成功后，将继续发现CBService包含的CBCharacteristic.
```
-(void)peripheral:(CBPeripheral *)peripheral didDiscoverServices:(NSError *)error {
        for (CBService *service in peripheral.services)
    {
        if ([service.UUID isEqual:[self.class serviceUUID]])
        {
            [_peripheral discoverCharacteristics:@[[self.class receiveDataCharacteristicUUID], [self.class sendDataToBandCharacteristicUUID]] forService:service];
        }
    }
}
```
###7. 发现CBCharacteristic成功的回调
当发现BLE特性成功之后，这个连接就成功建立起来了。同时我们要将这个CBCharacteristic保存下来，因为它将是我们读写数据的关键。
这里有一个小小的细节，CoreBluetooth并没有直接在代理方法中将CBCharacteristic暴露出来，因为CBCharacteristic是CBService的属性，所以可以从CBService中获得。
我对iOS这种封装方式持保留意见，一个CBService可以包含多个CBCharacteristic,但是同时CBChaacteritic中也会指明所属的CBService, 因此发现CBCharacteristic的回调直接使用CBCharacteristic可能更加合理，也避免了开发者必须去CBService中进行for循环以获得相应的特性。
```
-(void)peripheral:(CBPeripheral *)peripheral didDiscoverCharacteristicsForService:(CBService *)service error:(NSError *)error
{
    for (CBCharacteristic *characteristic in service.characteristics)
    {
        if ([characteristic.UUID isEqual:[self.class receiveDataCharacteristicUUID]])
        {
            _receiveDataFramBandCharacteristic = characteristic;
            [_peripheral setNotifyValue:YES forCharacteristic:_receiveDataFramBandCharacteristic];
            
        }
        else
        {
           _sendDataToBandCharacteristic = characteristic;
        }
    }
}
```
###8. 接收设备的数据
事实上，在获取到相应的CBCharacteristic之后，就可以轻松的向设备发送数据了。前提是我们已经将CBPeripheral和CBCharacteristic都保存了下来。
```
[peripheral writeValue:data forCharacteristic:characteristic type:CBCharacteristicWriteWithoutResponse];
```
但是如何接收设备的数据呢？注意前一步中我们开启了对特性的数据监听，这样才能进入下面的代理方法：
```
- (void) peripheral:(CBPeripheral *)peripheral didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
{ 
    NSLog(@"Did update value for characteristicValue: %@\n", characteristic.value);
}
```

这样我们已经完成了BLE交互的所有必须步骤：扫描设备，建立BLE连接，发送和接收数据等。
当然这种很繁琐的代理方法必然不是大多数开发者喜欢的，使用起来非常不方便。比如要连接一个已经知道Service和Characteristic的UUID的设备，我不得不实现很多个代理方法。或者说部分Service是加密的，连接之前必须经过认证，那么后期的连接订阅就非常麻烦。所以后面我也会介绍一些BLE框架的实现思路，以及自己构思的js的BLE库的实现。

##CoreBluetooth的框架概览
![CoreBluetooth类图](../../out/UML/CoreBluetooth/CoreBluetooth.png)
上面是我绘制的CoreBluetooth的类图，基本包含了所有用到的类。当我绘制完成这张图的时候，便发现了很多知识的漏洞，定期的梳理是非常有必要的。
### CBUUID
在GATT协议中，Service, Characteristic, Descriptor都有这着相应的UUID，虽然它只是一个2个（官方）或16个（自定义）字节的数据，但是把它抽象出来是很有必要的。它是服务或特性的唯一ID.
### CBAttribute
Attribute（属性）是ATT协议的核心。它是一种数据结构，包含Handle, UUID， Value等信息。
>属性句柄（Attribute Handle）是一个2字节的十六进制码，起始于0x0001，在系统初始化时候，各个属性的句柄逐步加一，最大不超过0xFFFF。这个跟软件编程中的句柄的概念类似，就是某个属性值的查询地址。
属性类型（Attribute Type）用以区分当前属性是服务项或是特征值等，它用UUID来表示。UUID（universally unique identifier，通用唯一识别码）是一个软件构建标准，并非BLE独有的概念，一个合法的UUID，一定是随机的、全球唯一的，不应该出现两个相同的UUID（出现了，就说明它们俩是同一个UUID）。标准的UUID是一串16字节十六进制字符串，如f6257d37-34e5-41dd-8f40-e308210498b4，在网上可以方便的生成一个UUID。
属性值（Attribute Value）是存放数据的地方。如果是服务项或者特征值声明，该数据为UUID等信息，如果是普通的特征值，该数据则是用户的数据。

但是CoreBluetooth在CBAttribute中只暴露了UUID信息，或许是认为Handle和Value对开发者并不是必须用到的，所以没有作为CBAttribute的property.

###CBPeer
CBPeer是对等网络（P2P）中设备的抽象，在BLE中设备与设备之间并不是对等的，天然的划分为central设备和peripheral, 或者client和server。但是对于这种设备的抽象是很有必要的，它可以描述central和peripheral的一些共性。
在CoreBluetooth中，CBPeer只暴露了一个CBUUID作为设备的Identifier, 所以它的接口和CBAttribute是非常像的。这也反映了在软件项目中，概念的清晰比代码冗余更加重要。

###CBService, CBCharacteristic和CBDescriptor
由于Service，Characteristic和Descriptor在GATT协议中都有着明确的定义，因此是易于理解的。
每个Service包含一个或多个Characteristic, 每个Characteristic包含一个或者多个Descriptor.
这三个类都继承于CBAttribute, CBService和CBCharacteristic是互相关联的：CBService中包含一个CBCharacteristic的数组，CBCharacteristic也有一个CBService指向所属的服务。同样的道理，CBCharacteristic和CBDescriptor也是互相关联的。
>关于Descriptor
  一个characteristic包含三种条目：characteristic 声明(Declaration)，characteristic的值(Value)，以及characteristic的描述符(Descriptor)（可以有多个描述符）.Descriptor描述了数据的额外信息，比如温度的单位是什么。CCCD是一种特殊的Descriptor，可以用来打开或关闭特性的Notify功能。

###CBPeripheral和CBPeripheralDelegate
对于一个外设Peripheral而言, 大部分CBPeripheral或者CBPeripheralDelegate里面的方法都是不言而喻的。外设可以主动调用的方法在CBPeripheral的实例方法里，回调的方法在CBPeripheralDelegate里。
我们先看看CBPeripheral中容易理解的方法或属性：
```
// CBPeripheralDelegate代理对象.
@property(weak, nonatomic, nullable) id<CBPeripheralDelegate> delegate;
// 外设名称
@property(retain, readonly, nullable) NSString *name;
// 外设的RSSI强度
@property(retain, readonly, nullable) NSNumber *RSSI;
// 外设的连接状态
@property(readonly) CBPeripheralState state;
// 是否可以发送无响应的消息。
@property(readonly) BOOL canSendWriteWithoutResponse;
// 外设包含的所有Service，而Characteristic包含在Service内。
@property(retain, readonly, nullable) NSArray<CBService *> *services;
// 在设备已经建立连接时，主动读取RSSI值
- (void)readRSSI;
// 发现设备的Services。
- (void)discoverServices:(nullable NSArray<CBUUID *> *)serviceUUIDs;
// 发现Service下面包含的Characteristic。
- (void)discoverCharacteristics:(nullable NSArray<CBUUID *> *)characteristicUUIDs forService:(CBService *)service;
// 主动读取Characteristic的值。
- (void)readValueForCharacteristic:(CBCharacteristic *)characteristic;
// 向Characteristic写入数据.
- (void)writeValue:(NSData *)data forCharacteristic:(CBCharacteristic *)characteristic type:(CBCharacteristicWriteType)type;
// 设置监听Characteristic, 同时该方法也会影响该CBCharacteristic的isNotifying属性。
- (void)setNotifyValue:(BOOL)enabled forCharacteristic:(CBCharacteristic *)characteristic;
// 发现Characteristic下的Descriptor。
- (void)discoverDescriptorsForCharacteristic:(CBCharacteristic *)characteristic;
// 主动读取Descriptor的数据。
- (void)readValueForDescriptor:(CBDescriptor *)descriptor;
// 主动更改Descriptor的值。
- (void)writeValue:(NSData *)data forDescriptor:(CBDescriptor *)descriptor;
```
写到这里我发现对每个函数都用中文描述一遍没有必要，因此接下来我将考察我认为比较难懂的地方。

```
@property(readonly) BOOL ancsAuthorized;
```
>ANCS(Apple Notification Center Service)即Apple通知中心服务,是让周边蓝牙设备（手环、手表等）可以通过低功耗蓝牙访问IOS设备（iphone、ipad等）上的各类通知提供的一种简单方便的机制。
注意和APNS进行区分。
**APNS的实现主要在硬件端，一般并不需要App做任何操作。**
参考：[ANCS协议分析](https://www.jianshu.com/p/2ddf76ab85b0)

ANCS不需要App改动任何代码，只要App正常的完成BLE连接即可，硬件会完成（支持）绑定认证的操作。因此该属性是标记该外设是否已经完成ANCS认证。

```
- (void)discoverIncludedServices:(nullable NSArray<CBUUID *> *)includedServiceUUIDs forService:(CBService *)service;
```
在GATT协议中，所有的Service可以分为2种：
- 主要服务（Primary Service）:拥有基本功能的服务,可被其他服务包含,可以通过Primary Service Discovery过程来发现
- 次要服务(Secondary Service):仅用来被Primary/Other Secondary Service、高层协议引用的服务,不会被Central设备发现。
>当一个服务需要用到别的服务里面的某些值的时候，也可以通过«Include»来完成。
然而， «Include»一定是在服务声明之后才能使用的，那么服务声明有两种方式，首要服务可以引用另一个首要服务。首要服务也可以引用一个次要服务，从而使用次要服务公开行为。次要服务可以引用一个次要服务或者首要服务。不过次要服务引用次要服务情况很少，次要引用首要服务就更少了。
参考：[GATT服务构成](https://blog.csdn.net/yk150915/article/details/87618584)

现在我们可以解释该方法的作用：它允许我们去发现某个Service内部的引用的其他Service。

```
- (NSUInteger)maximumWriteValueLengthForType:(CBCharacteristicWriteType)type;
```
看以下这个WriteType的定义,其含义无需多说：
```
typedef NS_ENUM(NSInteger, CBCharacteristicWriteType) {
	CBCharacteristicWriteWithResponse = 0,
	CBCharacteristicWriteWithoutResponse,
};
```
这个函数返回了对特定类型Characteristic的最大数据长度。
但是我认为这个方法里面有3个疑问：
* 1. 对于不同的Characteristic这个可写的length是否可能不同？
* 2. 为什么不同的CBCharacteristicWriteType参数可能得到不同的返回结果？
* 3. 发送和接收的数据的最大长度是否总是一致的？
我查阅了国内外一些资料，并没有发现能够很清晰的解决这些疑问的。

对于第一个疑问，个人的理解是这个可写长度实质上取决于ATT的MTU的限制，那么该最大长度应该对于不同的特性应该是一致的，因为GATT协议处于协议栈的最上层了。同时根据这个解释，发送和接收的数据的最大长度也应该是一致的。
第二个问题, stackoverflow上的一个[回答](https://stackoverflow.com/questions/41289783/what-does-peripheral-maximumwritevaluelengthfortypecbcharacteristicwritewithre)只是说我们应该尽可能避免发送超出这个最大值的数据，同时对于WithResponse类型的特性，可能会返回512. 我将测试对于WithoutResponse类型会返回多少。
 <font color='red'> WARNING: 对于该方法的说明待完善。</font>

 ```
 - (void)openL2CAPChannel:(CBL2CAPPSM)PSM;
 ```
 


##参考资料
* [ANCS协议分析](https://www.jianshu.com/p/2ddf76ab85b0)
* [ANCS Spec](https://link.jianshu.com/?t=https://developer.apple.com/library/content/documentation/CoreBluetooth/Reference/AppleNotificationCenterServiceSpecification/Introduction/Introduction.html#//apple_ref/doc/uid/TP40013460-CH2-SW1)
* [GATT服务构成](https://blog.csdn.net/yk150915/article/details/87618584)

##待解决问题：
* 为什么通过CBPeripheralDelegate来实现.
* CBPeer的存在意义. over
* CBService的isPrimary属性和includedServices. over
* CBCharacterisitc的isNotifying属性。over

