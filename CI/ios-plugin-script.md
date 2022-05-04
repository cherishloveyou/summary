### 自动插件



### 背景
团队进行组件化开发后

```shell
# Type a script or drag a script file from your workspace to insert its path.

DEVICE_DIR=${BUILD_ROOT}/${CONFIGURATION}-iphoneos
DEST_DIR=${SRCROOT}/lib/IMLibs

libName="IMKit"
tempDir="IMKitLibDir"
#业务lib单独打lib其他的合并成一个
plugLib="libChat"

arch=(armv7 arm64)
 
if [ ! -d ${DEST_DIR} ];then
    mkdir -p ${DEST_DIR}
else
    echo "=======Begin========="
fi

cd ${DEVICE_DIR}

echo "step1:search package podlibs"
podlibs=(GNLUIKit GNLFoundation GNLMediator GNLNetwork GNBModuleActions IVGPicker IVGViewer)

for j in ${arch[*]}; do
    if [ ! -d $tempDir-$j ];then
       mkdir -p $tempDir-$j
    else
       rm -f $tempDir-$j
    fi

    for i in ${podlibs[*]}; do
        if [ -f $i/lib$i.a ];then
            lipo $i/lib$i.a -thin $j -output $tempDir-$j/$i-$j.a
        else
          echo "$i/lib$i.a podlib不存在"
        fi
    done
done

sleep 1
echo "step2:search package sourcelibs"
libs=(libCommon libBaseComponent libCoreModel libAppModel)

for j in ${arch[*]}; do

    if [ ! -d $tempDir-$j ];then
       mkdir /$tempDir-$j
    else
       echo "floder exit"
    fi
  
    for i in ${libs[*]}; do
      if [ -f $i.a ];then
        lipo $i.a -thin $j -output $tempDir-$j/$i-$j.a
      else
        echo "$i.a lib不存在"
      fi
    done
done

echo "step3: ar -x *.o file"
for j in ${arch[*]}; 
do
  cd $tempDir-$j
  for file in *.a 
  do
    if [ -f $file ];then
        file=${file#./}
        ar -x $file
    else
       echo "ar -x error"
    fi
  done
  cd ..  
done

echo "step4:merge *.o file to .a"
for j in ${arch[*]}; do
    cd $tempDir-$j
    libtool -static -o ../"lib${libName}-$j.a" *.o
    cd ..
done

echo "step5:merge all arch to one lib"
lipo -create "lib${libName}-armv7.a" "lib${libName}-arm64.a" -output "lib${libName}.a"


if [ -f "lib${libName}.a" ];then
    echo "build lib sucessful!"
    if [ -f "${plugLib}.a" ];then
       cp -R -f "${plugLib}.a" ${DEST_DIR}
    else 
       echo ""
    fi
    cp -R -f "lib${libName}.a" "${DEST_DIR}/libCommon.a"
    open ${DEST_DIR}
    rm -f "lib${libName}.a"
    if [ ! "lib${libName}-armv7.a" ];then
      echo ""
    else
      rm -f "lib${libName}-armv7.a"
      rm -f "lib${libName}-arm64.a"
    fi
    for j in ${arch[*]}; do
      rm -rf $tempDir-$j
    done
else
    echo "build lib failure!"
fi
```
