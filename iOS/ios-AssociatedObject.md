# AssociatedObject关联对象的内部实现

存储关联对象的是一个全局的hash表，通过将object`DISGUISE`转化作为key来索引object的关联关系hash表

对象的关联关系hash表，是通过我们传入的key来索引关联对象的



**为什么要引入关联对象？**

> - 一般我们需要对现有的类做扩展，可以通过继承、类别等方式去实现；当我们使用类别的方式扩展，如果对现有的类增加属性的话，编译器是不会生成实例变量；类别的结构体中没有ivar的结构体，同时类的ivar设计的是一个const
> - 类别是运行时装载到类中的，当类realizeClass之后它的`instanceSize`就已经确定无法修改了，这些操作都是在load之前，main函数之前
> - 如果想通过runtime的方法class_addIvar它只适用于新建一个类的时候增加，对于类别中增加实例就不适用
> - 关联对象就是在不改变类的结构的情况下，将类需要关联的对象存储在关联表中，那么类别中添加的属性的值的存取就可以通过关联来解决



关联对象相关的类有四个**AssociationsManager、AssociationsHashMap、ObjectAssociationMap、ObjcAssociation**，整个结构是一个二级哈希表，一个实例对象对应一个ObjectAssociationMap，而ObjectAssociationMap中存储着此实例对象的所有关联对象

#### objc_getAssociatedObject

 内部调用了`objc_getAssociatedObject`

```cpp
id objc_getAssociatedObject(id object, const void *key) {
    return _object_get_associative_reference(object, (void *)key);
}
```

可以看到内部就是根据`disguised_object`找到存储object的关联对象的hash表，然后从hash表中根据key找到对应的关联对象的值，再根据内存策略如果需要会相应的retain autorelease

**objc_removeAssociatedObjects** -- 删除object的所有的关联对象

同样的根据`disguised_object`找到存储object的关联对象的hash表，将hash表中的元素存储到elements中，然后删除hash表，最后遍历删除elements中的元素.

```objective-c
void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}
```

