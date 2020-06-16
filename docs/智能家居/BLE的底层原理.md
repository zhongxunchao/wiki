我已经翻译了一篇文章：[BLE技术介绍](https://zhongxunchao.github.io/wiki/%E7%BF%BB%E8%AF%91/BLE%E6%8A%80%E6%9C%AF%E4%BB%8B%E7%BB%8D/) 
里面介绍了BLE在GAP和GATT上的实现，这篇文章希望能够系统探索BLE从应用层到物理层的基本实现，以及其他需要补充的一些内容。最权威的资料来源于[Bluetooth Low Energy Software Developer’s Guide](https://www.ti.com/lit/ug/swru271i/swru271i.pdf?ts=1592264196467&ref_url=https%253A%252F%252Fwww.ti.com%252Ftool%252FBLE-STACK)
##蓝牙协议栈描述
协议栈的层次如下：
![BLE Protocol Stack](./images/image1.png)
那么这里涉及到2个重要的概念：host和controller。 一般而言，controller是指跑在蓝牙模块上，更底层的那部分协议栈，而host是指跑在AP芯片，更接近应用层的部分。一般的移动应用开发只需要和host打交道，而host和controller可以通过HCI协议进行通信。其他的协议栈方案请参见引用文档。
>HCI协议是一个传输层协议，位于蓝牙高层协议和低层协议之间，提供了对基带控制器和链路管理器的命令以及访问蓝牙硬件的统一接口,它是我们实现自己的蓝牙设备要接触的第一个蓝牙协议,起着承上启下的作用。参见：[蓝牙HCI协议](https://blog.csdn.net/hushiganghu/article/details/61919261)

下面将对各层协议进行一定的描述。

##物理层
蓝牙是无线通信的一种，所以其通信介质是某个频率范围的频带资源.
物理层是通过BLE射频信号实现的，本身只负责发送和接收数据，
蓝牙是一种工业和民用，所以使用了免费的ISM频段, 在2400MHz -2480MHz范围。为了同时支持多个设备，将频段分为40份，每份的带宽为2MHZ，也就是RD Channel. 可以看到，蓝牙所分配的频段都是在2.4GHZ的。
BLE使用跳频技术来解决频段拥挤问题,这里不多展开。
>BLE广播信道一共有3个，分别为37、38和39信道，频点分布分别为2402MHz、2426MHz和2480MHz，这3个信道用于发送广播信息，广播信道分散在距离较远的频段上，过度的集中会导致如果该频段受干扰严重可能广播就无法进行的情况，分散的目的是为了增加容错率。剩下的37个频点用于通信，即连接以后用于数据交流。

##链路层（Link Layer）



##引用文档
* [Bluetooth Low Energy Software Developer’s Guide](https://www.ti.com/lit/ug/swru271i/swru271i.pdf?ts=1592264196467&ref_url=https%253A%252F%252Fwww.ti.com%252Ftool%252FBLE-STACK)
* [蓝牙HCI协议](https://blog.csdn.net/hushiganghu/article/details/61919261)
* [蓝牙协议栈方案](https://www.cnblogs.com/iini/p/8834970.html)
* [BLE从底层到上层](https://blog.csdn.net/wuneiqiang/article/details/100569072)