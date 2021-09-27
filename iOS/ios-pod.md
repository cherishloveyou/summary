# iOS Pod相关记录



xcode12   静态库pod lib lint报错

```jsx
** BUILD FAILED **
 The following build commands failed:
 Ld ***/Release-iphonesimulator/App.build/Objects-normal/arm64/Binary/App normal arm64
```

![img](https:////upload-images.jianshu.io/upload_images/794643-287dfaa379928662.png?imageMogr2/auto-orient/strip|imageView2/2/w/701)

解决办法：podspec中添加

```dart
s.pod_target_xcconfig = {'EXCLUDED_ARCHS[sdk=iphonesimulator*]' => 'arm64'}
s.user_target_xcconfig = {'EXCLUDED_ARCHS[sdk=iphonesimulator*]' => 'arm64'}
```

### Include of non-modular header inside framework module ‘xxx’

因为Xcode在默认情况下是不允许在framework中的头文件引入一个不属于任何Module的头文件。

普通解决：将Build Settings中的Allow Non-modular Includes In Framework Modules设为YES

私有库别人集成可能就会有问题，此时建议添加以下配置，本质上和第一种方式类似；

```
s.user_target_xcconfig = { 'CLANG_ALLOW_NON_MODULAR_INCLUDES_IN_FRAMEWORK_MODULES' => 'YES' }
```

[Xcode12-lint-error](http://gonghonglou.com/2021/03/09/xcode12-lint-error/)

