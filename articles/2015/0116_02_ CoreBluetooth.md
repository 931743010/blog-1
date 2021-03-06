#iOS蓝牙4.0协议简单介绍

2015-01-16

iOS开发蓝牙4.0的框架是CoreBluetooth，本文主要介绍CoreBluetooth的使用，关于本文中的代码片段大多来自github上的一个demo，地址是<a title="myz1104/Bluetooth" href="https://github.com/myz1104/Bluetooth">myz1104/Bluetooth</a>。

在CoreBluetooth中有两个主要的部分,Central和Peripheral，有一点类似Client Server。CBPeripheralManager 作为周边设备是服务器。CBCentralManager作为中心设备是客户端。所有可用的iOS设备可以作为周边（Peripheral）也可以作为中央（Central），但不可以同时既是周边也是中央。


一般手机是客户端， 设备（比如手环）是服务器，因为是手机去连接手环这个服务器。周边（Peripheral）是生成或者保存了数据的设备，中央（Central）是使用这些数据的设备。你可以认为周边是一个广播数据的设备，他广播到外部世界说他这儿有数据，并且也说明了能提供的服务。另一边，中央开始扫描附近有没有服务，如果中央发现了想要的服务，然后中央就会请求连接周边，一旦连接建立成功，两个设备之间就开始交换传输数据了。

除了中央和周边，我们还要考虑他俩交换的数据结构。这些数据在服务中被结构化，每个服务由不同的特征（Characteristics）组成，特征是包含一个单一逻辑值的属性类型。

<h3>Peripheral的实现步骤</h3>

首先是创建一个周边

<pre lang="objc" line="1">_peripheralManager = [[CBPeripheralManager alloc]initWithDelegate:self queue:nil];</pre>
接下来它就会响应代理的peripheralManagerDidUpdateState方法，可以获得peripheral的状态等信息，
<pre lang="objc" line="1">- (void)peripheralManagerDidUpdateState:(CBPeripheralManager *)peripheral
{
    switch (peripheral.state)
    {
        case CBPeripheralManagerStatePoweredOn:
        {
            [self setupService];
        }
            break;

        default:
        {
            NSLog(@"Peripheral Manager did change state");
        }
            break;
    }
}
</pre>
当发现周边设备的蓝牙是可以的时候，这就需要去准备你需要广播给其他中央设备的服务和特征了，这里通过调用setupService方法来实现。
每一个服务和特征都需要用一个UUID（unique identifier）去标识，UUID是一个16bit或者128bit的值。如果你要创建你的中央-周边App，你需要创建你自己的128bit的UUID。你必须要确定你自己的UUID不能和其他已经存在的服务冲突。如果你正要创建一个自己的设备，需要实现标准委员会需求的UUID；如果你只是创建一个中央-周边App，我建议你打开Mac OS X的Terminal.app，用uuidgen命令生成一个128bit的UUID。你应该用该命令两次，生成两个UUID，一个是给服务用的，一个是给特征用的。然后，你需要添加他们到中央和周边App中。现在，在view controller的实现之前，我们添加以下的代码：
<pre lang="objc" line="1">static NSString * const kServiceUUID = @"1C85D7B7-17FA-4362-82CF-85DD0B76A9A5";
static NSString * const kCharacteristicUUID = @"7E887E40-95DE-40D6-9AA0-36EDE2BAE253";
</pre>
下面就是setupService方法
<pre lang="objc" line="1">- (void)setupService
{
    CBUUID *characteristicUUID = [CBUUID UUIDWithString:kCharacteristicUUID];

    self.customCharacteristic = [[CBMutableCharacteristic alloc] initWithType:characteristicUUID properties:CBCharacteristicPropertyNotify value:nil permissions:CBAttributePermissionsReadable];

    CBUUID *serviceUUID = [CBUUID UUIDWithString:kServiceUUID];

    self.customService = [[CBMutableService alloc] initWithType:serviceUUID primary:YES];
    [self.customService setCharacteristics:@[self.customCharacteristic]];
    [self.peripheralManager addService:self.customService];


}
</pre>
当调用了CBPeripheralManager的addService方法后，这里就会响应CBPeripheralManagerDelegate的- (void)peripheralManager:(CBPeripheralManager *)peripheral didAddService:(CBService *)service error:(NSError *)error方法。这个时候就可以开始广播我们刚刚创建的服务了。
<pre lang="objc" line="1">- (void)peripheralManager:(CBPeripheralManager *)peripheral didAddService:(CBService *)service error:(NSError *)error
{
    if (error == nil)
    {
        [self.peripheralManager startAdvertising:@{ CBAdvertisementDataLocalNameKey : @"ICServer", CBAdvertisementDataServiceUUIDsKey : @[[CBUUID UUIDWithString:kServiceUUID]] }];
    }
}</pre>
当然到这里，你已经做完了peripheralManager的工作了，中央设备已经可以接受到你的服务了。不过这是静止的数据，你还可以调用- (BOOL)updateValue:(NSData *)value forCharacteristic:(CBMutableCharacteristic *)characteristic onSubscribedCentrals:(NSArray *)centrals方法可以给中央生成动态数据的地方。
<pre lang="objc" line="1">- (void)sendToSubscribers:(NSData *)data {
  if (self.peripheral.state != CBPeripheralManagerStatePoweredOn) {
    LXCBLog(@"sendToSubscribers: peripheral not ready for sending state: %d", self.peripheral.state);
    return;
  }

  BOOL success = [self.peripheral updateValue:data
                            forCharacteristic:self.characteristic
                         onSubscribedCentrals:nil];
  if (!success) {
    LXCBLog(@"Failed to send data, buffering data for retry once ready.");
    self.pendingData = data;
    return;
  }
}</pre>
central订阅了characteristic的值，当更新值的时候peripheral会调用updateValue: forCharacteristic: onSubscribedCentrals:(NSArray*)centrals去为数组里面的centrals更新对应characteristic的值，在更新过后peripheral为每一个central走一遍下面的代理方法
<pre lang="objc" line="1">- (void)peripheralManager:(CBPeripheralManager *)peripheral central:(CBCentral *)central didSubscribeToCharacteristic:(CBCharacteristic *)characteristic</pre>
peripheral接受到一个读或者写的请求时，会响应以下两个代理方法
<pre lang="objc" line="1">- (void)peripheralManager:(CBPeripheralManager *)peripheral didReceiveReadRequest:(CBATTRequest *)request

- (void)peripheralManager:(CBPeripheralManager *)peripheral didReceiveWriteRequests:(NSArray *)requests</pre>
那么现在peripheral就已经创建好了。
<h3>创建一个中央</h3>
创建中央并且连接周边
现在，我们已经有了一个周边，让我们创建我们的中央。中央就是那个处理周边发送来的数据的设备。
<pre lang="objc" line="1">self.manager = [[CBCentralManager alloc] initWithDelegate:self queue:dispatch_get_main_queue()];</pre>
当Central Manager被初始化，我们要检查它的状态，以检查运行这个App的设备是不是支持BLE。实现CBCentralManagerDelegate的代理方法：
<pre lang="objc" line="1">- (void)centralManagerDidUpdateState:(CBCentralManager *)central
{
    switch (central.state)
    {
        case CBCentralManagerStatePoweredOn:
        {
            [self.manager scanForPeripheralsWithServices:@[ [CBUUID UUIDWithString:kServiceUUID]]
                                                 options:@{CBCentralManagerScanOptionAllowDuplicatesKey : @YES }];
        }
            break;
        default:
        {
            NSLog(@"Central Manager did change state");
        }
            break;
    }
}</pre>
当app的设备是支持蓝牙的时候，需要调用CBCentralManager实例的- (void)scanForPeripheralsWithServices:(NSArray *)serviceUUIDs options:(NSDictionary *)options方法，用来寻找一个指定的服务的peripheral。一旦一个周边在寻找的时候被发现，中央的代理会收到以下回调：
<pre lang="objc" line="1">- (void)centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)peripheral advertisementData:(NSDictionary *)advertisementData RSSI:(NSNumber *)RSSI
{

    NSString *UUID = [peripheral.identifier UUIDString];
    NSString *UUID1 = CFBridgingRelease(CFUUIDCreateString(NULL, peripheral.UUID));
    NSLog(@"----发现外设----%@%@", UUID,UUID1);
    [self.manager stopScan];

    if (self.peripheral != peripheral)
    {
        self.peripheral = peripheral;
        NSLog(@"Connecting to peripheral %@", peripheral);
        [self.manager connectPeripheral:peripheral options:nil];
    }
}</pre>
这个时候一个附带着广播数据和信号质量(RSSI-Received Signal Strength Indicator)的周边被发现。这是一个很酷的参数，知道了信号质量，你可以用它去判断远近。任何广播、扫描的响应数据保存在advertisementData 中，可以通过CBAdvertisementData 来访问它。
这个时候你用可以连接这个周边设备了，
<pre lang="objc" line="1">[self.manager connectPeripheral:peripheral options:nil];</pre>
它会响应下面的代理方法，
<pre lang="objc" line="1">- (void)centralManager:(CBCentralManager *)central didConnectPeripheral:(CBPeripheral *)peripheral
{
    NSLog(@"----成功连接外设----");
    [self.peripheral setDelegate:self];
    [self.peripheral discoverServices:@[ [CBUUID UUIDWithString:kServiceUUID]]];
}</pre>
访问周边的服务
上面的CBCentralManagerDelegate代理会返回CBPeripheral实例，它的- (void)discoverServices:(NSArray *)serviceUUIDs方法就是访问周边的服务了，这个方法会响应CBPeripheralDelegate的方法。
<pre lang="objc" line="1">- (void)peripheral:(CBPeripheral *)aPeripheral didDiscoverServices:(NSError *)error
{
    NSLog(@"----didDiscoverServices----Error:%@",error);
    if (error)
    {
        NSLog(@"Error discovering service: %@", [error localizedDescription]);
        [self cleanup];
        return;
    }

    for (CBService *service in aPeripheral.services)
    {
        NSLog(@"Service found with UUID: %@", service.UUID);
        if ([service.UUID isEqual:[CBUUID UUIDWithString:kServiceUUID]])
        {
            [self.peripheral discoverCharacteristics:@[[CBUUID UUIDWithString:kCharacteristicUUID],[CBUUID UUIDWithString:kWrriteCharacteristicUUID]] forService:service];
        }
    }
}</pre>
在上面的方法中如果没有error，可以调用discoverCharacteristics方法请求周边去寻找它的服务所列出的特征，它会响应下面的方法
<pre lang="objc" line="1">- (void)peripheral:(CBPeripheral *)peripheral didDiscoverCharacteristicsForService:(CBService *)service error:(NSError *)error
{
    if (error)
    {
        NSLog(@"Error discovering characteristic: %@", [error localizedDescription]);
        return;
    }
    if ([service.UUID isEqual:[CBUUID UUIDWithString:kServiceUUID]])
    {
        for (CBCharacteristic *characteristic in service.characteristics)
        {
            NSLog(@"----didDiscoverCharacteristicsForService---%@",characteristic);
            if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:kCharacteristicUUID]])
            {
                [peripheral readValueForCharacteristic:characteristic];
                [peripheral setNotifyValue:YES forCharacteristic:characteristic];
            }

            if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:kWrriteCharacteristicUUID]])
            {
                writeCharacteristic = characteristic;
            }

        }
    }
}</pre>
这个时候peripheral可以调用两个方法，
[peripheral readValueForCharacteristic:characteristic]这个是读特征值的，会响应- (void)peripheral:(CBPeripheral *)peripheral didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error；

[peripheral setNotifyValue:YES forCharacteristic:characteristic];会响应- (void)peripheral:(CBPeripheral *)peripheral didUpdateNotificationStateForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error；
<pre lang="objc" line="1">- (void)peripheral:(CBPeripheral *)peripheral didUpdateNotificationStateForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
{
    if (error)
    {
        NSLog(@"Error changing notification state: %@", error.localizedDescription);
    }

    // Exits if it's not the transfer characteristic
    if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:kCharacteristicUUID]] )
    {
        // Notification has started
        if (characteristic.isNotifying)
        {
            NSLog(@"Notification began on %@", characteristic);
            [peripheral readValueForCharacteristic:characteristic];
        }
        else
        { // Notification has stopped
            // so disconnect from the peripheral
            NSLog(@"Notification stopped on %@.  Disconnecting", characteristic);
            [self.manager cancelPeripheralConnection:self.peripheral];
        }
    }
}

- (void)peripheral:(CBPeripheral *)peripheral didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
{
    NSLog(@"----Value---%@",characteristic.value);
    if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:kCharacteristicUUID]])
    {

        if (writeCharacteristic)
        {
            Byte ACkValue[3] = {0};
            ACkValue[0] = 0xe0; ACkValue[1] = 0x00; ACkValue[2] = ACkValue[0] + ACkValue[1];
            NSData *data = [NSData dataWithBytes:&amp;ACkValue length:sizeof(ACkValue)];
            [self.peripheral writeValue:data
                      forCharacteristic:writeCharacteristic
                                   type:CBCharacteristicWriteWithoutResponse];
        }
    }
}</pre>
在上面的方法中，- (void)writeValue:(NSData *)data forCharacteristic:(CBCharacteristic *)characteristic type:(CBCharacteristicWriteType)type是一个对周边设备写数据的方法，它会响应下面的方法
<pre lang="objc" line="1">- (void)peripheral:(CBPeripheral *)peripheral didWriteValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
{
    NSLog(@"---didWriteValueForCharacteristic-----");
    if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:kWrriteCharacteristicUUID]])
    {
        NSLog(@"----value更新----");

    }
}</pre>
这样，中央设备也实现了读写数据的功能了。

另外，github上有一个封装的第三方开源蓝牙框架，地址是<a title="kickingvegas/YmsCoreBluetooth" href="https://github.com/kickingvegas/YmsCoreBluetooth">kickingvegas/YmsCoreBluetooth</a>
