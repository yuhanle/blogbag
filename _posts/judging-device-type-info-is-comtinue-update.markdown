---
layout: post
title:  "iOS-判断设备类型信息 持续更新..."
date:   2015-03-20 09:00:00
categories: iOS学习
tags: [iOS学习]
---

由于项目需求，需要像服务器上传客户端的机型信息，方便反馈查询错误。但是uuid并不唯一，但也是目前用的最多的唯一标识符，我们可以将其存入钥匙串中，以保证用户在卸载或升级系统时仍保证其唯一性。<!--more-->

### 1.UUID

uuid并不唯一，但也是目前用的最多的唯一标识符，我们可以将其存入钥匙串中，以保证用户在卸载或升级系统时仍保证其唯一性。

> "CHKeychain.h"  可以在keychain中存取数据，使用类似于NSUserDefault


```
#pragma mark--获取设备UUID后存入keyChain中
NSString * GetUUID () {
    if ([CHKeychain load:UUIDKEY]) {
        NSString *result = [CHKeychain load:UUIDKEY];
        return result;
    }else {
        CFUUIDRef puuid = CFUUIDCreate( nil );
        CFStringRef uuidString = CFUUIDCreateString( nil, puuid );
        NSString * result = (NSString *)CFBridgingRelease(CFStringCreateCopy(NULL, uuidString));
        CFRelease(puuid);
        CFRelease(uuidString);
        
        NSMutableString * tmpResult = result.mutableCopy;
        NSRange range = [tmpResult rangeOfString:@"-"];
        
        while (range.location != NSNotFound) {
            [tmpResult deleteCharactersInRange:range];
            
            range = [tmpResult rangeOfString:@"-"];
        }
        [CHKeychain save:UUIDKEY data:tmpResult.copy];
        
        return tmpResult.copy;
    }
    
    return nil;
}
```

### 2.设备型号

```
+ (NSString *)platform {
    struct utsname systemInfo;
    uname(&systemInfo);
    
    NSString *platform = [NSString stringWithCString:systemInfo.machine encoding:NSASCIIStringEncoding];
    
    if ([platform isEqualToString:@"iPhone1,1"]) return @"iPhone 2G";
    if ([platform isEqualToString:@"iPhone1,2"]) return @"iPhone 3G";
    if ([platform isEqualToString:@"iPhone2,1"]) return @"iPhone 3GS";
    if ([platform isEqualToString:@"iPhone3,1"]) return @"iPhone 4";
    if ([platform isEqualToString:@"iPhone3,2"]) return @"iPhone 4";
    if ([platform isEqualToString:@"iPhone3,3"]) return @"iPhone 4";
    if ([platform isEqualToString:@"iPhone4,1"]) return @"iPhone 4S";
    if ([platform isEqualToString:@"iPhone5,1"]) return @"iPhone 5";
    if ([platform isEqualToString:@"iPhone5,2"]) return @"iPhone 5";
    if ([platform isEqualToString:@"iPhone5,3"]) return @"iPhone 5c";
    if ([platform isEqualToString:@"iPhone5,4"]) return @"iPhone 5c";
    if ([platform isEqualToString:@"iPhone6,1"]) return @"iPhone 5s";
    if ([platform isEqualToString:@"iPhone6,2"]) return @"iPhone 5s";
    if ([platform isEqualToString:@"iPhone7,1"]) return @"iPhone 6 Plus";
    if ([platform isEqualToString:@"iPhone7,2"]) return @"iPhone 6";
    if ([platform isEqualToString:@"iPhone8,1"]) return @"iPhone 6s";
    if ([platform isEqualToString:@"iPhone8,2"]) return @"iPhone 6s Plus";
    
    if ([platform isEqualToString:@"iPod1,1"])   return @"iPod Touch 1G";
    if ([platform isEqualToString:@"iPod2,1"])   return @"iPod Touch 2G";
    if ([platform isEqualToString:@"iPod3,1"])   return @"iPod Touch 3G";
    if ([platform isEqualToString:@"iPod4,1"])   return @"iPod Touch 4G";
    if ([platform isEqualToString:@"iPod5,1"])   return @"iPod Touch 5G";
    
    if ([platform isEqualToString:@"iPad1,1"])   return @"iPad 1G";
    
    if ([platform isEqualToString:@"iPad2,1"])   return @"iPad 2";
    if ([platform isEqualToString:@"iPad2,2"])   return @"iPad 2";
    if ([platform isEqualToString:@"iPad2,3"])   return @"iPad 2";
    if ([platform isEqualToString:@"iPad2,4"])   return @"iPad 2";
    if ([platform isEqualToString:@"iPad2,5"])   return @"iPad Mini 1G";
    if ([platform isEqualToString:@"iPad2,6"])   return @"iPad Mini 1G";
    if ([platform isEqualToString:@"iPad2,7"])   return @"iPad Mini 1G";
    
    if ([platform isEqualToString:@"iPad3,1"])   return @"iPad 3";
    if ([platform isEqualToString:@"iPad3,2"])   return @"iPad 3";
    if ([platform isEqualToString:@"iPad3,3"])   return @"iPad 3";
    if ([platform isEqualToString:@"iPad3,4"])   return @"iPad 4";
    if ([platform isEqualToString:@"iPad3,5"])   return @"iPad 4";
    if ([platform isEqualToString:@"iPad3,6"])   return @"iPad 4";
    
    if ([platform isEqualToString:@"iPad4,1"])   return @"iPad Air";
    if ([platform isEqualToString:@"iPad4,2"])   return @"iPad Air";
    if ([platform isEqualToString:@"iPad4,3"])   return @"iPad Air";
    if ([platform isEqualToString:@"iPad4,4"])   return @"iPad Mini 2G";
    if ([platform isEqualToString:@"iPad4,5"])   return @"iPad Mini 2G";
    if ([platform isEqualToString:@"iPad4,6"])   return @"iPad Mini 2G";
    
    if ([platform isEqualToString:@"i386"])      return @"iPhone Simulator";
    if ([platform isEqualToString:@"x86_64"])    return @"iPhone Simulator";
    return platform;
}
```

### 3.设备系统版本号

```
[[UIDevice currentDevice] systemVersion]]
```

![QQ20151019-0.png](http://7xqhcq.com1.z0.glb.clouddn.com/she-bei-lei-xing-chixugeng.png)

### 4.运营商信息

关于获取运营商信息，需通过CoreTelephony Framework中的CTTelephonyNetworkInfo和CTCarrier类型。这些都在iOS 4.0后就有了。
import必要的header：

```
#import <CoreTelephony/CTCarrier.h>
#import <CoreTelephony/CTTelephonyNetworkInfo.h>
```

CTCarrier类型代表着具体的运营商信息。调用CTTelephonyNetworkInfo的subscriberCellularProvider方法来获取当前运营商信息，或者调用subscriberCellularProviderDidUpdateNotifier方法来觉察运营商变化。
获取了CTCarrier类型，就可以执行从他的属性中获取运营商信息了。
目前他有如下属性：allowsVOIP，carrierName，isoCountryCode，mobileCountryCode ，mobileNetworkCode。参考[官方文档](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Reference/CTCarrier/Reference/Reference.html#//apple_ref/doc/c_ref/CTCarrier)。
 
其中isoCountryCode使用ISO 3166-1标准，参考：[3166](http://en.wikipedia.org/wiki/ISO_3166-1)
mobileCountryCode(MCC)和mobileNetworkCode(MNC)可以参考：[3166-1](http://en.wikipedia.org/wiki/Mobile_country_code)
中国的MCC是460。中国的MNC也在列表中，如下表：

MNC | 运营商 
:----:|:------:
00 | 中国移动 
01| 中国联通
02 | 中国移动
03 | 中国电信
05| 中国电信
06 | 中国联通
07 | 中国移动
20| 中国铁通

> 获取运营商实例代码

```
-(void)getcarrierName{
CTTelephonyNetworkInfo *telephonyInfo = [[CTTelephonyNetworkInfo alloc] init];
CTCarrier *carrier = [telephonyInfo subscriberCellularProvider];
NSString *currentCountry=[carrier carrierName];
NSLog(@"[carrier isoCountryCode]==%@,[carrier allowsVOIP]=%d,[carrier mobileCountryCode=%@,[carrier mobileCountryCode]=%@",[carrier isoCountryCode],[carrier allowsVOIP],[carrier mobileCountryCode],[carrier mobileNetworkCode]);
}
```

附：

>  CHKeychain.h文件


```
//
//  CHKeychain.h
//  G100
//
//  Created by Tilink on 15/4/3.
//  Copyright (c) 2015年 Tilink. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface CHKeychain : NSObject

+ (void)save:(NSString *)service data:(id)data;
+ (id)load:(NSString *)service;
+ (void)deleteData:(NSString *)service;

@end
```

 > CHKeychain.m文件

```
//
//  CHKeychain.m
//  G100
//
//  Created by Tilink on 15/4/3.
//  Copyright (c) 2015年 Tilink. All rights reserved.
//

#import "CHKeychain.h"

@implementation CHKeychain

+ (NSMutableDictionary *)getKeychainQuery:(NSString *)service {
    return [NSMutableDictionary dictionaryWithObjectsAndKeys:
            (id)kSecClassGenericPassword,(id)kSecClass,
            service, (id)kSecAttrService,
            service, (id)kSecAttrAccount,
            (id)kSecAttrAccessibleAfterFirstUnlock,(id)kSecAttrAccessible,
            nil];
}

+ (void)save:(NSString *)service data:(id)data {
    //Get search dictionary
    NSMutableDictionary *keychainQuery = [self getKeychainQuery:service];
    //Delete old item before add new item
    SecItemDelete((CFDictionaryRef)keychainQuery);
    //Add new object to search dictionary(Attention:the data format)
    [keychainQuery setObject:[NSKeyedArchiver archivedDataWithRootObject:data] forKey:(id)kSecValueData];
    //Add item to keychain with the search dictionary
    SecItemAdd((CFDictionaryRef)keychainQuery, NULL);
}

+ (id)load:(NSString *)service {
    id ret = nil;
    NSMutableDictionary *keychainQuery = [self getKeychainQuery:service];
    //Configure the search setting
    //Since in our simple case we are expecting only a single attribute to be returned (the password) we can set the attribute kSecReturnData to kCFBooleanTrue
    [keychainQuery setObject:(id)kCFBooleanTrue forKey:(id)kSecReturnData];
    [keychainQuery setObject:(id)kSecMatchLimitOne forKey:(id)kSecMatchLimit];
    CFDataRef keyData = NULL;
    if (SecItemCopyMatching((CFDictionaryRef)keychainQuery, (CFTypeRef *)&keyData) == noErr) {
        @try {
            ret = [NSKeyedUnarchiver unarchiveObjectWithData:(NSData *)keyData];
        } @catch (NSException *e) {
            NSLog(@"Unarchive of %@ failed: %@", service, e);
        } @finally {
        }
    }
    if (keyData)
        CFRelease(keyData);
    return ret;
}

+ (void)deleteData:(NSString *)service {
    NSMutableDictionary *keychainQuery = [self getKeychainQuery:service];
    SecItemDelete((CFDictionaryRef)keychainQuery);
}

@end
```

> 有多少人和我一样，坐在不足10平米的空间里，看着书里九万五千公里的绚丽。又或是和我一样，拥有一颗比九万五千公里还辽阔的心，却坐在不足一平米的椅子上。

本站文章如无其他特殊说明，均为原创，转载请注明出处。

