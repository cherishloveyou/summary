# iOS底层原理探究之-KVC

**允许开发者通过key直接访问对象的属性方法或者成员变量，而不需要调用明确的存取方法。**

实际上，KVC是对NSObject的扩展：NSKeyValueCoding，当然其中对NSArray、NSDictionary、NSMutableDictionary、NSOrderedSet、NSSet也添加了扩展，更方便使用。
 其中主要提供了以下四个方法

```objectivec
- (nullable id)valueForKey:(NSString *)key;
- (void)setValue:(nullable id)value forKey:(NSString *)key;
- (nullable id)valueForKeyPath:(NSString *)keyPath;
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;
```

#### 原理介绍

- set的搜索规则如下

> 1.查找set<Key>:或_set<Key>命名的setter，按照这个顺序，如果找到的话，调用这个方法并将值传进去(根据需要进行对象转换)。
>  2.如果没有发现一个简单的setter，但是accessInstanceVariablesDirectly类属性返回YES，则查找一个命名规则为_<key>、_is<Key>、_<key>、is<Key>的实例变量。根据这个顺序，如果发现则将value赋值给实例变量。
>  3.如果没有发现setter或实例变量，则调用setValue:forUndefinedKey:方法，并默认提出一个异常，但是一个NSObject的子类可以提出合适的行为。

上面提到了一个类属性accessInstanceVariablesDirectly.

它表示是否允许读取实例变量的值，如果为YES则在KVC查找的过程中，从内存中读取属性、实例变量的值。如果不允许外界通过KVC对我们的私有属性和成员变量进行操作，则可以设置此值为NO。
 set的规则相对比较简单，相信大家都能看懂。我们也可以按照这个搜索顺序自己验证是否符合。（实现两个setter方法以及添加四种成员变量，然后依次注释掉后检查是不是按照上面的顺序进行赋值操作。）

- get的搜索规则
   get的搜索规则相对于set就有点复杂了

> 1.通过getter方法搜索实例，例如get<Key>, <key>, is<Key>, _<key>的拼接方案。按照这个顺序，如果发现符合的方法，就调用对应的方法并拿着结果跳转到第五步。否则，就继续到下一步。
>  2.如果没有找到简单的getter方法，则搜索其匹配模式的方法countOf<Key>、objectIn<Key>AtIndex:、<key>AtIndexes:。如果找到其中的第一个和其他两个中的一个，则创建一个集合代理对象NSKeyValueArray，该对象响应所有NSArray的方法并返回该对象。否则，继续到第三步。代理对象随后将NSArray接收到的countOf<Key>、objectIn<Key>AtIndex:、<key>AtIndexes:的消息给符合KVC规则的调用方。当代理对象和KVC调用方通过上面方法一起工作时，就会允许其行为类似于NSArray一样。
>  3.如果没有找到NSArray简单存取方法，或者NSArray存取方法组。则查找有没有countOf<Key>、enumeratorOf<Key>、memberOf<Key>:命名的方法。如果找到三个方法，则创建一个集合代理对象，该对象响应所有NSSet方法并返回。否则，继续执行第四步。此代理对象随后转换countOf<Key>、enumeratorOf<Key>、memberOf<Key>:方法调用到创建它的对象上。实际上，这个代理对象和NSSet一起工作，使得其表象上看起来是NSSet。
>  4.如果没有发现简单getter方法，或集合存取方法组，以及接收类方法accessInstanceVariablesDirectly是返回YES的。搜索一个名为_<key>、_is<Key>、<key>、is<Key>的实例，根据他们的顺序。如果发现对应的实例，则立刻获得实例可用的值并跳转到第五步，否则，跳转到第六步。
>  5.如果取回的是一个对象指针，则直接返回这个结果。如果取回的是一个基础数据类型，但是这个基础数据类型是被NSNumber支持的，则存储为NSNumber并返回。如果取回的是一个不支持NSNumber的基础数据类型，则通过NSValue进行存储并返回。
>  6.如果所有情况都失败，则调用valueForUndefinedKey:方法并抛出异常，这是默认行为。但是子类可以重写此方法。

*其中第二步搜索的意思是：没有找到第一步中的简单getter方法，但是实现了countOf<Key>以及objectIn<Key>AtIndex:、<key>AtIndexes:两个中的其中一个，此时意味着当前对象拥有一个属性名为<key>的NSKeyValueArray类型的属性，它可以响应NSArray的所有方法。到这里其实就可以回答上面提到的第二个问题，一个对象不一定需要显式的写出自己的属性也可以进行存取操作！
 第三步搜索的意思和第二步相似，只是条件更苛刻，且最终返回的是NSSet对象，响应NSSet的所有方法。*

了解了上面的set、get的搜索规则，上面的第一个问题也就回答了，苹果底层会根据你传入的key字符串按照搜索规则进行搜索，并进行存取操作。

- NSMutableArray 、NSMutableSet和NSMutableOrderedSet对应的搜索规则

这几种可变集合的搜索规则基本一致，只是搜索时调用的方法不同。详细的搜索方法都可以在[KVC官方文档](https://link.jianshu.com/?t=https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueCoding/SearchImplementation.html#//apple_ref/doc/uid/20000955-CJBBBFFA)中找到。

上面调用的方法- (NSMutableArray *)mutableArrayValueForKey:(NSString \*)key;其实有一个很重要的使用场景：**KVO监听可变集合的变化**，当然这里的可变集合包括NSMutableArray，NSMutableSet，NSMutableOrderedSet，但不包括NSDictionary。
 通常使用KVO监听某个对象的可变集合属性，当可变集合发生如add、remove等元素操作时，对象并不能收到通知，具体原因可以留到下次分享KVO原理中说明。但是如果使用上面的这种方法获取的可变集合，当内部元素发生变化时也可以收到通知。

## 自己实现

原理：给NSObject添加分类，实现自己的set、get方法，在方法中根据苹果定义的搜索规则进行实现。
 这里就以最简单的set和get方法来说明自己实现的思路，权当抛砖引玉
 为NSObject添加自己的KVC分类NSObject (NewKVC)

```objectivec
NSObject+NewKVC.h

@interface NSObject (NewKVC)
-(void)setMyValue:(id)value forKey:(NSString*)key;
-(id)myValueforKey:(NSString*)key;
@end
```

```objectivec
NSObject+NewKVC.m

@implementation NSObject (NewKVC)
- (void)setMyValue:(id)value forKey:(NSString *)key{
    if (key == nil || key.length == 0) {  //验证key
        return;
    }
    if ([value isKindOfClass:[NSNull class]]) {
        [self setNilValueForKey:key]; //完全自定义需要自定义setMyNilValueForKey
        return;
    }
    if (![value isKindOfClass:[NSObject class]]) {
        @throw @"must be s NSObject type";
        return;
    }
  
    NSString* funcName = [NSString stringWithFormat:@"set%@:",key.capitalizedString];
    if ([self respondsToSelector:NSSelectorFromString(funcName)]) {  //默认优先调用set方法
        [self performSelector:NSSelectorFromString(funcName) withObject:value];
        return;
    }
    unsigned int count;
    BOOL flag = false;
    Ivar* vars = class_copyIvarList([self class], &count);
    for (NSInteger i = 0; i<count; i++) {
        Ivar var = vars[i];
        NSString* keyName = [[NSString stringWithCString:ivar_getName(var) encoding:NSUTF8StringEncoding] substringFromIndex:1];
        
        if ([keyName isEqualToString:[NSString stringWithFormat:@"_%@",key]]) {
            flag = true;
            object_setIvar(self, var, value);
            break;
        }
        
        if ([keyName isEqualToString:key]) {
            flag = true;
            object_setIvar(self, var, value);
            break;
        }
    }
    if (!flag) {
        [self setValue:value forUndefinedKey:key];//如果完全自定义，那么需要写一个setMyValue: forUndefinedKey:方法，这里必要性不是很大，就省略了
    }
}

- (id)myValueforKey:(NSString *)key{
    if (key == nil || key.length == 0) {
        return [NSNull new];
    }
    //这里没有做相关集合的方法查询
    NSString* funcName = [NSString stringWithFormat:@"get%@",key.capitalizedString];
    if ([self respondsToSelector:NSSelectorFromString(funcName)]) {
        return [self performSelector:NSSelectorFromString(funcName)];
    }
    
    unsigned int count;
    BOOL flag = false;
    Ivar* vars = class_copyIvarList([self class], &count);
    for (NSInteger i = 0; i<count; i++) {
        Ivar var = vars[i];
        NSString* keyName = [[NSString stringWithCString:ivar_getName(var) encoding:NSUTF8StringEncoding] substringFromIndex:1];
        if ([keyName isEqualToString:[NSString stringWithFormat:@"_%@",key]]) {
            flag = true;
            return     object_getIvar(self, var);
            break;
        }
        if ([keyName isEqualToString:key]) {
            flag = true;
            return     object_getIvar(self, var);
            break;
        }
    }
    if (!flag) {
        [self valueForUndefinedKey:key];//需要自己实现myValueForUndefinedKey
    }
    return [NSNull new];
}
@end
```

注意.m中需要引入#import <objc/runtime.h>，因为我们需要动态获取当前对象的成员变量，以便存取操作。上面简化版的KVC，只考虑了最简单的情况，如果大家感兴趣，完全可以实现一整套自己的KVC哦。

#### 使用场景

 **1.动态地设值和取值**
 这个应用就不多说了，最基本的应用
 **2.用KVC来访问和修改私有变量**
 对于KVC来说，一个对象没有自己的隐私，只要它愿意，就可以修改任何私有的东西。不信可以试试在.m文件中声明私有属性或者成员变量，KVC一样可以获取到。
 **3.多值操作（model和字典互转）**

```objectivec
- (void)setValuesForKeysWithDictionary:(NSDictionary<NSString *, id> *)keyedValues;
- (NSDictionary<NSString *, id> *)dictionaryWithValuesForKeys:(NSArray<NSString *> *)keys;
```

主要通过这两个API来实现，也很简单，不多介绍。
 **4.修改一些系统控件的内部属性**
 使用runtime来获取Apple不想开放的成员变量，利用KVC进行修改。比如自定义tabbar，textfield等，这个的应用也是比较常见。
 **5.用KVC实现高阶消息传递**
 这个应用场景就比较少了，它的意思是在对容器类使用KVC时，valueForKey:将会被传递给容器中的每一个对象，而不是对容器本身进行操作。比如下面的代码：

```objectivec
NSArray* arrStr = @[@"english",@"franch",@"chinese"];
NSArray* arrCapStr = [arrStr valueForKey:@"capitalizedString"];
for (NSString* str  in arrCapStr) {
    NSLog(@"%@",str);
}
NSArray* arrCapStrLength = [arrStr valueForKeyPath:@"capitalizedString.length"];
for (NSNumber* length  in arrCapStrLength) {
    NSLog(@"%ld",(long)length.integerValue);
}
```

打印结果如下：

![img](https:////upload-images.jianshu.io/upload_images/2719461-a7f21f0a0fe55987?imageMogr2/auto-orient/strip|imageView2/2/w/836/format/webp)

在这里插入图片描述


**6.用KVC中的函数来操作集合（集合主要指NSArray和NSSet，不包括NSDictionary）**

![img](https:////upload-images.jianshu.io/upload_images/2719461-ebcfee9db0609975.png?imageMogr2/auto-orient/strip|imageView2/2/w/503/format/webp)

集合运算符格式


 上面的图是集合运算符的格式，主要是对象调用valueForKeyPath:方法进行操作。运算符有三种：
 1）简单集合运算符共有@avg， @count ， @max ， @min ，@sum5种

```objectivec
NSArray* arrBooks = @[book1,book2,book3,book4];
NSNumber* sum = [arrBooks valueForKeyPath:@"@sum.price"];
```

如上，arrBooks种存放的是4个Book对象，[arrBooks valueForKeyPath:@"@sum.price"]的意思就是计算arrBooks中的每个Book对象的price的和。当然还会有：

```objectivec
NSNumber* avg = [arrBooks valueForKeyPath:@"@avg.price"];
NSNumber* count = [arrBooks valueForKeyPath:@"@count"];
NSNumber* min = [arrBooks valueForKeyPath:@"@min.price"];
NSNumber* max = [arrBooks valueForKeyPath:@"@max.price"];
```

2）对象运算符

```dart
@distinctUnionOfObjects
@unionOfObjects
```

它们的返回值都是NSArray，区别是前者返回的元素都是唯一的，是去重以后的结果；后者返回的元素是全集。
 注意：以上两个方法中，如果操作的属性为nil，在添加到数组中时会导致Crash。
 比如：`[arrBooks valueForKeyPath:@"@distinctUnionOfObjects.price"];`
 3）Array和Set嵌套操作符

```dart
@distinctUnionOfArrays
@unionOfArrays
@distinctUnionOfSets
```

这种操作是指数组嵌套数组或者Set嵌套Set等的操作，比如：

```objective-c
NSArray *bookArrs = @[arrBooks,arrBooks];
[bookArrs valueForKeyPath:@"@distinctUnionOfArrays.price"];
```

这样就可以取出来多个Book数组中price不同的对象，是不是很赞？



[iOS底层原理探究之-KVC](https://www.jianshu.com/p/7c818767344b)