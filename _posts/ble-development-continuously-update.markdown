---
layout: post
title:  "iOS-BLE蓝牙开发持续更新"
date:   2015-06-24 17:00:00
categories: iOS学习
tags: [iOS学习]
---

在写这个博客之前，空余时间抽看了近一个月的文档和Demo，系统给的解释很详细，接口也比较实用，唯独有一点，对于设备的唯一标示，网上众说纷纭，在这里我目前也还没有自己的见解，只是在不断的测试各种情况，亲测同一设备的UUID对于每台iPhone设备都不一样，只能尽量保证设备的唯一性，特别是自动重连的过程，让用户没有感知。<!--more-->我之前也找了很久，发现CBCentralManager和CBPeripheral里边都找不到和Mac地址有关的东西，后来发现一般是外设在Device Information服务中的某个特征返回的。经过与硬件工程师的协商，决定APP端将从这个服务中获取到蓝牙设备以及我的iPhone手机的蓝牙Mac地址，为自动连接的唯一性做准备。

这里经过和硬件工程师的测试，发现设备端在获取手机蓝牙MAC地址的时候，当用户手机重启之后，这个地址也是会随机变化的，也就是说，作为开发者，只有设备的MAC地址能够保持唯一性不变化。

如有疑问的朋友可以先去这里瞅一瞅：[一个关于蓝牙4.0的智能硬件Demo详解](https://yuhanle.github.io/2016/01/10/about-ble-intelligent-hd-demo-detailed/)

* 下面是两台iPhone6连接同一台蓝牙设备的结果：

```
**成功连接**** peripheral: <CBPeripheral: 0x1700f4500, identifier = 50084F69-BA5A-34AC-8A6E-6F0CEADB21CD, name = 555555555588, state = connected> with UUID: <__NSConcreteUUID 0x17003d980> 50084F69-BA5A-34AC-8A6E-6F0CEADB21CD**
****
****
**成功连接**** peripheral: <CBPeripheral: 0x1742e3000, identifier = 55B7D759-0F1E-6271-EA14-BC5A9C9EEEEC, name = 555555555588, state = connected> with UUID: <__NSConcreteUUID 0x174036c00> 55B7D759-0F1E-6271-EA14-BC5A9C9EEEEC**
```

### 进入正题

iOS的蓝牙开发很简单，只要包含一个库，创建CBCentralManager实例，实现代理方法，然后就可以直接和设备进行通信。

![发现附近的特定蓝牙设备](http://7xqhcq.com1.z0.glb.clouddn.com/ble-chixugeng-1.png)


```
#import <CoreBluetooth/CoreBluetooth.h>
```

首先可以定义一些即将使用到的UUID的宏


```
#define kPeripheralName     @"360qws Electric Bike Service" //外围设备名称
#define kServiceUUID        @"7CACEB8B-DFC4-4A40-A942-AAD653D174DC" //服务的UUID
#define kCharacteristicUUID @"282A67B2-8DAB-4577-A42F-C4871A3EEC4F" //特征的UUID
```

如果不是把手机作为中心设备的话，这些没有必要设置。
这里我也没有用到，仅仅是提了一下，具体操作后续添加。

对于生成UUID，大家可以谷歌一下，直接通过mac终端生成32位UUID。

> 1.声明属性


```
@property (weak, nonatomic) IBOutlet UITableView *bluetoothTable;
@property (weak, nonatomic) IBOutlet UITextView *resultTextView;

@property BOOL cbReady;
@property(nonatomic) float batteryValue;
@property (nonatomic, strong) CBCentralManager *manager;
@property (nonatomic, strong) CBPeripheral *peripheral;

@property (strong ,nonatomic) CBCharacteristic *writeCharacteristic;

@property (strong,nonatomic) NSMutableArray *nDevices;
@property (strong,nonatomic) NSMutableArray *nServices;
@property (strong,nonatomic) NSMutableArray *nCharacteristics;
```

> 2.遵守协议(这里我用到了table)


```
@interface ViewController () <CBCentralManagerDelegate, CBPeripheralDelegate, UITableViewDataSource, UITableViewDelegate>
```

> 3.初始化数据


```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    self.manager = [[CBCentralManager alloc] initWithDelegate:self queue:nil];
    
    _cbReady = false;
    _nDevices = [[NSMutableArray alloc]init];
    _nServices = [[NSMutableArray alloc]init];
    _nCharacteristics = [[NSMutableArray alloc]init];
    
    _bluetoothTable.delegate = self;
    _bluetoothTable.dataSource = self;
    
    count = 0;
}
```

> 4.实现蓝牙的协议方法


* （1）检测蓝牙状态


```
//开始查看服务，蓝牙开启
-(void)centralManagerDidUpdateState:(CBCentralManager *)central
{
    switch (central.state) {
        case CBCentralManagerStatePoweredOn:
        {
            [self updateLog:@"蓝牙已打开,请扫描外设"];
            [_activity startAnimating];
            [_manager scanForPeripheralsWithServices:@[[CBUUID UUIDWithString:@"FF15"]]  options:@{CBCentralManagerScanOptionAllowDuplicatesKey : @YES }];
        }
            break;
        case CBCentralManagerStatePoweredOff:
            [self updateLog:@"蓝牙没有打开,请先打开蓝牙"];
            break;
        default:
            break;
    }
}

```

注：[_manager scanForPeripheralsWithServices:@[[CBUUID UUIDWithString:@"FF15"]]  options:@{CBCentralManagerScanOptionAllowDuplicatesKey : @YES }];中间的@[[CBUUID UUIDWithString:@"FF15"]]是为了过滤掉其他设备，可以搜索特定标示的设备。

* （2）检测到外设后，停止扫描，连接设备


```
//查到外设后，停止扫描，连接设备
-(void)centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)peripheral advertisementData:(NSDictionary *)advertisementData RSSI:(NSNumber *)RSSI
{
    [self updateLog:[NSString stringWithFormat:@"已发现 peripheral: %@ rssi: %@, UUID: %@ advertisementData: %@ ", peripheral, RSSI, peripheral.identifier, advertisementData]];
    
    _peripheral = peripheral;
    [_manager connectPeripheral:_peripheral options:nil];
    
    [self.manager stopScan];
    [_activity stopAnimating];
    
    BOOL replace = NO;
    // Match if we have this device from before
    for (int i=0; i < _nDevices.count; i++) {
        CBPeripheral *p = [_nDevices objectAtIndex:i];
        if ([p isEqual:peripheral]) {
            [_nDevices replaceObjectAtIndex:i withObject:peripheral];
            replace = YES;
        }
    }
    if (!replace) {
        [_nDevices addObject:peripheral];
        [_bluetoothTable reloadData];
    }
}
```

* （3）连接外设后的处理


```
//连接外设成功，开始发现服务
- (void)centralManager:(CBCentralManager *)central didConnectPeripheral:(CBPeripheral *)peripheral {
    NSLog(@"%@", [NSString stringWithFormat:@"成功连接 peripheral: %@ with UUID: %@",peripheral,peripheral.identifier]);
    
    [self updateLog:[NSString stringWithFormat:@"成功连接 peripheral: %@ with UUID: %@",peripheral,peripheral.identifier]];
    
    [self.peripheral setDelegate:self];
    [self.peripheral discoverServices:nil];
    [self updateLog:@"扫描服务"];
}
//连接外设失败
-(void)centralManager:(CBCentralManager *)central didFailToConnectPeripheral:(CBPeripheral *)peripheral error:(NSError *)error
{
    NSLog(@"%@",error);
}

-(void)peripheralDidUpdateRSSI:(CBPeripheral *)peripheral error:(NSError *)error
{
    NSLog(@"%s,%@",__PRETTY_FUNCTION__,peripheral);
    int rssi = abs([peripheral.RSSI intValue]);
    CGFloat ci = (rssi - 49) / (10 * 4.);
    NSString *length = [NSString stringWithFormat:@"发现BLT4.0热点:%@,距离:%.1fm",_peripheral,pow(10,ci)];
    [self updateLog:[NSString stringWithFormat:@"距离：%@", length]];
}
```

* （4）发现服务和搜索到的Characteristice


```
//已发现服务
-(void) peripheral:(CBPeripheral *)peripheral didDiscoverServices:(NSError *)error{
    
    [self updateLog:@"发现服务."];
    int i=0;
    for (CBService *s in peripheral.services) {
        [self.nServices addObject:s];
    }
    for (CBService *s in peripheral.services) {
        [self updateLog:[NSString stringWithFormat:@"%d :服务 UUID: %@(%@)",i,s.UUID.data,s.UUID]];
        i++;
        [peripheral discoverCharacteristics:nil forService:s];
        
        if ([s.UUID isEqual:[CBUUID UUIDWithString:@"FF15"]]) {
            BOOL replace = NO;
            // Match if we have this device from before
            for (int i=0; i < _nDevices.count; i++) {
                CBPeripheral *p = [_nDevices objectAtIndex:i];
                if ([p isEqual:peripheral]) {
                    [_nDevices replaceObjectAtIndex:i withObject:peripheral];
                    replace = YES;
                }
            }
            if (!replace) {
                [_nDevices addObject:peripheral];
                [_bluetoothTable reloadData];
            }
        }
    }
}

//已搜索到Characteristics
-(void) peripheral:(CBPeripheral *)peripheral didDiscoverCharacteristicsForService:(CBService *)service error:(NSError *)error{
    [self updateLog:[NSString stringWithFormat:@"发现特征的服务:%@ (%@)",service.UUID.data ,service.UUID]];
    
    for (CBCharacteristic *c in service.characteristics) {
        [self updateLog:[NSString stringWithFormat:@"特征 UUID: %@ (%@)",c.UUID.data,c.UUID]];
        
        if ([c.UUID isEqual:[CBUUID UUIDWithString:@"FF01"]]) {
            _writeCharacteristic = c;
        }
        
        if ([c.UUID isEqual:[CBUUID UUIDWithString:@"FF02"]]) {
            [_peripheral readValueForCharacteristic:c];
            [_peripheral setNotifyValue:YES forCharacteristic:c];
        }
        
        if ([c.UUID isEqual:[CBUUID UUIDWithString:@"FF04"]]) {
            [_peripheral readValueForCharacteristic:c];
        }
        
        if ([c.UUID isEqual:[CBUUID UUIDWithString:@"FF05"]]) {
            [_peripheral readValueForCharacteristic:c];
            [_peripheral setNotifyValue:YES forCharacteristic:c];
        }
        
        if ([c.UUID isEqual:[CBUUID UUIDWithString:@"FFA1"]]) {
            [_peripheral readRSSI];
        }
        
        [_nCharacteristics addObject:c];
    }
}

- (void)centralManager:(CBCentralManager *)central didDisconnectPeripheral:(CBPeripheral *)peripheral error:(NSError *)error {
    [self updateLog:[NSString stringWithFormat:@"已断开与设备:[%@]的连接", peripheral.name]];
}
```

* （5）获取外设发来的数据


```
//获取外设发来的数据，不论是read和notify,获取数据都是从这个方法中读取。
- (void)peripheral:(CBPeripheral *)peripheral didUpdateValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
{
    if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:@"FF02"]]) {
        NSData * data = characteristic.value;
        Byte * resultByte = (Byte *)[data bytes];
        
        for(int i=0;i<[data length];i++)
            printf("testByteFF02[%d] = %d\n",i,resultByte[i]);
        
        if (resultByte[1] == 0) {
            switch (resultByte[0]) {
                case 3: // 加解锁
                {
                    if (resultByte[2] == 0) {
                        [self updateLog:@"撤防成功!!!"];
                    }else if (resultByte[2] == 1) {
                        [self updateLog:@"设防成功!!!"];
                    }
                }
                    break;
                case 4: // 开坐桶
                {
                    if (resultByte[2] == 0) {
                        [self updateLog:@"关坐桶成功!!!"];
                    }else if (resultByte[2] == 1) {
                        [self updateLog:@"开坐桶成功!!!"];
                    }
                }
                    break;
                case 5: // 锁定电机
                {
                    if (resultByte[2] == 0) {
                        [self updateLog:@"解锁电机控制器成功!!!"];
                    }else if (resultByte[2] == 1) {
                        [self updateLog:@"锁定电机控制器成功!!!"];
                    }
                }
                    break;
                default:
                    break;
            }
        }else if (resultByte[1] == 1) {
            [self updateLog:@"未知错误"];
        }else if (resultByte[1] == 2) {
            [self updateLog:@"鉴权失败"];
        }
    }
    
    if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:@"FF04"]]) {
        NSData * data = characteristic.value;
        Byte * resultByte = (Byte *)[data bytes];
        
        for(int i=0;i<[data length];i++)
            printf("testByteFF04[%d] = %d\n",i,resultByte[i]);
        
        if (resultByte[0] == 0) {
            // 未绑定 -》写鉴权码
            [self updateLog:@"当前车辆未绑定，请鉴权"];
            [self authentication];  // 鉴权
            [self writePassword:nil newPw:nil];
        }else if (resultByte[0] == 1) {
            // 已绑定 -》鉴权
            [self updateLog:@"当前车辆已经绑定，请鉴权"];
            [self writePassword:nil newPw:nil];
        }else if (resultByte[0] == 2) {
            // 允许绑定
            [self updateLog:@"当前车辆允许绑定"];
        }
    }
    
    if ([characteristic.UUID isEqual:[CBUUID UUIDWithString:@"FF05"]]) {
        NSData * data = characteristic.value;
        Byte * resultByte = (Byte *)[data bytes];
        
        for(int i=0;i<[data length];i++)
            printf("testByteFF05[%d] = %d\n",i,resultByte[i]);
        
        if (resultByte[0] == 0) {
            // 设备加解锁状态 0 撤防     1 设防
            [self updateLog:@"当前车辆撤防状态"];
        }else if (resultByte[0] == 1) {
            // 设备加解锁状态 0 撤防     1 设防
            [self updateLog:@"当前车辆设防状态"];
        }
    }
}

//中心读取外设实时数据
- (void)peripheral:(CBPeripheral *)peripheral didUpdateNotificationStateForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error {
    if (error) {
        NSLog(@"Error changing notification state: %@", error.localizedDescription);
    }
    
    // Notification has started
    if (characteristic.isNotifying) {
        [peripheral readValueForCharacteristic:characteristic];
        
    } else { // Notification has stopped
        // so disconnect from the peripheral
        NSLog(@"Notification stopped on %@.  Disconnecting", characteristic);
        [self updateLog:[NSString stringWithFormat:@"Notification stopped on %@.  Disconnecting", characteristic]];
        [self.manager cancelPeripheralConnection:self.peripheral];
    }
}
//用于检测中心向外设写数据是否成功
-(void)peripheral:(CBPeripheral *)peripheral didWriteValueForCharacteristic:(CBCharacteristic *)characteristic error:(NSError *)error
{
    if (error) {
        NSLog(@"=======%@",error.userInfo);
        [self updateLog:[error.userInfo JSONString]];
    }else{
        NSLog(@"发送数据成功");
        [self updateLog:@"发送数据成功"];
    }
    
    /* When a write occurs, need to set off a re-read of the local CBCharacteristic to update its value */
    [peripheral readValueForCharacteristic:characteristic];
}
```

* （6）其他辅助性的


```
#pragma mark - 蓝牙的相关操作
- (IBAction)bluetoothAction:(UIButton *)sender {
    switch (sender.tag) {
        case 201:
        {   // 搜索设备
            [self updateLog:@"正在扫描外设..."];
            [_activity startAnimating];
            [_manager scanForPeripheralsWithServices:@[[CBUUID UUIDWithString:@"FF15"]]  options:@{CBCentralManagerScanOptionAllowDuplicatesKey : @YES }];
            
            double delayInSeconds = 30.0;
            dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
            dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
                [self.manager stopScan];
                [_activity stopAnimating];
                [self updateLog:@"扫描超时,停止扫描"];
            });
        }
            break;
        case 202:
        {   // 连接
            if (_peripheral && _cbReady) {
                [_manager connectPeripheral:_peripheral options:nil];
                _cbReady = NO;
            }
        }
            break;
        case 203:
        {   // 断开
            if (_peripheral && !_cbReady) {
                [_manager cancelPeripheralConnection:_peripheral];
                _cbReady = YES;
            }
        }
            break;
        case 204:
        {   // 暂停搜索
            [self.manager stopScan];
        }
            break;
        case 211:
        {   // 车辆上锁
            [self lock];
        }
            break;
        case 212:
        {   // 车辆解锁
            [self unLock];
        }
            break;
        case 213:
        {   // 开启坐桶
            [self open];
        }
            break;
        case 214:
        {   // 立即寻车
            [self find];
        }
            break;
        default:
            break;
    }
}

-(NSOperationQueue *)queue {
    if (!_queue) {  // 请求队列
        self.queue = [[NSOperationQueue alloc]init];
        [self.queue setMaxConcurrentOperationCount:1];
    }
    
    return _queue;
}

#pragma mark - 命令
#pragma mark - 鉴权
-(void)authentication {
    Byte byte[] = {1, 1, 2, 3, 4, 5, 6, 7, 8};
    if (_peripheral.state == CBPeripheralStateConnected) {
        [_peripheral writeValue:[NSData dataWithBytes:byte length:9] forCharacteristic:_writeCharacteristic type:CBCharacteristicWriteWithoutResponse];
    }
}

#pragma mark - 写密码 
-(void)writePassword:(NSString *)initialPw newPw:(NSString *)newPw {
    Byte byte[] = {2, 0, 0, 0, 0, 0, 0, 0, 0, 1, 2, 3, 4, 5, 6, 7, 8};
    if (_peripheral.state == CBPeripheralStateConnected) {
        [_peripheral writeValue:[NSData dataWithBytes:byte length:17] forCharacteristic:_writeCharacteristic type:CBCharacteristicWriteWithoutResponse];
    }
}

#pragma mark - 车辆上锁
-(void)lock {
    Byte byte[] = {3, 1};
    if (_peripheral.state == CBPeripheralStateConnected) {
        [_peripheral writeValue:[NSData dataWithBytes:byte length:2] forCharacteristic:_writeCharacteristic type:CBCharacteristicWriteWithoutResponse];
    }
}

#pragma mark - 车辆解锁
-(void)unLock {
    Byte byte[] = {3, 0};
    if (_peripheral.state == CBPeripheralStateConnected) {
        [_peripheral writeValue:[NSData dataWithBytes:byte length:2] forCharacteristic:_writeCharacteristic type:CBCharacteristicWriteWithoutResponse];
    }
}

#pragma mark - 开启坐桶
-(void)open {
    Byte byte[] = {4, 1};
    if (_peripheral.state == CBPeripheralStateConnected) {
        [_peripheral writeValue:[NSData dataWithBytes:byte length:2] forCharacteristic:_writeCharacteristic type:CBCharacteristicWriteWithoutResponse];
    }
}

#pragma mark - 立即寻车
-(void)find {
    Byte byte[] = {7};
    if (_peripheral.state == CBPeripheralStateConnected) {
        [_peripheral writeValue:[NSData dataWithBytes:byte length:1] forCharacteristic:_writeCharacteristic type:CBCharacteristicWriteWithoutResponse];
    }
}

-(NSData *)hexString:(NSString *)hexString {
    int j=0;
    Byte bytes[20];
    ///3ds key的Byte 数组， 128位
    for(int i=0; i<[hexString length]; i++)
    {
        int int_ch;  /// 两位16进制数转化后的10进制数
        
        unichar hex_char1 = [hexString characterAtIndex:i]; ////两位16进制数中的第一位(高位*16)
        int int_ch1;
        if(hex_char1 >= '0' && hex_char1 <='9')
            int_ch1 = (hex_char1-48)*16;   //// 0 的Ascll - 48
        else if(hex_char1 >= 'A' && hex_char1 <='F')
            int_ch1 = (hex_char1-55)*16; //// A 的Ascll - 65
        else
            int_ch1 = (hex_char1-87)*16; //// a 的Ascll - 97
        i++;
        
        unichar hex_char2 = [hexString characterAtIndex:i]; ///两位16进制数中的第二位(低位)
        int int_ch2;
        if(hex_char2 >= '0' && hex_char2 <='9')
            int_ch2 = (hex_char2-48); //// 0 的Ascll - 48
        else if(hex_char1 >= 'A' && hex_char1 <='F')
            int_ch2 = hex_char2-55; //// A 的Ascll - 65
        else
            int_ch2 = hex_char2-87; //// a 的Ascll - 97
        
        int_ch = int_ch1+int_ch2;
        NSLog(@"int_ch=%d",int_ch);
        bytes[j] = int_ch;  ///将转化后的数放入Byte数组里
        j++;
    }
    
    NSData *newData = [[NSData alloc] initWithBytes:bytes length:20];
    
    return newData;
}
```

> 在和硬件之间的数据发送和接受，用的都是byte数组。最后，添加一个存储已连接过得设备


```
- (void) addSavedDevice:(CFUUIDRef)uuid {
    NSArray *storedDevices = [[NSUserDefaults standardUserDefaults] arrayForKey:@"StoredDevices"];
     NSMutableArray *newDevices = nil;
    CFStringRef uuidString = NULL;
    if (![storedDevices isKindOfClass:[NSArray class]]) {
        NSLog(@"Can't find/create an array to store the uuid");
        return;
    }
    
    newDevices = [NSMutableArray arrayWithArray:storedDevices];
    uuidString = CFUUIDCreateString(NULL, uuid);
    if (uuidString) {
        [newDevices addObject:(__bridge NSString*)uuidString];
        CFRelease(uuidString);
    }
    
    /* Store */
    [[NSUserDefaults standardUserDefaults] setObject:newDevices forKey:@"StoredDevices"];
    [[NSUserDefaults standardUserDefaults] synchronize];
}

- (void) loadSavedDevices {
    NSArray *storedDevices = [[NSUserDefaults standardUserDefaults] arrayForKey:@"StoredDevices"];
    if (![storedDevices isKindOfClass:[NSArray class]]) {
         NSLog(@"No stored array to load");
        return;
    }
    
    for (id deviceUUIDString in storedDevices) {
        if (![deviceUUIDString isKindOfClass:[NSString class]]) continue;
        
        CFUUIDRef uuid = CFUUIDCreateFromString(NULL, (CFStringRef)deviceUUIDString);
        if (!uuid) continue;
        [CBCentralManager retrievePeripherals:[NSArray arrayWithObject:(__bridge id)uuid]];
        CFRelease(uuid);
    }
}

//And the delegate function for the Retrieve Peripheral
- (void) centralManager:(CBCentralManager *)central didRetrieveConnectedPeripherals:(NSArray *)peripherals {
    CBPeripheral *peripheral;
    /* Add to list. */
    for (peripheral in peripherals) {
         [central connectPeripheral:peripheral options:nil];
    }
}

 - (void) centralManager:(CBCentralManager *)central didRetrievePeripheral:(CBPeripheral *)peripheral {
    [central connectPeripheral:peripheral options:nil];
}
```

### 后记

* 最主要是用UUID来确定你要干的事情，特征和服务的UUID都是外设定义好的。我们只需要读取，确定你要读取什么的时候，就去判断UUID是否相符。 一般来说我们使用的iPhone都是做centralManager的，蓝牙模块是peripheral的，所以我们是want datas，需要接受数据。 
1.判断状态为powerOn，然后执行扫描 
2.停止扫描，连接外设 
3.连接成功，寻找服务 
4.在服务里寻找特征 
5.为特征添加通知 
5.通知添加成功，那么就可以实时的读取value[也就是说只要外设发送数据[一般外设的频率为10Hz]，代理就会调用此方法]。 
6.处理接收到的value，[hex值，得转换] 之后就自由发挥了，在这期间都是通过代理来实现的，也就是说你只需要处理你想要做的事情，代理会帮你调用方法。[别忘了添加代理] 

### 2015-07-28 更

##### 关于write我这里还有些注意的地方要强调！！！！

并不是每一个Characteristic都可以通过回调函数来查看它写入状态的。就比如针对 immediateAlertService（1802） 的 alertLevelCharacteristic（2A06），就是一个不能有response的Characteristic。刚开始我就一直用CBCharacteristicWriteType.WithResponse来进行写入始终不成功，郁闷坏了，最后看到每个Characteristic还有个属性值是指示这个的，我将每个Characteristic打印出来有如下信息：

```
immediateAlertService Discover characteristic <CBCharacteristic: 0x15574d00, UUID = 2A06, properties = 0x4, value = (null), notifying = NO>
linkLossAlertService Discover characteristic <CBCharacteristic: 0x15671d00, UUID = 2A06, properties = 0xA, value = (null), notifying = NO>
```

这个的properties是什么刚开始不知道，觉得他没意义，后面才注意到properties是Characteristic的一个参数，具体解释如下：


```
Declaration
SWIFT
struct CBCharacteristicProperties : RawOptionSetType {
init(_ value: UInt)
var value: UInt
static var Broadcast: CBCharacteristicProperties { get }
static var Read: CBCharacteristicProperties { get }
static var WriteWithoutResponse: CBCharacteristicProperties { get }
static var Write: CBCharacteristicProperties { get }
static var Notify: CBCharacteristicProperties { get }
static var Indicate: CBCharacteristicProperties { get }
static var AuthenticatedSignedWrites: CBCharacteristicProperties { get }
static var ExtendedProperties: CBCharacteristicProperties { get }
static var NotifyEncryptionRequired: CBCharacteristicProperties { get }
static var IndicateEncryptionRequired: CBCharacteristicProperties { get }
}
OBJECTIVE-C
typedef enum {
CBCharacteristicPropertyBroadcast = 0x01,
CBCharacteristicPropertyRead = 0x02,
CBCharacteristicPropertyWriteWithoutResponse = 0x04,
CBCharacteristicPropertyWrite = 0x08,
CBCharacteristicPropertyNotify = 0x10,
CBCharacteristicPropertyIndicate = 0x20,
CBCharacteristicPropertyAuthenticatedSignedWrites = 0x40,
CBCharacteristicPropertyExtendedProperties = 0x80,
CBCharacteristicPropertyNotifyEncryptionRequired = 0x100,
CBCharacteristicPropertyIndicateEncryptionRequired = 0x200,
} CBCharacteristicProperties;
```

可以看到`0x04`对应的是`CBCharacteristicPropertyWriteWithoutResponse`
`0x0A`对应的是`CBCharacteristicPropertyNotify`

所以 immediateAlertService（1802） 的 alertLevelCharacteristic（2A06）是不能用CBCharacteristicWriteType.WithRespons进行写入，只能用CBCharacteristicWriteType.WithOutRespons。这样在以后的开发中可以对每个Characteristic的这个参数进行检查再进行设置。



最后讲一下关于蓝牙绑定的过程，在iOS中，没有讲当绑定的过程，直接就是扫描、连接、交互。从而很多人会认为，连接就是绑定了，其实不然。在iOS开发中，连接并没有完成绑定，在网上找到了个很好的解释：

```
you cannot initiate pairing from the iOS central side. Instead, you have to read/write a characteristic value,
and then let your peripheral respond with an "Insufficient Authentication" error.
iOS will then initiate pairing, will store the keys for later use (bonding) and encrypts the link. As far as I know,
it also caches discovery information, so that future connections can be set up faster.
```

就是当发生读写交互时，系统在会和外设进行绑定操作!!!

### 2016-02-20 更 

[ios蓝牙如何获取广播包数据](http://www.cocoachina.com/bbs/read.php?tid-336169.html)   

如题，手机作为主设备，在使用CoreBluetooth时候，想获取蓝牙的数据广播包。在使用- (void)centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)aPeripheral advertisementData:(NSDictionary *)advertisementData RSSI:(NSNumber *)RSSI方法时候获取的advertisementData打印出来只有** ****kCBAdvDataIsConnectable****kCBAdvDataLocalName****kCBAdvDataManufacturerData**三个属性对应的值，但这并非广播包的数据。例如安卓可以通过scandata来获取到广播包的值，那么iOS这边我应该怎么做呢？

好像苹果这边禁止读取这种广播内容的的，真要的话你可以让硬件那边把数据做到kCBAdvDataManufacturerData这个字段里面。

Demo地址：[一个蓝牙4.0的智能硬件Demo](https://github.com/yuhanle/WEBlueToothManager)


本站文章如无其他特殊说明，均为原创，转载请注明出处。
