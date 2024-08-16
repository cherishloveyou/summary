# iOS 中 copy 的原理

在 `objc-accessor.mm` 中，有 property 中 copy 的实现。

这里有两个函数 `objc_copyStruct` 和 `objc_copyCppObjectAtomic`,分别对应结构体的拷贝和对象的拷贝。具体代码如下：

```c
/**
 * 结构体拷贝
 * src：源指针
 * dest：目标指针
 * size：大小
 * atomic：是否是原子操作
 * hasStrong：可能是表示是否是strong修饰
 */
void objc_copyStruct(void *dest, const void *src, ptrdiff_t size, BOOL atomic, BOOL hasStrong __unused) {
    spinlock_t *srcLock = nil;
    spinlock_t *dstLock = nil;
    // >> 如果是原子操作，则加锁
    if (atomic) {
        srcLock = &StructLocks[src];
        dstLock = &StructLocks[dest];
        spinlock_t::lockTwo(srcLock, dstLock);
    }
    // >> 实际的拷贝操作
    memmove(dest, src, size);
 
    // >> 解锁
    if (atomic) {
        spinlock_t::unlockTwo(srcLock, dstLock);
    }
}
 
/**
 * 对象拷贝
 * src：源指针
 * dest：目标指针
 * copyHelper：对对象进行实际拷贝的函数指针，参数是src和dest
*/
 
void objc_copyCppObjectAtomic(void *dest, const void *src, void (*copyHelper) (void *dest, const void *source)) {
    // >> 获取源指针的对象锁
    spinlock_t *srcLock = &CppObjectLocks[src];
    // >> 获取目标指针的对象锁
    spinlock_t *dstLock = &CppObjectLocks[dest];
    // >> 对源对象和目标对象进行上锁
    spinlock_t::lockTwo(srcLock, dstLock);
 
    // let C++ code perform the actual copy.
    // >> 调用函数指针对应的函数，让C++进行实际的拷贝操作
    copyHelper(dest, src);
    // >> 解锁
    spinlock_t::unlockTwo(srcLock, dstLock);
}
```



我们可以得出结论：

1. 对结构体进行 copy，直接对结构体指针所指向的内存进行拷贝即可；
2. 对对象进行 copy，则会传入的源指针和目标对象同时进行加锁，然后在去进行拷贝操作，所以我们可以知道，**copy 操作是线程安全的**。

不过这里的 `copyHelper(dest, src);`找不到实现方法，则有些遗憾。



### copy与mutablecopy

copy 方法：
copy 方法用于创建一个不可变的副本，无论原始对象是可变的还是不可变的。
对于不可变对象，copy 方法仅仅是增加了引用计数，返回的仍然是原始对象本身。
对于可变对象，copy 方法会创建一个新的不可变对象，内容与原始对象相同。

mutableCopy 方法：

mutableCopy 方法用于创建一个可变的副本，无论原始对象是可变的还是不可变的。
对于不可变对象，mutableCopy 方法会创建一个新的可变对象，内容与原始对象相同。
对于可变对象，mutableCopy 方法会创建一个新的可变对象，内容与原始对象相同。

总结一下，主要区别在于：

copy 返回的是不可变副本，而 mutableCopy 返回的是可变副本。
copy 对不可变对象和可变对象的行为不同，而 mutableCopy 则对所有对象的行为都一致

## **深浅拷贝**

- 浅拷贝：只创建一个新的指针，指向原指针指向的内存；
- 深拷贝：创建一个新的指针，并开辟新的内存空间，内容拷贝自原指针指向的内存，并指向它。



非容器对象：

| 可不可变对象   | copy类型    | 深浅拷贝 | 返回对象是否可变 |
| :----------- | :---------- | :------- | :--------------- |
| 不可变对象     | copy        | 浅拷贝   | 不可变           |
| 可变对象       | copy        | 深拷贝   | 不可变           |
| 不可变对象     | mutableCopy | 深拷贝   | 可变             |
| 可变对象       | mutableCopy | 深拷贝   | 可变             |

- 这里的源字符串如果是 Tagged Pointer 类型，即 NSTaggedPointerString，会有些有趣的情况，不过并不影响结果。
- 注意：接收 copy 结果的对象，也需要是可变的并且属性关键字是 strong，才可以进行修改，也就是可变，两个条件一个不符合则无法改变。



容器对象：

| 可不可变对象   | copy类型    | 深浅拷贝 | 返回对象可变 | 内部元素信息     | Info                                  |
| :------------| :---------- | :-------- | :--------------- | :--------------- | :------------------------------------ |
| 不可变对象     | copy        | 浅拷贝   | 不可变           | 内部元素是浅拷贝 | 集合地址不变                          |
| 可变对象       | copy        | 浅拷贝   | 不可变           | 内部元素是浅拷贝 | 集合地址改变 看起来是深拷贝，实际不是 |
| 不可变对象     | mutableCopy | 浅拷贝   | 可变             | 内部元素是浅拷贝 | 集合地址改变 看起来是深拷贝，实际不是 |
| 可变对象       | mutableCopy | 浅拷贝   | 可变             | 内部元素是浅拷贝 | 集合地址改变 看起来是深拷贝，实际不是 |

- 除了不可变对象使用 copy，其他的 copy 和 mutableCopy，都是开辟了一个新的集合空间，但是内部的元素的指针还是指向源地址。
- 有的人将集合地址改变的拷贝称之为深拷贝，但是这个其实是非常错误的理解，深拷贝就是全层次的拷贝。



[copy原理](https://blog.csdn.net/olsQ93038o99S/article/details/107420597)