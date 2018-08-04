---
layout: post
title: "CoreData 学习笔记"
date: 2017-08-20 18:41:18 +0800
comments: true
categories: iOS
---
<!-- more -->

### CoreData 学习笔记

#### 1.Core Data Stack

##### iOS 10 以前创建CoreDataStack

###### 1 Data Model
Data Model 是Xcode提供的一个可视化Model 编辑器，可以创建实例，定义实例属性，定义实例之间的关系，以及创建一些常用的FetchRequst
####### 1.1 新建DataModel 文件

![](https://github.com/engili/engili.github.io/raw/master/images/coredata-notes-01.png)

####### 1.2 添加实例&定义属性

![](https://github.com/engili/engili.github.io/raw/master/images/coredata-notes-02.png)

######## 1.21 属性类型（Attribute Type）

- Integer16 Integer32 Integer64
- Decimal Float Double
- String
- Boolean
- Date
- Binary Data
- Transformable

其中，`Binary Data` 为二进制数据，在Xcode右侧Model Inspector 中，有一个可以设置Allows External Storage的选项，CoreData会智能选择是存储文件的二进制数据，还是存储文件URL

`Transformable` 可以为任意遵循NSCoding协议的对象类型

####### 1.3 添加关系（relationShip）

- to one （一对一）
- to Manay （一对多）

这里再定义一个实例，主人（master），设定主人和狗子的关系为一对多，关系图如下：

![](https://github.com/engili/engili.github.io/raw/master/images/coredata-notes-03.png)

###### 2 CoreDataStack

CoreDataStck，是自定义的一个CoreData 的栈对象，为一个单例，可以通过它，初始化项目的CoreData，以及获取到Context，对数据库进行增删改查等操作

####### 2.1 单例

```obj-c
//  CoreDataStack.h

#import <Foundation/Foundation.h>
#import <CoreData/CoreData.h>

@interface CoreDataStack : NSObject

+ (instancetype)sharedInstance;

@end 
```

```obj-c
//  CoreDataStack.m

#import "CoreDataStack.h"

@implementation CoreDataStack

+ (instancetype)sharedInstance {
    static CoreDataStack *stack;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        stack = [[CoreDataStack alloc] init];
    });

    return stack;
}
@end
```

###### 3 NSManagedModel

The NSManagedObjectModel represents each object type in your app’s data model, the properties they can have, and the relationship between them.

```obj-c
//  CoreDataStack.m

@interface CoreDataStack ()

@property (nonatomic, strong) NSManagedObjectModel *managedModel;

@end

......
- (NSManagedObjectModel *)managedModel {
    if (!_managedModel) {
        NSURL *momdURL = [[NSBundle mainBundle]URLForResource:@"Model" withExtension:@"momd"];
        _managedModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:momdURL];
    }
    return _managedModel;
}
```

###### 4 NSPersistentStoreCoordinator

####### 4.1 NSpersistentStore

######## atomic vs nonatomic
An atomic persistent store needs to be completely deserialized and loaded into memory before you can make any read or write operations. In contrast, a non- atomic persistent store can load chunks of itself onto memory as needed.

######## type
- NSSQLiteStoreType (nonatomic)
- NSXMLStoreType (atomic)
- NSBinaryStoreType (atomic)
- NSInMemoryStoreType (atomic)

####### 4.2 NSPersistentStoreCoordinator
the bridge between the managed object model and the persistent store

- 通过managed object model初始化
- 通过addPersistentStoreWithType:configuration:URL:options:error: 添加persistent store

```obj-c
//  CoreDataStack.m

@interface CoreDataStack ()

@property (nonatomic, strong) NSPersistentStoreCoordinator *psc;

@end

......

- (NSPersistentStoreCoordinator *)psc {
    if (!_psc) {
        _psc = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:self.managedModel];

        NSURL *URL = [[self documentURL] URLByAppendingPathComponent:@"Model.sqlite"];
        NSError *error;
        if (![_psc addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:URL options:nil error:&error]) {
            NSLog(@"addPersistentStoreWithType Error: %@",[error localizedDescription]);
        }
    }
    return _psc;
}

- (NSURL *)documentURL {
    return [[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask] firstObject];
}
```

###### 5 NSManagedObjectContext

Context是唯一暴露给外界，供外界使用的接口

- context an in-memory scratchpad for managed objects.
- any change you make won’t affect the underlying data on disk until you call -save: of context
- context manages the lifecycle of the objects that it creates or fetches.
- A managed object cannot exist without an associated context.
- once a managed object has associated with a particular context, it will remain associated with the same context for the duration of its lifecycle.
- context and managed object is not thread safe

```obj-c
//  CoreDataStack.h

@interface CoreDataStack : NSObject

@property (nonatomic, strong, readonly) NSManagedObjectContext *managedContext;

@end
```

```obj-c
//  CoreDataStack.m

@interface CoreDataStack ()

@property (nonatomic, strong, readwrite) NSManagedObjectContext *managedContext;

@end

- (NSManagedObjectContext *)managedContext {
    if (!_managedContext) {
        _managedContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
        _managedContext.persistentStoreCoordinator = self.psc;
    }
    return _managedContext;
}
```

###### 6 Save Context

```obj-c
- (void)saveContext {
    NSError *error;
    if ([self.managedContext hasChanges] && ![self.managedContext save:&error]) {
        NSLog(@"NSManagedObjectContext Save Error: %@",[error localizedDescription]);
    }
}
```

##### iOS 10 以后创建CoreDataStack

iOS 10 以后，苹果系统为我们封装了一个CoreData Stack 的类，叫NSPersistentContainer,可以通过它的属性viewContext获取到NSManagedObjectContext

```obj-c
@property (readonly, strong) NSPersistentContainer *persistentContainer;

- (NSPersistentContainer *)persistentContainer {
    // The persistent container for the application. This implementation creates and returns a container, having loaded the store for the application to it.
    @synchronized (self) {
        if (_persistentContainer == nil) {
            _persistentContainer = [[NSPersistentContainer alloc] initWithName:@"testCoreData"];
            [_persistentContainer loadPersistentStoresWithCompletionHandler:^(NSPersistentStoreDescription *storeDescription, NSError *error) {
                if (error != nil) {
                    // Replace this implementation with code to handle the error appropriately.
                    // abort() causes the application to generate a crash log and terminate. You should not use this function in a shipping application, although it may be useful during development.

                    /*
                     Typical reasons for an error here include:
                     * The parent directory does not exist, cannot be created, or disallows writing.
                     * The persistent store is not accessible, due to permissions or data protection when the device is locked.
                     * The device is out of space.
                     * The store could not be migrated to the current model version.
                     Check the error message to determine what the actual problem was.
                    */
                    NSLog(@"Unresolved error %@, %@", error, error.userInfo);
                    abort();
                }
            }];
        }
    }

    return _persistentContainer;
}

- (void)saveContext {
    NSManagedObjectContext *context = self.persistentContainer.viewContext;
    NSError *error = nil;
    if ([context hasChanges] && ![context save:&error]) {
        // Replace this implementation with code to handle the error appropriately.
        // abort() causes the application to generate a crash log and terminate. You should not use this function in a shipping application, although it may be useful during development.
        NSLog(@"Unresolved error %@, %@", error, error.userInfo);
        abort();
    }
}

```

#### 2.创建NSManagedObject对象

可视化Data Model文件在程序中，对应了NSManagedModel对象，这些Entity在程序中，所对应的对象就是NSManagedObject (当然其实并不完全对等，可以先这么理解)

##### Document

> NSManagedObject is a generic class that implements all the basic behavior required of a Core Data model object. It is not possible to use instances of direct subclasses of NSObject (or any other class not inheriting from NSManagedObject) with a managed object context. You may create custom subclasses of NSManagedObject, although this is not always required. If no custom logic is needed, a complete Object graph can be formed with NSManagedObject instances.
 

这里大概有三个要点

- NSManagedObject 代表了Data Model中的对象
- NSManagedObject 需要和 NSManagedObjectContext 结合使用才有意义
- NSManagedObject 满足所有你自定义的model 对象需求，通过KVC 去访问相应的属性

##### 生成NSManagedObject子类

虽然通过KVC就可以访问NSManagedObject所有的属性，但这样使用起来不是很方便。

在Xcode Data Model Editor 中选中你要创建NSManagedObject子类的实例，在右侧Data Model Inspector中选择Codegen

![](https://github.com/engili/engili.github.io/raw/master/images/coredata-notes-04.png)

- Manual/None
    - Manual/None is the default, and previous behavior; in this case you should implement your own subclass or use NSManagedObject.

这个是iOS10以前的默认行为，需要我们手动通过Xcode Editor-> Create NSManagedObject SubClass... 生成ClassName+CoreDataClass 文件以及 ClassName+CoreDataGeneratedProperties 其中前者，是类的定义，以及类行为定义，后者通过category 定义了类的属性（注：Objective-C中Category定义属性不支持生成成员变量，但可以生成get set 方法，CoreData属性通过@dynamic修饰，表示，运行时生成 get set 方法）

当每次修改，data model的时候，重新通过以上方法，生成NSManagedObject对象，系统只会覆盖ClassName+CoreDataGeneratedProperties文件，而不会修改ClassName+CoreDataClass 文件。
- Category/Extension.
    - Category/Extension generates a class extension in a file named like ClassName+CoreDataGeneratedProperties. You need to declare/implement the main class (if in Obj-C, via a header the extension can import named ClassName.h).

一般情况下，因为我们不需要修改属性Category的定义，而只需要修改类行为的定义，所以这个选项，就直接不对开发者暴露Category 文件，当你修改了DataModel的时候，就可以自动同步最新的代码
- Class Definition

    - Class Definition generates subclass files named like ClassName+CoreDataClass as well as the files generated for Category/Extension.

当我们也不需要为NSManagedObject定义行为的时候，我们就可以选中这个选项，然后直接在项目里引用头文件就可以直接使用。

##### 数据库 增删改查

###### 增
增加一个数据库对象，首先，需要创建一个NSEntityDescription,前面说 Data Model Editor 里的实例并不完全和NSManagedObject对等，就是因为创建NSManagedObject对象，还需要entity相关的描述对象，这个对象才与实例对等。

```obj-c
NSEntityDescription *entity = [NSEntityDescription entityForName:@"Doge" inManagedObjectContext:_context];
Doge *doge = [[Doge alloc] initWithEntity:entity insertIntoManagedObjectContext:_context];
doge.name = @"xxx";
...
...
//注意，这里并没有实际把对象存入数据库，实际存入需要调用NSManagedObjectContext 的save:方法
[_context save:nil];
```

###### 改

直接访问NSManagedObject属性的set方法，就可以修改属性，同时，也只有当调用了NSManagedObjectContext 的-save:方法后才能存入实际的存储文件中

###### 删

删除调用NSManagedObjectContext的-deleteObject:方法，然后调用-save:方法

```obj-c
Doge *doge = ...
 [_context deleteObject:doge];
 [_context save:nil]
```

###### 查

####### 1. NSFetchRequest
######## resultType
- NSManagedObjectResultType
- NSCountResultType
- NSDictionaryResultType
- NSManagedObjectIDResultType

其中默认属性为`NSManagedObjectResultType`

######## NSManagedObjectResultType

NSManagedObjectResultType是NSFetchRequest 的默认属性，执行查询后，数组里每个元素，就是NSFetchRequest 对应的NSManagedObject对象。

```obj-c

NSFetchRequest *fetchRequest =  [NSFetchRequest fetchRequestWithEntityName:@"Doge"];
//默认就为NSManagedObjectResultType
//fetchRequest.resultType = NSManagedObjectResultType;
NSArray *result = [_context executeFetchRequest:fetchRequest error:nil];
//这里数组的元素为Doge对象

```

######## NSCountResultType

NSCountResultType执行查询后，返回的数组，包含一个对象，NSNmber.

```obj-c
NSFetchRequest *fetchRequest =  [NSFetchRequest fetchRequestWithEntityName:@"Doge"];
fetchRequest.resultType = NSCountResultType;
NSArray *result = [_context executeFetchRequest:fetchRequest error:nil];
//这里数组的元素为 NSNumber ，一般数组只有一个元素
if ([result count] > 0){
    NSInteger count = [[results objectAtIndex:0] integerValue];
}
```

######## NSDictionaryResultType  
NSDictionaryResultType 顾名思义，是返回一个字典，除了设置fetchRequest.resultType,还需要设置fetchRequest.propertiesToFetch

######## NSManagedObjectIDResultType

返回唯一标识

Prior to iOS 5, fetching by ID was popular because NSManagedObjectID is thread-safe and using it helped developers implement the thread confinement concurrency model. Now that thread confinement has been deprecated in favor of more modern concurrency models, there’s little reason to fetch by object ID anymore.

####### 2. NSPredicate

The NSPredicate class is used to define logical conditions used to constrain a search either for a fetch or for in-memory filtering.

NSFetchRequest 支持使用谓词来作为，查询筛选条件

```obj-c
NSFetchRequest *fetchRequest =  [NSFetchRequest fetchRequestWithEntityName:@"Doge"];
fetchRequestWithEntityName:@"Doge"];
fetchRequest.predicate = [NSPredicate predicateWithFormat:@"age > 5"];
NSArray *result = [_context executeFetchRequest:fetchRequest error:nil];
```

####### 3. NSSortDescriptor

除了使用支持使用谓词，NSFetchRequest 还支持使用NSSortDescriptor 进行排序

```obj-c
NSFetchRequest *fetchRequest =  [NSFetchRequest fetchRequestWithEntityName:@"Doge"];
    NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"age" ascending:YES];
    fetchRequest.sortDescriptors = @[sortDescriptor];

    NSArray *result = [_context executeFetchRequest:fetchRequest error:nil];
```

####### 4. NSAsynchronousFetchRequest

NSFetchRequest 是同步查询的，CoreData 也提供了异步查询的类NSAsynchronousFetchRequest,它通过一个 block ，将数据异步返回，注意，查询方法使用的是 -executeRequest:error:

```obj-c
NSFetchRequest *fetchRequest =  [NSFetchRequest fetchRequestWithEntityName:@"Doge"];
NSAsynchronousFetchRequest *asyncFetchRequest = [[NSAsynchronousFetchRequest alloc] initWithFetchRequest:fetchRequest completionBlock:^(NSAsynchronousFetchResult * _Nonnull results) {
    NSLog(@"%@",results);
}];
[_context executeRequest:asyncFetchRequest error:nil];
```

#### Demo

##### 添加数据

```obj-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    [self importSeedJsonSeedIfNeeded];
    return YES;
}
- (void)importSeedJsonSeedIfNeeded {

    NSManagedObjectContext *context = [[MTCoreDataStack sharedInstance] managedContext];

    NSFetchRequest * fetchRequest = [Master fetchRequest];
    fetchRequest.resultType = NSCountResultType;

    NSError *error;
    NSArray *results = [context executeFetchRequest:fetchRequest error:&error];
    if (!error) {
        if ([results count] > 0) {
            NSInteger masterCount = [[results objectAtIndex:0] integerValue];
            if (masterCount <= 0) {
                [self importJsonSeed];
            }
        } else {
            [self importJsonSeed];
        }

    }else {
        [self importJsonSeed];
    }

}

- (void)importJsonSeed {
    NSURL *jsonURL = [[NSBundle mainBundle] URLForResource:@"seed" withExtension:@"json"];

    NSError *error;
    NSData *jsonData = [NSData dataWithContentsOfURL:jsonURL options:0 error:&error];

    if (!error) {
        NSDictionary *jsonDict = [NSJSONSerialization JSONObjectWithData:jsonData options:0 error:&error];
        if (!error) {

            NSManagedObjectContext *context = [[MTCoreDataStack sharedInstance] managedContext];

            NSEntityDescription *masterEntity = [NSEntityDescription entityForName:@"Master" inManagedObjectContext:context];
            NSEntityDescription *dogeEntity = [NSEntityDescription entityForName:@"Doge" inManagedObjectContext:context];

            NSDictionary *masterDict = jsonDict[@"master"];
            if (![masterDict isKindOfClass:[NSDictionary class]]) {
                return;
            }

            Master *master = [[Master alloc] initWithEntity:masterEntity insertIntoManagedObjectContext:context];
            master.name = masterDict[@"name"];
            master.age = [masterDict[@"age"] integerValue];

            NSArray *doges = jsonDict[@"doges"];

            for (NSDictionary *dogeDict in doges) {
                if (![dogeDict isKindOfClass:[NSDictionary class]]) {
                    continue;
                }
                Doge *doge = [[Doge alloc] initWithEntity:dogeEntity insertIntoManagedObjectContext:context];
                doge.name = dogeDict[@"name"];
                doge.age = [dogeDict[@"age"] integerValue];
                doge.master = master; //因为在DataModel 里设置了 inverse， 所以master 和 doge可以相互关联上
            }
            //一定要调用 save 方法，才能真正的将文件存入数据库
            [[MTCoreDataStack sharedInstance] saveContext];
        }
    }
}
```