### 背景
自研Jekins配置

- #### Debug 

```
#!/bin/zsh

#begin 

#clean
rm ./XXX.ipa
rm -rf Gemfile.lock
rm -rf Podfile.lock

if [ -f ".shell_ipa_path.txt" ]; then
  rm -rf .shell_ipa_path.txt
fi

echo "*****命令行*****"
echo $?

if [ $? != 1 ]; then

#rn bundle
./Scripts/getrnbundle_debug.sh

#bundle install
bundle install

#pod install
pod update --verbose

#Switch keychain
security unlock-keychain -p"123456" ${HOME}/Library/Keychains/login.keychain-db

agvtool new-version -all ${BUILD_NUMBER} 

pbpath="./XXX.xcodeproj/project.pbxproj"

#replacement certificates
sed -i '' s/Apple\ Distribution/Apple\ Development/g ${pbpath}

sed -i '' s/com.crgt.XXX\;/com.crgt.XXX.dev\;/g ${pbpath}
sed -i '' s/com.crgt.XXX.today\;/com.crgt.XXX.dev.today\;/g ${pbpath}
sed -i '' s/com.crgt.XXX.notification\;/com.crgt.XXX.dev.notification\;/g ${pbpath}
sed -i '' s/com.crgt.XXX.widget\;/com.crgt.XXX.dev.widget\;/g ${pbpath}

sed -i '' s/XXXDis/XXXDev/g ${pbpath}
sed -i '' s/XXXTodayDis/XXXTodayDev/g ${pbpath}
sed -i '' s/XXXNotificationDis/XXXNotificationDev/g ${pbpath}
sed -i '' s/XXXWidgetDis/XXXWidgetDev/g ${pbpath}

time=$(date +%Y%m%d%H_%M_%S)
export_path="./export/XXX${time}"
xcodebuild archive -archivePath ${export_path}/XXX.xcarchive -workspace XXX.xcworkspace -scheme XXX -configuration Debug DWARF_DSYM_FOLDER_PATH=$export_path/${BUILD_NUMBER}XXX.app.dSYM
xcodebuild -exportArchive -archivePath ${export_path}/XXX.xcarchive -exportPath ${export_path}/ipa -exportOptionsPlist ../ExportOptions/ExportOptionsDebug.plist

#export
cd $export_path
zip -r ${BUILD_NUMBER}XXX.app.dSYM.zip ${BUILD_NUMBER}XXX.app.dSYM
cd ../
cd ../

#upload
python3 ../upload_ipa.py $export_path $branch $user_name 

ipa_path=$export_path/ipa/XXX.ipa

echo "ipa_path=${ipa_path}" > .shell_ipa_path.txt
fi
```

- #### Distribution

```
#!/bin/zsh

#begin 
#clean

rm ./XXX.ipa

rm -r -f '../iOS_XXX_Zeus_Distribution_sec'

rm -rf Gemfile.lock

rm -rf Podfile.lock

if [ -f ".shell_ipa_path.txt" ]; then
  rm -rf .shell_ipa_path.txt
fi


echo "*****命令行*****"
echo $?

if [ $? != 1 ]; then

#rn bundle
./Scripts/getrnbundle_release.sh -rn_version $version

#bundle install
bundle install

#pod install
pod update --verbose


#secrity
echo $(pwd)
cd ../SCShield/native_tools_mac_2012201515
echo $(pwd)
./XXX_distribution.sh
echo $(pwd)
cd ../../iOS_XXX_Zeus_Distribution_sec
echo $(pwd)

# Switch keychain
security unlock-keychain -p"123456" ${HOME}/Library/Keychains/login.keychain-db

agvtool new-version -all ${BUILD_NUMBER} 

# bundle exec fastlane Distribution

pbpath="./XXX.xcodeproj/project.pbxproj"

#replacement certificates
sed -i '' s/Apple\ Development/Apple\ Distribution/g ${pbpath}
sed -i '' s/com.crgt.XXX.dev\;/com.crgt.XXX\;/g ${pbpath}
sed -i '' s/com.crgt.XXX.dev.today\;/com.crgt.XXX.today\;/g ${pbpath}
sed -i '' s/com.crgt.XXX.dev.notification\;/com.crgt.XXX.notification\;/g ${pbpath}
sed -i '' s/com.crgt.XXX.dev.widget\;/com.crgt.XXX.widget\;/g ${pbpath}
sed -i '' s/'MARKETING_VERSION = .*'/'MARKETING_VERSION ='${version}'\;'/g ${pbpath}

sed -i '' s/XXXDev/XXXDis/g ${pbpath}
sed -i '' s/XXXTodayDev/XXXTodayDis/g ${pbpath}
sed -i '' s/XXXNotificationDev/XXXNotificationDis/g ${pbpath}
sed -i '' s/XXXWidgetDev/XXXWidgetDis/g ${pbpath}

time=$(date +%Y%m%d%H_%M_%S)
export_path="./export/XXX${time}"
xcodebuild archive -archivePath ${export_path}/XXX.xcarchive -workspace XXX.xcworkspace -scheme XXX -configuration Release DWARF_DSYM_FOLDER_PATH=$export_path/${BUILD_NUMBER}XXX.app.dSYM
xcodebuild -exportArchive -archivePath ${export_path}/XXX.xcarchive -exportPath ${export_path}/ipa -exportOptionsPlist ../ExportOptions/ExportOptionsDistribution.plist

#upload ipa
xcrun altool --validate-app -t ios -f ${export_path}/ipa/XXX.ipa -u cong.he01@crgecent.com -p @keychain:MY_SECRET --output-format xml
xcrun altool --upload-app -t ios -f ${export_path}/ipa/XXX.ipa -u cong.he01@crgecent.com -p @keychain:MY_SECRET --output-format xml 

echo $export_path

cd $export_path

zip -r ${BUILD_NUMBER}XXX.app.dSYM.zip ${BUILD_NUMBER}XXX.app.dSYM

cd ../
cd ../

cp -r $export_path ../发布包资料存档/XXX${time}

ipa_path=$(pwd)/export/XXX${time}/ipa/XXX.ipa

echo "ipa_path=${ipa_path}" > .shell_ipa_path.txt

#bugly dsym
curl -k "https://api.bugly.qq.com/openapi/file/upload/symbol?app_key=XXXX&app_id=XXX" --form "api_version=1" --form "app_id=fdba631b92" --form "app_key=3ae87c07-4cb1-4927-885f-28b519942762" --form "symbolType=2"  --form "bundleId=com.crgt.XXX" --form "channel=appstore" --form "fileName=XXX${BUILD_NUMBER}.dSYM.zip" --form "file=@$export_path/${BUILD_NUMBER}XXX.app.dSYM.zip" --verbose
fi
```

