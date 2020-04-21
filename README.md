#### 一、介绍

模型转字典，字典转模型，这是开发中最基本的功能。系统类中提供了一个setValuesForKeysWithDictionary方法来实现字典转模型，至于模型转字典，这个就需要使用runtime来实现了。其实字典和模型的互转可以完全使用运行时runtime来实现。典型的第三方有MJExtension和YYModel。现在我就来借助runtime的思想来进行大致的实现。

 

#### 二、思路

- 字典转模型
  - 遍历key
  - objc_msgSend对key发送消息
- 模型转字典
  - class_copyPropertyList获取属性列表properties
  - 遍历properties，获取属性名name
  - objc_msgSend对name发送消息，设置value
  - 存入dic并返回
 
 
#### 三、示例

`BaseModel.h`
```objc
//
//  BaseModel.h
//  HWMultiMedia
//
//  Created by 宇航 on 2020/3/23.
//  Copyright © 2020 yuhang. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface BaseModel : NSObject
/**
 字典转模型
 @param dict 字典
 @return 模型
 */
-(instancetype)initWithDict:(NSDictionary*)dict;
/**
 模型转字典
 @return 字典
 */
-(NSDictionary*)convertModelToDict;
@end
```

`BaseModel.h`
```objc
//
//  BaseModel.m
//  HWMultiMedia
//
//  Created by 宇航 on 2020/3/23.
//  Copyright © 2020 yuhang. All rights reserved.
//

#import "BaseModel.h"
#import <objc/message.h>

@implementation BaseModel

//实现简单字典与模型转换.YY,MJ的核心代码理解
//使用运行时(objc_msgSend)发信息 调用set方法给对象赋值
//调用objc_msgsend,需要进行类型强转,通过函数指针
//函数指针格式:返回类型(*函数名)(param1,param2)
//objc_msgSend(sel,sel,value)
-(instancetype)initWithDict:(NSDictionary*)dict{
    self = [super init];
    if (self) {
        for (NSString *key in dict.allKeys) {
            //1.创建一个set选择器
            NSString *methodName = [NSString stringWithFormat:@"set%@:",key.capitalizedString];
            SEL selector = NSSelectorFromString(methodName);
            
            //2.set参数
            id value = dict[key];
            if([value isKindOfClass:[NSNull class]]) continue;
            
            //3.发送消息
            if (!selector) continue;
            ((void (*)(id, SEL, id))objc_msgSend)(self, selector, value);
        }
    }
    return self;
}

-(NSDictionary*)convertModelToDict{
    
    NSMutableDictionary*dict = [NSMutableDictionary dictionary];
    unsigned int count = 0;
    objc_property_t *properties = class_copyPropertyList([self class], &count);
    for (int i=0; i<count; i++) {
        objc_property_t property = properties[i];
        //属性名称
        const char * name = property_getName(property);
        NSString *propertyName = [NSString stringWithCString:name encoding:NSUTF8StringEncoding];
      
        //1.创建一个get选择器
        SEL selector = NSSelectorFromString(propertyName);
        //发送消息
        if (!selector) continue;
        id value = ((id (*)(id, SEL))objc_msgSend)(self, selector);
        //空与类型异常处理
        if(!value || [value isKindOfClass:[NSNull class]]) continue;
        //3值存储
        [dict setObject:value forKey:propertyName];
    }
    //copy创建的需要释放
    free(properties);
    return dict;
}
@end
```

#### 四、测试

1、dic -> model
```objc
   NSMutableDictionary*dict = [NSMutableDictionary dictionary];
    [dict setObject:@"奔驰200" forKey:@"name"];
    [dict setObject:@"北京丰台" forKey:@"location"];
    Car*mycar = [[Car alloc] initWithDict:dict];
```
2.model -> dic
```objc
    Car*acar = [[Car alloc] initWithDict:dict];
    acar.name=@"bmw";
    acar.location=@"beijing";
    NSDictionary*param = [acar convertModelToDict];
```
