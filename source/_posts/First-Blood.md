---
title: 在已有原生工程中集成Flutter
date: 2019-10-10 13:37:24
tags:
  - Flutter
  - iOS
categories: flutter
---

##### 创建Flutter模块

与纯Flutter工程不同，在原生工程中接入Flutter，是以 `Flutter Module` 的形式接入的。

假设我们已有一个iOS工程在 `some/path/MyApp` 路径下，使用如下命令创建Flutter模块:

```bash
$ cd some/path/
$ flutter create -t module my_flutter
```

命令完成后，会在`some/path/my_flutter`下生成Flutter模块，目录下有隐藏的`.ios`包含了此模块的iOS Flutter工程。创建时要求Module名称为全小写，否则将创建失败。

```bash
my_flutter
├── .android
│   ├── Flutter
│   ├── app
│   ├── build.gradle
│   ├── gradle
│   ├── gradle.properties
│   ├── gradlew
│   ├── gradlew.bat
│   ├── include_flutter.groovy
│   ├── local.properties
│   └── settings.gradle
│
├── .ios # 该module的Flutter工程
│   ├── Config
│   ├── Flutter # 插件、App.framework、Flutter.framework生成的目录
│   ├── Runner
│   ├── Runner.xcodeproj
│   └── Runner.xcworkspace
│
├── README.md
│
├── lib
│   └── main.dart # 默认创建的带main函数入口的Dart文件
├── pubspec.lock # 完整依赖链和版本信息，类似Podfile.lock
└── pubspec.yaml # 依赖库管理，类似Podfile
```

##### 让宿主工程依赖Flutter模块

2.  在`Podfile`中添加如下内容:
    
    ```ruby
    flutter_application_path = 'path/to/my_flutter/'
    eval(File.read(File.join(flutter_application_path, '.ios', 'Flutter', 'podhelper.rb')), binding)
    ```
    
3.  运行`pod install`
    
    `podHelp.rb` 这个ruby脚本会关闭Pod工程内Flutter target的bitcode，并将`Flutter.framework`和Plugin相关的pod集成到工程中。
    
    当我们在`some/path/my_flutter/pubspec.yaml`改变Flutter插件依赖时，都需要运行`flutter packages get`来刷新`podhelper.rb`中的插件列表，然后再运行`pod install`来更新宿主工程。
    

```ruby
   # Ensure that ENABLE_BITCODE is set to NO, add a #include to Generated.xcconfig, and
   # add a run script to the Build Phases.
   post_install do |installer|
       installer.pods_project.targets.each do |target|
           target.build_configurations.each do |config|
               # 关闭pod工程的bitcode
               config.build_settings['ENABLE_BITCODE'] = 'NO'
               next if  config.base_configuration_reference == nil
               xcconfig_path = config.base_configuration_reference.real_path
               File.open(xcconfig_path, 'a+') do |file|
                   # 在xcconfig中导入Flutter的xcconfig
                   file.puts "#include \"#{File.realpath(File.join(framework_dir, 'Generated.xcconfig'))}\""
               end
           end
       end
   end
```

另外由于`podHelp.rb`是通过 `post_install hook` 来关闭pod工程的bitcode和引入`Generated.xcconfig` ，如果原生工程的 `Podfile` 中也使用了 `post_install` 则两者会冲突，解决方式可以将 `podHelp.rb` 中`post_install` 所处理的逻辑封装成一个方法，在工程的 `Podfile` 中调用。

4.  关闭原生工程的bitcode。由于Flutter并不支持bitcode，且脚本只是关闭了pod工程的bitcode，因此需要再将原生工程的bitcode关闭。

##### 添加build phase来构建Dart代码

在原生宿主工程的`Build Phase`中，添加`New Run Script Phase`，输入脚本内容：

```bash
# build模式 会构建App.framework，将Flutter.framework放置到engine目录中
# embed模式 会将两个framework拷贝到.app内的Frameworks目录中

"$FLUTTER_ROOT/packages/flutter_tools/bin/xcode_backend.sh" build
"$FLUTTER_ROOT/packages/flutter_tools/bin/xcode_backend.sh" embed
```

确保新添加的`Run Script`在`Target Denpendenies`之后，现在我们可以编译集成Flutter之后的工程了。脚本会将Dart代码编译成`App.framework`嵌入工程中。

##### 在原生工程中年使用`FlutterViewController`

在 `AppDelegate.h` 中，修改继承关系，让它继承自`FlutterAppDelegate` ，添加 `FlutterEngine`类型的属性：

```objectivec
#import 
#import 

@interface AppDelegate : FlutterAppDelegate
@property (nonatomic,strong) FlutterEngine *flutterEngine;
@end
```

重写`application:didFinishLaunchingWithOptions:`方法：

```objectivec
#import  // Only if you have Flutter Plugins

#include "AppDelegate.h"

@implementation AppDelegate

// This override can be omitted if you do not have any Flutter Plugins.
- (BOOL)application:(UIApplication *)application
    didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // 初始化Flutter引擎
  self.flutterEngine = [[FlutterEngine alloc] initWithName:@"io.flutter" project:nil];
  [self.flutterEngine runWithEntrypoint:nil];
  // 注册插件
  [GeneratedPluginRegistrant registerWithRegistry:self.flutterEngine];
  return [super application:application didFinishLaunchingWithOptions:launchOptions];
}

@end
```

使用`FlutterViewController`：

```objectivec
#import 
#import "AppDelegate.h"
#import "ViewController.h"

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    UIButton *button = [UIButton buttonWithType:UIButtonTypeCustom];
    [button addTarget:self
               action:@selector(handleButtonAction)
     forControlEvents:UIControlEventTouchUpInside];
    [button setTitle:@"Press me" forState:UIControlStateNormal];
    [button setBackgroundColor:[UIColor blueColor]];
    button.frame = CGRectMake(80.0, 210.0, 160.0, 40.0);
    [self.view addSubview:button];
}

- (void)handleButtonAction {
    FlutterEngine *flutterEngine = [(AppDelegate *)[[UIApplication sharedApplication] delegate] flutterEngine];
    FlutterViewController *flutterViewController = [[FlutterViewController alloc] initWithEngine:flutterEngine nibName:nil bundle:nil];
    [self presentViewController:flutterViewController animated:false completion:nil];
}
@end
```

到这里，一个集成Flutter的原生工程就能正常运行了，点击VC上的按钮将进入一个Flutter的页面中。

当编写了多个widget时，我们也可以使用路由来展示不同的widgets：

```objectivec
[flutterViewController setInitialRoute:@"route1"];
```

如果应用的`AppDelegate`已经继承自某个基类，且不方便直接修改继承关系，则需要自己去实现`FlutterAppLifeCycleProvider`协议。

##### 热加载

在混编工程中，也可以使用热加载。区别于纯Flutter工程的完善的热加载支持，混编工程的热加载需要通过命令行工具和Dart Observatory的web页面。

首先使用 `flutter attach` 让flutter等待连接

```bash
cd path/to/flutter_module
flutter attach -d device-uuid
Waiting for a connection from Flutter on iPhone 7...
```

然后在原生工程中Run起工程，然后让flutter附加到进程上。

attach成功后，就可以编辑flutter module中的Dart代码，在命令行中输入`r`即可热加载。

并且热加载需要管理员权限来执行 `stream` 操作，当没有管理员权限时在终端执行`flutter attach` 会在等待一段时间后自动断开，容易让人不知所以。而在VS Code中对纯Flutter工程进行热加载调试时，会有相应的提示。这也体现了千面所说的两种混编工程热加载支持还所有不足。

## Flutter与原生相互调用

##### Flutter调用原生方法

Flutter与原生之间的调用是以`MethodChannel`为桥梁来完成的 ，在Native中通过`FlutterMethodChannel`设置代码调用逻辑`callHandler`，在Flutter中通过`MethodChannel`来触发原生代码。

原生代码：

```objectivec
#import 

FlutterEngine *flutterEngine = [(AppDelegate *)[[UIApplication sharedApplication] delegate] engine];
    NSString *channelName = @"callNative";
    FlutterMethodChannel *methodChannel = [FlutterMethodChannel methodChannelWithName:channelName binaryMessenger:flutterEngine];
    // 处理fullter调用
    [methodChannel setMethodCallHandler:^(FlutterMethodCall * _Nonnull call, FlutterResult _Nonnull result) {
        if ([call.method isEqualToString:@"changeColor"]) {
            UIColor *randomColor = [UIColor colorWithRed:arc4random_uniform(255) / 255. green:arc4random_uniform(255) / 255. blue:arc4random_uniform(255) / 255. alpha:1];
            self.view.backgroundColor = randomColor;
            result(@"Native color did changed!");
        }
        else {
            result(FlutterMethodNotImplemented);
        }
    }];
```

Flutter侧代码：

```dart
import 'package:flutter/services.dart';

MaterialButton(
  color: Colors.blue,
  textColor: Colors.white,
  child: Text("call native"),
  onPressed: () async {
    var methodChannel = MethodChannel('callNative');
    String res =  await methodChannel.invokeMethod('changeColor');
    Toast.show(res, context, duration: Toast.LENGTH_LONG);
  },
)
```

##### 原生调用Flutter

原生调用Flutter的方式与上面一致，双方互换，在Flutter侧设置`callHandler`，在原生通过`FlutterMethodChannel`调用相关方法。

Flutter代码:

```dart
import 'package:flutter/services.dart';

Future changeCount (MethodCall call) {
  if (call.method == 'changeCount') {
    var args = call.arguments;
    setState(() {
      _counter = args[0];
    });
    return Future.value("Flutter count did change!");
  }
  throw MissingPluginException();
}

var channel = MethodChannel('callFlutter');
channel.setMethodCallHandler(changeCount);
```

原生代码:

```objectivec
#import 

FlutterEngine *flutterEngine = [(AppDelegate *)[[UIApplication sharedApplication] delegate] engine];
NSString *channelName = @"callFlutter";
FlutterMethodChannel *methodChannel = [FlutterMethodChannel methodChannelWithName:channelName binaryMessenger:flutterEngine];
[methodChannel invokeMethod:@"changeCount" arguments:@[@(arc4random_uniform(255))] result:^(id  _Nullable result) {
    NSString *message;
    if ([result isKindOfClass:[FlutterError class]]) {
        message = @"Flutter运行失败";
    }
    else if ([result isEqual:FlutterMethodNotImplemented]) {
        message = @"Flutter 未实现此方法";
    } else {
        message = result ? : @"Flutter调用成功";
    }
    [self.view makeToast:message];
}];
```

Flutter调用原生方法时，可以获取到原生`callHandler`中设置的结果作为返回值。

原生调用Flutter时，在`invokeMethod:arguments:result:`方法中的`result` 获取返回值。当Flutter运行失败时，`result`回调中会是一个`FlutterError`实例，如果Flutter侧未实现该方法，则会是一个`FlutterMethodNotImplemented`对象，其他值包括`nil`均表示调用成功。

## 隔离开发环境

假设团队中有一部分负责Flutter开发，其他人还是在原生环境下开发，按上面的方式把Flutter接入到一个工程中后，如果要团队的所有人员都能正常运行这个工程，则需要所有人都配置Flutter环境。

因此，需要将Flutter的开发环境与原生的开发环境通隔离，过Pod的方式将Flutter的所有产物导入到原生工程中。

Flutter产物主要有如下几个：

-   Flutter引擎相关的 `Flutter.framework`
    
-   Dart 代码相关的 `App.framework`
    
-   插件注册入口 `FlutterPluginRegistrant`
    
-   原生相关插件 plugins
    

通过接入了Flutter的混编工程可以看到，`Flutter.framework` 、`FlutterPluginRegistrant` 和各个插件都是以pod的形式接入，而 `App.framework` 是通过 `xcode_backend.sh` 脚本从Flutter Module中拷贝到App内的。

因此我们主要处理收集 `App.framework` 并提供podspec文件即可。

这里示例脚本收集了`App.framework` 、 `Flutter.framework` ，并将插件相关的产物编译为静态库。收集完成后将podspec修改为本地pod的方式方便验证。

```bash
#!/bin/bash
#

export FLUTTER_ROOT=/Users/lindubo505/Documents/flutter
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
export PATH="$PATH:/Users/lindubo505/Documents/flutter/bin"


if [ ! -d "./product" ];then
mkdir ./product
else
echo "文件夹已经存在"
rm -rf ./product/*
fi

config='Debug'
appConfig="$(echo $config | tr '[:upper:]' '[:lower:]')"

echo "===清理Flutter历史编译==="
flutter clean

echo "===重新生成plugin索引==="
flutter packages get

# 删除原生成的App.framework
rm -rf ./.ios/Flutter/App.framework
echo "===生成App.framework==="
echo "flutter build ios --${appConfig}"
flutter build ios --${appConfig}

# 创建放置产物的目录
if [ ! -d "./product/app" ];then
mkdir ./product/app
fi

# 收集App.framework，生成podspec
cp -fr ./.ios/Flutter/App.framework ./product/app
cp -f $FLUTTER_ROOT/App.podspec ./product/app

# 收集Flutter.framework与podspec
echo "===拷贝Flutter.framework==="
cp -fr ./.ios/Flutter/engine ./product/engine


echo "===生成plugin的静态库==="

while read -r line
do 
    if [[ ! "$line" =~ ^// ]]; then
        array=(${line//=/ })
        plugin_name=${array[0]}
        cd .ios/Pods
        echo "生成lib${plugin_name}.a..."
        /usr/bin/env xcrun xcodebuild build -configuration ${config} ARCHS='arm64 armv7' -target ${plugin_name} BUILD_DIR=../../build/ios -sdk iphoneos -quiet
        /usr/bin/env xcrun xcodebuild build -configuration ${config} ARCHS='x86_64' -target ${plugin_name} BUILD_DIR=../../build/ios -sdk iphonesimulator -quiet
        echo "合并lib${plugin_name}.a..."
        if [[ ! -d "../../product/lib${plugin_name}" ]]; then
            mkdir "../../product/lib${plugin_name}"
        fi
        lipo -create "../../build/ios/${config}-iphonesimulator/${plugin_name}/lib${plugin_name}.a" "../../build/ios/${config}-iphoneos/${plugin_name}/lib${plugin_name}.a" -o "../../product/lib${plugin_name}/lib${plugin_name}.a"
        echo "复制头文件"
        classes=${array[1]}ios/Classes
        for header in `find "$classes" -name *.h`; do
            cp -f $header "../../product/lib${plugin_name}/"
        done

        echo "复制podspec文件"
        specDir=${array[1]}ios
        for spec in `find "$specDir" -name *.podspec`; do
            cp -f $spec "../../product/lib${plugin_name}/"
        done
    else 
        echo "读取文件出错"
    fi
done < .flutter-plugins

echo "===生成注册入口的二进制库文件==="
for reg_enter_name in "FlutterPluginRegistrant"
do
    echo "生成libFlutterPluginRegistrant.a..."
    /usr/bin/env xcrun xcodebuild build -configuration ${config} ARCHS='arm64 armv7' -target FlutterPluginRegistrant BUILD_DIR=../../build/ios -sdk iphoneos -quiet
    /usr/bin/env xcrun xcodebuild build -configuration ${config} ARCHS='x86_64' -target FlutterPluginRegistrant BUILD_DIR=../../build/ios -sdk iphonesimulator -quiet

    if [[ ! -d "../../product/lib${reg_enter_name}" ]]; then
        mkdir "../../product/lib${reg_enter_name}"
    fi

    echo "合并lib${reg_enter_name}.a..."
    lipo -create "../../build/ios/${config}-iphonesimulator/$reg_enter_name/lib${reg_enter_name}.a" "../../build/ios/${config}-iphoneos/${reg_enter_name}/lib${reg_enter_name}.a" -o "../../product/lib${reg_enter_name}/lib${reg_enter_name}.a"
    echo "复制头文件"
    classes="../Flutter/${reg_enter_name}/Classes"
    for header in `find "$classes" -name *.h`; do
        cp -f $header "../../product/lib${reg_enter_name}/"
    done

    echo "复制podspec文件"
    specDir="../Flutter/${reg_enter_name}"
    for spec in `find "$specDir" -name *.podspec`; do
        cp -f $spec "../../product/lib${reg_enter_name}/"
    done
done

# 修改podspec为本地pod
python ../../changePodSpec.py ../../product

# podspec
# 替换原s.source  => s.source           = { :path => '.' }

# 增加s.vendored_libraries = './*.a'

# 替换s.source_files = 'Classes/**/*',为 s.source_files = './*.{h}'
# 删除s.public_header_files = 'Classes/**/*.h'

# sed  -n ‘s/^jcdd/ganji/p’ device_info.podspec
# sed -n '/^[\s]*s.source[\s]+[\d\D]*}/d' device_info.podspec
# sed 's/^[\s]*s.source[\s]+[\d\D]*}/replace/g' device_info.podspec


```

`changePodSpec.py`:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

import os,sys,re


def replaceOrAdd(content, pattern, mark, new_desc):
	print("pattern: %s" % pattern)
	match = re.search(pattern, content, flags=re.MULTILINE)
	if match:
		res = match.group()
		print("match: %s" % res)
		content = content.replace(res, new_desc)
	else:
		print("unmatch: %s" % pattern)
		content = content.replace(mark, mark + '\n  ' + new_desc)
	return content

def modifyspecsInDir(floder):
	output = os.popen('find %s -name "*.podspec"' % floder)
	allLines = output.readlines()

	for line in allLines:
		specPath = line.strip()
		with open(specPath, 'r+') as f:
			content = f.read()
			# 替换原s.source  => s.source           = { :path => '.' }
			# 替换s.source_files = 'Classes/**/*',为 s.source_files = '*.{h}'
			# 增加s.vendored_libraries = '*.a'
			# 删除s.public_header_files = 'Classes/**/*.h'
			# re.sub(pattern,repl,string,count,flags)

			# re.compile(r'%s = [-\\.$()/\w]+?;' % childrenNodeName)
			# version_pattern = "s\\.version[\\s]+[\\d\\D]+?[}'\"]{1}$"
			version_pattern = r"s\.version[\s]+[\d\D]+?['}\"]$"
			match = re.search(version_pattern, content, flags=re.MULTILINE)
			try:
				version_des = match.group()
			except Exception as e:
				raise e
			

			source_pattern = r"s\.source[\s]+[\d\D]+?['}\"]$"
			source_desc = r"s.source = { :path => '.' }"

			source_files_pattern = r"s\.source_files[\s]+[\d\D]+?['}\"]$"
			source_files_desc = r"s.source_files = '*.{h}'"

			public_header_files_pattern = r"s\.public_header_files[\s]+[\d\D]+?['}\"]$"

			vendored_libraries_pattern = r"s\.vendored_libraries[\s]+[\d\D]+?['}\"]$"
			vendored_libraries_desc = r"s.vendored_libraries = '*.a'"

			

			content = replaceOrAdd(content, source_pattern,version_des, source_desc)

			index = content.find("s.vendored_frameworks")
			if index < 0:
				print("not framework podspec")
				content = replaceOrAdd(content, source_files_pattern, version_des, source_files_desc)
				content = replaceOrAdd(content, vendored_libraries_pattern, version_des, vendored_libraries_desc)
				content = replaceOrAdd(content, public_header_files_pattern, version_des, '')					
			else:
				print('skip framework podspec')



			f.seek(0, 0)
			f.truncate()
			f.write(content)



if __name__ == '__main__':
	try:
		floder = sys.argv[1]
		# floder = '/Users/lindubo505/Desktop/NativeApp/flutter_module/product'
		modifyspecsInDir(os.path.abspath(floder))
	except Exception as e:
		raise e
```

收集完成后，修改工程的Podfile

删除:

```ruby
  flutter_application_path = '/Users/lindubo505/Desktop/NativeApp/flutter_module'
  eval(File.read(File.join(flutter_application_path, '.ios', 'Flutter', 'podhelper.rb')), binding)
```

改为依赖本地pod

```ruby
  pod 'Flutter', :path => 'flutter_module/product/engine'
  pod 'App', :path => 'flutter_module/product/app'
  pod 'device_info', :path => 'flutter_module/product/libdevice_info'
  pod 'FlutterPluginRegistrant', :path => 'flutter_module/product/libFlutterPluginRegistrant'
  pod 'image_picker', :path => 'flutter_module/product/libimage_picker'
```

最后删除工程`Build Phase` 中添加的 `Run Script`

```bash
"$FLUTTER_ROOT/packages/flutter_tools/bin/xcode_backend.sh" build
"$FLUTTER_ROOT/packages/flutter_tools/bin/xcode_backend.sh" embed
```

## 接入成本

##### 包体积变大

| IPA包体积 | 大小 | 备注 |
| --- | --- | --- |
| 初始化原生工程，无Flutter | 25 KB |  |
| 接入Flutter | 17.6 MB | 主要差异为：<br/>  
1. App.framework 6.1MB<br/>  
2. Flutter.framework 37.9 MB |

##### 内存增长

| 内存 | 使用情况 | 备注 |
| --- | --- | --- |
| 原生页面 | 12.7 M | iPhone 7 12.3.1 |
| 原生页面+Flutter页面 | 83.2 M | iPhone 7 12.3.1 |

##### 无法跨越原生代码

`Flutter`虽然具有跨平台能力，但实际开发中并躲不开原生代码，这反而对`Flutter`开发者提出了更高的要求，需要去了解两个平台的知识，理想中一个`Flutter`开发顶替两个终端开发并不存在。

即使官方提供的Dart UI组件也区分了Google的`Material`和苹果的`Cuprtino`两种风格，如果想在两端展示各自风格的UI特点，会让开发者十分头痛。

##### 组合而非继承

`Flutter`提倡“组合”，而不是“继承”。在iOS开发中，我们经常会继承UIView，重写UIView的某个生命周期函数，再添加一些方法和属性，来完成一个自定义的View。但是在`Flutter`中这些都是不可能的——属性都是`final`的，例如你继承了了一个`Container`，你是不能在它的生命周期中修改他的属性的。你始终需要嵌套组合几种`Widget`，例如`Row`，`Container`，`ListView`等`Widget`。这种方法非常不符合直觉，初学时很难想明白如何构建一个完整的组件。

并且这样组合模式让Dart内的UI代码显得非常不简洁，层层的括号嵌套让人眼花缭乱，尤其是在对比SwiftUI后更加明显。

```dart
SliverList(
  delegate: SliverChildBuilderDelegate(
    (context, index) {
      final landmark = landmarks[index];
      return LandmarkCell(
        landmark: landmark,
        onTap: () {
          Navigator.push(
            context,
            CupertinoPageRoute(
              builder: (context) => LandmarkDetail(
                landmark: landmark,
              ),
            ),
          );
        },
      );
    },
    childCount: landmarks.length,
  ),
)
```

```swift
ForEach(userData.landmarks) { landmark in
  NavigationButton( destination: LandmarkDetail(landmark: landmark))
 {
    LandmarkRow(landmark: landmark)
  }
}
```

两者都实现了一个可以点击的列表页，都调用了一个自定义 Cell 。

##### 其他

官方在[Flutter 2019 RoadMap](https://github.com/flutter/flutter/wiki/Roadmap)中也提出了2019年的计划：

-   基础优化，包括bug修复、性能表现、错误信息、API文档等方面。
    
-   易于接入，提供新的module模板、更友好的文档页面、提供最佳实践、丰富iOS风格组件
    
-   生态系统，更好的C/C++支持、更多官方插件包、map/WebView组件完善、Google service接入、本地数据存储等。
    
-   工具完善，编辑器支持的完善、支持 **[Language Server Protocol](https://langserver.org/)** 、[Dart DevTools](https://flutter.github.io/devtools/)完善等。
    

参考:

[Add-Flutter-to-existing-apps]([https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps)

[Flutter Platform Channels](https://cloud.tencent.com/developer/article/1415221)

[有赞 Flutter 混编方案]([https://tech.youzan.com/you-zan-flutter-hun-bian-fang-an/](https://tech.youzan.com/you-zan-flutter-hun-bian-fang-an/)

[闲鱼Flutter混合工程持续集成的最佳实践]([https://juejin.im/post/5b5811f3e51d4519700f6979](https://juejin.im/post/5b5811f3e51d4519700f6979)

[SwiftUI vs Flutter](https://juejin.im/post/5d05b45bf265da1bcc193ff4)

[浅谈Flutter的优缺点]([https://segmentfault.com/a/1190000017164263](https://segmentfault.com/a/1190000017164263)
