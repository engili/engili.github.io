---
layout: post
title: "iOS Tips"
date: 2017-05-14 06:58:39 +0800
comments: true
categories: iOS
---
<!-- more -->

### 1.在block执行期间，weak引用强持有

```obj-c
__weak typedef(someClass) weakClass = someClass;
[xxx setHandler:^{
    typedef(someClass) strongClass = weakClass;
    [strongClass doSomething];
}];
```

- 为了避免强引用环，在使用闭包时，一些引入的对象，都使用不增加引用计数的`weak`指针持有，但当使用的时候，为了避免对象被提前释放，可以用`strong`指针持有，可以确保在使用该对象的时候，不会被释放。 
- 使用不带修饰的指针变量，默认是带有`__strong`修饰

### 2. UISwitch 重复触发action方法
理论上，只用当用户点击了`UISwitch`,才会触发，`vauleChange` 的action方法 但是实际开发中，发现iOS10机型上，如果在action方法里调用了-`setOn:animated:`或者`setOn:` ，就会多触发一次action方法。

#### 解决方案
1. 避免在valueChange方法里调用-setOn:animated:或setOn:
2. 如果无法避免，使用dispatch_async,在主队列执行这些方法（这个方法有个缺点，比较卡的手机会看到闪动的现象）

```obj-c
- (IBAction)valueChanged:(id)sender {
    ...
    dispatch_async(dispatch_get_main_queue(), ^{
       [sender setOn:YES];
    });
    ...
}
```
3.iOS 10 以下机型不会出现这个问题，iOS11 待验证

#### 参考：
[UISwitch setOn(:, animated:) does not work as document](https://stackoverflow.com/questions/39566361/uiswitch-seton-animated-does-not-work-as-document)

### 3. Xcode8 生成CoreData NSManagedObject 报Duplicate symbol error

上周写CoreData的时候，当使用Editor的Create NSManagedObject SubClass 生成好对应Entity的ManagedObject后，编译的时候报Duplicate symbol error，意思是符号重复定义的错误，一脸蒙蔽好吧，都是系统生成的，我什么都没有做=。=

然后网上搜了一下，搜到了问题的原因和解决方案 [duplicate-symbol-error-when-adding-nsmanagedobject-subclass-duplicate-link xcode-beta-8-cant-create-core-data](http://stackoverflow.com/questions/40460307/duplicate-symbol-error-when-adding-nsmanagedobject-subclass-duplicate-link)

#### 原因
Xcode 8 的.xcdatamodeId 文件会自动生成那些Entity的类，当你在Editor->Create NSManagedObject SubClass 再次生成这些Entity类的时候，编译的时候，就会有符号重定义的错误

#### 解决方案
- 当你不需要自己修改NSManagedObject 文件时候，就使用Xcode生成的文件，直接引用头文件就好。
- 当你需要自己修改NSManagedObject 文件，在Xcode中选中.xcdatamodeId 文件 在右侧属性栏中， ~~把Tools Version 修改成小于8.0的版本 ，然后就可以使用Editor 生成NSManagedObject类了。~~ 修改Entity Inspector 里Class对应的Codegen 如下图：
![](https://github.com/engili/engili.github.io/raw/master/images/ios-tips-01.png)

### 4. Swift 单例

#### 使用类常量

```swift
class MTNetWorkManager {

static let shared = MTNetWorkManager(baseURL: API.baseURL)

let baseURL: URL

private init(baseURL: URL) {
self.baseURL = baseURL
}
}
//外部调用 MTNetWorkManager.shared
```

#### 使用类方法

```swift
class MTNetWorkManager {

static private let sharedInstance = {

let shared = MTNetWorkManager(baseURL: API.baseURL)

//Configure 可以做一些初始化配置

return shared

}()

let baseURL: URL

private init(baseURL: URL) {
self.baseURL = baseURL
}


static func shared() {
return sharedInstance
}
}

//外部调用 MTNetWorkManager.shared()
```