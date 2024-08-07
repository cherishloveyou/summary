## WebRTC编译

```shell
安装depot_tools，配置环境变量

git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=$PATH:/Users/moon/Documents/GitHub/depot_tools

下载WebRTC源码
#cd 到希望放置源代码的目录 执行以下命令
fetch --nohooks webrtc_ios
#执行完毕上面命令后
gclient sync

#cd SRC
gn gen out/ios --args='target_os="ios" target_cpu="arm64" is_debug=true'
ninja -C out/ios framework_objc
或
gn gen out/ios --args='target_os="ios" target_cpu="arm64" is_debug=true'
python tools_webrtc/ios/build_ios_libs.py --bitcode

gn gen out/mac-release --args='target_os="mac" target_cpu="x64" is_debug=false use_rtti=true is_component_build=false rtc_use_h264=false rtc_include_tests=false' --ide=xcode

ninja -C out/mac-release

gn gen out/mac-release --args='target_os="mac" target_cpu="x64" is_debug=false use_rtti=true is_component_build=false rtc_use_h264=false rtc_include_tests=false' --ide=xcode

#执行完毕后可以在src/out/ios目录中看到生成的项目工程

```

**MAC编译**

```shell
gn gen out/mac-release --args='target_os="mac" target_cpu="x64" is_debug=false use_rtti=true is_component_build=false rtc_use_h264=false rtc_include_tests=false' --ide=xcode

ninja -C out/mac-release

#编译成功后会在src\out\xxxx\下生成all.xcworkspace文件。打开就可以构建、调试webrtc的项目。其中APPRTCMobile是谷歌提供的示例demo，可以在Mac下直接编译运行
```

**iOS编译**

```shell
# 编译不带证书版本
gn gen out/ios-release --args='target_os="ios" target_cpu="arm64" is_debug=false use_rtti=true is_component_build=false ios_enable_code_signing=false proprietary_codecs=false rtc_use_h264=false rtc_include_tests=false' --ide=xcode
ninja -C out/ios-release

# 获取证书名
security find-identity -v -p codesigning

# 编译带证书版本
gn gen out/ios-release-sign --args='target_os="ios" target_cpu="arm64" is_debug=false use_rtti=true is_component_build=false ios_code_signing_identity="上面命令获取到的那串数字" proprietary_codecs=false rtc_use_h264=false rtc_include_tests=false' --ide=xcode
ninja -C out/ios-release-sign

#编译成功后，会在src\out\xxxx\下生成all.xcworkspace文件。打开就可以构建、调试webrtc的项目。其中APPRTCMobile是谷歌提供的示例demo，可打包在真机上运行。在src\out\xxxx\也生成了WebRTC.framework库文件，在外部项目中引用该库文件就可以使用其音视频能力了。



WebRTC.framework库文件也可以通过ninja命令或者python脚本单独生成。
# 通过ninja命令单独生成WebRTC.framework库文件
ninja -C out/ios-release-sign framework_objc

# 通过build_ios_libs.py脚本生成WebRTC.framework库文件
python tools_webrtc/ios/build_ios_libs.py --bitcode

```





[Mac下编译WebRTC（Mac和iOS)](https://blog.csdn.net/caesar1228/article/details/122146162?spm=1001.2101.3001.6650.1&amp;utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-122146162-blog-120485864.235%5Ev38%5Epc_relevant_yljh&amp;depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-122146162-blog-120485864.235%5Ev38%5Epc_relevant_yljh&amp;utm_relevant_index=2 )
