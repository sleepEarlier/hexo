---
title: iOS越狱环境搭建
date: 2021-05-17 17:54:01
tags:
---

# 越狱环境搭建与使用

## 一、越狱

使用 [unc0ver](https://www.unc0ver.dev/) ，目前支持iOS 11.0 - 14.3，将它提供的 IPA 安装到设备上，打开即可进行一键越狱。官网提供了各种安装方式，采用最简单的重签名方式即可。

二、Cydia源与应用

越狱完成后就已经有了Cydia，添加各类源、安装各种插件与应用。

```
# 源
https://apt.binger.com/

https://build.firda.re/

https://getdelta.co/

http://apt.thebigboss.org/reprofiles/cydia/

Https://repo.dynastic.co/

Https://cydia.angelxwind.net/

http://apt.modmyi.com/

https://repo.chariz.com

https://repo.incendo.ws/

https://tigisoftware.com/cydia/

https://cydia.zodttd.com/repo/cydia/

# 应用、插件
Flex3、Flexible、OpenSSH、Frida

AppSync Unified
源：http://cydia.angelxwind.net/
作用：允许安装未签名或破解的App

Apple File Conduit "2" 
源：https://apt.xbsite.cn
作用：允许通过USB访问手机系统目录

AppList
RocketBootstrap
Preferenceloader
源：https://rpetri.ch/repo/
作用：允许插件访问设置列表及依赖

Filza
源：http://tigisoftware.com/cydia/
作用：越狱设备必备文件管理器

Flex
源：http://getdelta.co/
作用：图形化UI调整工具，必备


ReProvision 
源：http://repo.incendo.ws
作用：自动重签名应用工具

P佬源
源：https://pulandres.me/repo/
作用：提供各种热门和谐版插件
```

## 三、连接iPhone与查看信息

通过SSH连接越狱设备，将iOS设备与电脑置于同一局域网下，默认密码 `alpine`

```
$ ssh root@10.252.24.149
root@10.252.24.149's password:
Black:~ root#
```

所有的应用位于 `/var/mobile/Containers/Data/Application/` 下

```
Black:~ root# cd /var/mobile/Containers/Data/Application/
Black:/var/mobile/Containers/Data/Application root# ls

# 输出
013FD0C1-6076-4085-BA69-649770327965/  3EF80435-AE63-48A7-854B-1CC5A124FD1D/  7EEA33EE-345E-4F80-919F-C21D294D03C4/  BC003F9B-534F-47CD-AA2A-0CB58767A0ED/
0163D32A-24C6-44C4-BC98-F8E909F69AEF/  3F628105-64BF-4B30-9DD6-D60D6296EF05/
```

每个应用沙盒都是一个UUID，并不能直接与应用对应上，可以借助Cycript来查看



cycript使用

ssh连接iOS设备后，输入cycript启动cycript（出现 `cy#`提示符）

ps -e | grep 'SpringBoard' 获取进程id，可以将所有进程杀死，仅打开目标进程后执行，减少干扰项

cycript - p 1184（进程id） 注入进程

借助cycript在页面中展示一个弹窗

```
cy# alertView = [[UIAlertView alloc] initWithTitle:@"test" message:@"Cyrill" delegate:nil cancelButtonTitle:@"OK" otherButtonTitles:nil]
#"<UIAlertView: 0x156f9f0a0; frame = (0 0; 0 0); layer = <CALayer: 0x156f96fc0>>"
cy# [alertView show]
```

如果知道内存地址，也可以直接调用

```
cy# [#0x156f9f0a0 show]
```



## 砸壳

### 方式一: [dumpdecrypted](https://link.jianshu.com/?t=https://github.com/stefanesser/dumpdecrypted)

1. 越狱iOS设备安装 [Cycript]http://www.cycript.org/) 与 [OpenSSH]https://www.openssh.com/)

2. 在Mac上 `git clone https://github.com/stefanesser/dumpdecrypted` 

3. cd并执行 `make` 得到 `dumpdecrypted.dylib` 

4. ssh连接到iOS设备

5. `ps -e` 找到目前进程路径

   ```shell
   root# ps -e
   1091 ??         0:03.35 /var/mobile/Containers/Bundle/Application/A5960257-7E26-45EC-A28C-315FCBB15852/WeChat.app/WeChat
   ```

6. 使用 `cycript` 找到应用沙盒

   ```
   root# cycript -p WeChat
   // 将输出WeChat沙盒路径，如
   /var/mobile/Containers/Data/Application/05330136-5902-48C1-96A6-3C03F275DC73/Documents
   ```

7. Mac上使用 scp 或 远程文件管理将前面得到的 `dylib` 拷贝到沙盒中

   ```
   scp ./dumpdecrypted.dylib root@192.168.2.2:/var/mobile/Containers/Data/Application/05330136-5902-48C1-96A6-3C03F275DC73/Documents
   ```

8. 进入到沙盒，注入动态库

   ```
   $ cd /var/mobile/Containers/Data/Application/05330136-5902-48C1-96A6-3C03F275DC73/Documents$ DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Containers/Bundle/Application/A5960257-7E26-45EC-A28C-315FCBB15852/WeChat.app/WeChat
   ```

   执行完成后，会在沙盒 `Documents` 内生成 `WeChat.decrypted` 文件

9. 将 `WeChat.decrypted` 拷贝到 mac 中

   ```
   scp root@192.168.2.2:/var/mobile/Containers/Data/Application/05330136-5902-48C1-96A6-3C03F275DC73/Documents/WeChat.decrypted ~/dumpdecrypted
   ```

 `WeChat.decrypted` 文件可用于 `class-dump` 或 `mach-o view` ，但不能用于跑 MonkeyDev。

## 方式二: [Clutch](https://github.com/KJCracks/Clutch)

不适用所有App，对于有 Extension 的app会失败，考虑使用 [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump)

1. 在 [release](https://github.com/KJCracks/Clutch/releases) 页下载最新版本，或下载源码自行编译
2. 将Clutch scp到iOS设备的 `/usr/bin` 下
3. 添加可执行权限 `chmod +x /usr/bin/Clutch`
4. 列出已安装的app `Clutch -i`
5. 砸壳 `Clutch -d 序号或bundleId`
6. 结果中会给出ipa路径，将ipa scp到mac上

## 方式三：LLDB砸壳

[**如何优雅的在LLDB里dumpdecrypted**](http://4ch12dy.site/2020/02/26/lldb-how-to-dump-gracefully/lldb-how-to-dump-gracefully/)

## 方式三:  [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump)

1. iOS上配置

   1. Cydia添加源: https://build.frida.re
   2. 搜索安装 frida
   3. 在手机终端运行 `frida-ps -U` 检查是否安装成功，检查失败也可砸壳

2. mac端配置

   1. 支持 `Python 2.x` 和 `3.x` 一般已带有 `pip` ，如果没有则自行安装 `pip`

      ```
      brew install wgetwget https://bootstrap.pypa.io/get-pip.pysudo python get-pip.py清理残留: rm ~/get-pip.py
      ```

   2. 克隆

      ```
      git clone https://github.com/AloneMonkey/frida-ios-dump
      ```

   3. 安装依赖

      ```
      cd frida-ios-dumpsudo pip install -r requirements.txt --upgrade
      ```

   4. dump.py 中包含所使用的连接iOS用的用户名、密码、端口，如有需要进行修改，一般默认情况无需改动

      ```
      User = 'root'Password = 'alpine'Host = 'localhost'Port = 2222
      ```

   5. 运行 `usbmuxd` 或 `iproxy` 等通过 USB 转发 SSH 工具（默认 2222 -> 22），例如: `iproxy 2222 22`

   6. 列出安装的app `./dump.py -l` ，找到目标app复制bundleId

   7. 砸壳 `./dump.py bundleId`，完成后 IPA 会在 mac 当前目录下（此处也可以忽略上一步，直接使用应用中文名）

   8. 对于 SSH \ SCP 操作，确保将电脑上的公钥添加到iOS设备的 `~/.ssh/authorized_keys` 上

## 符号表恢复

[restore-symbol](https://github.com/tobefuturer/restore-symbol)

[restore-symbol](https://github.com/HeiTanBc/restore-symbol) 适配iOS14

[Frida调用栈符号恢复](http://4ch12dy.site/2019/07/02/xia0CallStackSymbols/xia0CallStackSymbols/)

此处以 restore-symbol 为例:

1. 克隆并编译

   ```
   git clone --recursive https://github.com/HeiTanBc/restore-symbol.gitcd restore-symbol && make ./restore-symbol
   ```

2. 符号表恢复

   ```
   ./restore-symbol /pathto/origin_mach_o_file -o /pathto/mach_o_with_symbol 
   ```

3. 将新的 mach-o 放入 app中重签名，或直接放入 MonkeyDev 的 `TargetApp` 目录中的 .app 中

## Block符号恢复

1. 下载Python脚本 [`search_oc_block/ida_search_block.py`](https://github.com/tobefuturer/restore-symbol/blob/master/search_oc_block/ida_search_block.py) 
2. 用IDA打开目标mach-o文件，等待分析完成
3. 菜单栏 `File-Script file...` 选择前面下载的脚本
4. 等待脚本运行完成，在mach-o同级目录下会生成 `block_symbol.json` 
5. 重新运行 `restore-symol` 

```
./restore-symbol /pathto/origin_mach_o_file -o /pathto/mach_o_with_symbol -j /pathto/block_symbol.json
```



_printHierarchy

```
[[[UIWindow keyWindow] rootViewController] _printHierarchy]<MMUINavigationController 0x18392800>, state: appeared, view: <UILayoutContainerView 0x17f33790>               | <WCAccountLoginLastUserViewController 0x18b52600>, state: appeared, view: <UIView 0x192740d0>
```

**[[**UIApp keyWindow**]** recursiveDescription**]**

手机连接Xcode调试后，Xcode会将 debugServer 放到 `/Developer/usr/bin/` 下，我们将其拷贝到mac下，分离出指定架构

```
lipo -thin arm64 ~/debugserver -output ~/debugserver
```

使用指定权限文件重签名

```xml
<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd"><plist version="1.0"><dict>	<key>com.apple.backboardd.debugapplications</key>	<true/>	<key>com.apple.backboardd.launchapplications</key>	<true/>	<key>com.apple.frontboard.debugapplications</key>	<true/>	<key>com.apple.frontboard.launchapplications</key>	<true/>	<key>com.apple.private.cs.debugger</key>	<true/>	<key>com.apple.private.logging.diagnostic</key>	<true/>	<key>com.apple.private.memorystatus</key>	<true/>	<key>com.apple.springboard.debugapplications</key>	<true/>	<key>get-task-allow</key>	<true/>	<key>platform-application</key>	<true/>	<key>run-unsigned-code</key>	<true/>	<key>task_for_pid-allow</key>	<true/></dict></plist>
```

```shell
codesign -s - --entitlements ent.plist -f debugserver
```

## DebugServer + lldb 调试

将[处理好的debugServer ](https://github.com/wstclzy2010/iOS-debugserver)scp到手机的 `/usr/bin` 下，并添加可执行权限

```shell
# iOS设备上chmod +x /usr/bin/debugserver
```

这里之所以把处理过的 `debugserver` 存放在 iOS 的 `/usr/bin/` 下，而没有覆盖 `/Developer/usr/bin/` 下的原版 debugserver，一是因为原版 debugserver 是不可写的，无法覆盖；二是因为 `/usr/bin/` 下的命令无须输入全路径就可以执行，即在任意目录下运行 `debugserver` 都可启动处理过的 `debugserver`。

在 iOS 上启动想调试的应用后，开启 `debugServer`

```shell
debugserver *:1234 -a "SpringBoard"
```

```shell
# 示例输出Black:~ root# debugserver *:1234 -a "SpringBoard"debugserver-@(#)PROGRAM:LLDB  PROJECT:lldb-1200.2.12 for arm64.Attaching to process SpringBoard...Listening to port 1234 for a connection from *...
```

`1234` 是我们指定的端口号，可以自行替换，`-a` 后接我们想调试的进程名

切换至mac上，首先进入 `lldb` 

```bash
$ lldb(lldb)
```

链接到手机的 debugServer

```shell
process connect connect://iOSIP:1234
```

连接过程可能耗时较长，需要等待一会。

如果在 `connect` 时报错

```
error: failed to get reply to handshake packet
```

可以使用 `USB` 连接，切换到 `127.0.0.1` 然后使用 `iproxy` 转发

```
# 手机端debugserver 127.0.0.1:1234 -a "SpringBoard"# mac端1iproxy 1234 1234# mac端2lldbprocess connect connect://127.0.0.1:1234
```

## Theos越狱插件Tweak开发

参考: [iOS 越狱的Tweak开发](https://www.jianshu.com/p/a5435650e828)

### 1. Theos 安装

```
git clone https://github.com/theos/theos.git /opt/theos
```

一般情况放在 `/opt/theos` 下，也可以更换到自定义路径。完成后建立环境变量

```bash
export THEOS=/opt/theos
```

为了后续使用方便，可以将环境变量写入到 `zshrc` 或 `bash_profile `等文件中，让全局终端都能生效。

### 2. 其他安装

安装 `ldid` 和 `dpkg`。

```bash
brew install ldidbrew install dpkg
```

`ldid` 是用来替代 `codesign` 进行签名的，让插件 `deb` 产物能顺利安装到手机上。

`dpkg` 是 `theos` 用来把工程打包成 `deb` 文件用的。

### 3. 创建新的插件

打开终端执行 `$THEOS/bin/nic.pl` :

```bash
NIC 2.0 - New Instance Creator------------------------------  [1.] iphone/activator_event  [2.] iphone/activator_listener  [3.] iphone/application_modern  [4.] iphone/application_swift  [5.] iphone/cydget  [6.] iphone/flipswitch_switch  [7.] iphone/framework  [8.] iphone/library  [9.] iphone/notification_center_widget  [10.] iphone/notification_center_widget-7up  [11.] iphone/preference_bundle_modern  [12.] iphone/theme  [13.] iphone/tool  [14.] iphone/tool_swift  [15.] iphone/tweak  [16.] iphone/tweak_with_simple_preferences  [17.] iphone/xpc_serviceChoose a Template (required): 
```

输入 15 选择 `iPhone/tweak` 。

```bash
Project Name (required):myTweak
```

输入插件名称。

```bash
Package Name [com.yourcompany.mytweak]:
```

输入包名或输入回车使用提供的默认包名。

```bash
Author/Maintainer Name [lindubo]:
```

输入作者名或输入回车使用默认。

```bash
[iphone/tweak] MobileSubstrate Bundle filter [com.apple.springboard]:
```

输入插件要作用的包名（进程名？），如  `com.apple.UIKit` 可以作用于所有App，如 `com.apple.webinspectord` 作用于`webinspectord` 的守护进程。

```bash
[iphone/tweak] List of applications to terminate upon installation (space-separated, '-' for none) [SpringBoard]:
```

输入插件安装完成后需要重新的进程名完成创建，创建完后目录结构为:

```bash
.└── mytweak    ├── Makefile    ├── Tweak.x    ├── control    └── myTweak.plist
```

其中 `control` 文件记录了插件相关的信息:

```bash
$ cat controlPackage: com.yourcompany.mytweakName: myTweakVersion: 0.0.1Architecture: iphoneos-armDescription: An awesome MobileSubstrate tweak!Maintainer: linduboAuthor: linduboSection: TweaksDepends: mobilesubstrate (>= 0.9.5000)
```

而 `plist` 文件中记录了前面输入的插件作用的目标包名:

```bash
{ Filter = { Bundles = ( "com.apple.webinspectord" ); }; }
```

`makefile` 用来编译工程:

```bash
$ cat MakefileTARGET := iphone:clang:latest:7.0INSTALL_TARGET_PROCESSES = SpringBoardinclude $(THEOS)/makefiles/common.mkTWEAK_NAME = myTweakmyTweak_FILES = Tweak.xmyTweak_CFLAGS = -fobjc-arcinclude $(THEOS_MAKE_PATH)/tweak.mk
```

里面引用了 `$THEOS` 环境变量，因此前面需要先设置好环境变量。

`tweak.xm` 是我们需要编译的文件：

```bash
$ cat Tweak.x/* How to Hook with LogosHooks are written with syntax similar to that of an Objective-C @implementation.You don't need to #include <substrate.h>, it will be done automatically, as willthe generation of a class list and an automatic constructor.%hook ClassName// Hooking a class method+ (id)sharedInstance {	return %orig;}// Hooking an instance method with an argument.- (void)messageName:(int)argument {	%log; // Write a message about this call, including its class, name and arguments, to the system log.	%orig; // Call through to the original function with its original arguments.	%orig(nil); // Call through to the original function with a custom argument.	// If you use %orig(), you MUST supply all arguments (except for self and _cmd, the automatically generated ones.)}// Hooking an instance method with no arguments.- (id)noArguments {	%log;	id awesome = %orig;	[awesome doSomethingElse];	return awesome;}// Always make sure you clean up after yourself; Not doing so could have grave consequences!%end*/
```

其中 `%hook` 、 `%orig` 、`%log` 都是 `theos` 对 `Cydia Substrate` 提供的函数的宏封装，介绍和示例可以参考[Cydia_Substrate](https://link.jianshu.com/?t=http://iphonedevwiki.net/index.php/Cydia_Substrate):

```
IMP MSHookMessage(Class class, SEL selector, IMP replacement, const char* prefix); > // prefix should be NULL. void MSHookMessageEx(Class class, SEL selector, IMP replacement, IMP *result); void MSHookFunction(void* function, void* replacement, void** p_original);
```

`Cydia Substrate` 还提供了 `MobileLoader` ，上面的这些钩子需要在运行时被加载就是由于  `MobileLoader` 会在适当的时机加载越狱设备上 `/Library/MobileSubstrate/DynamicLibraries/` 目录里的动态库，我们的 `tweak` 最终产物就是动态库。

```
...// The attribute forces this function to be called on load.__attribute__((constructor))static void initialize() {  NSLog(@"MyExt: Loaded");  MSHookFunction(CFShow, replaced_CFShow, &original_CFShow);}
```

另外由于 `__attribute__((constructor))` 的执行时机早于 `main` 函数，因此钩子得以在较早的时机执行。

`tweak` 文件的后缀名中，`.x` 表示支持 [logos](https://link.jianshu.com/?t=http://iphonedevwiki.net/index.php/Logos) 语法和 C语法；`.xm` 表示支持 [logos](https://link.jianshu.com/?t=http://iphonedevwiki.net/index.php/Logos) 语法和 C/C++ 语法。

### 编译与安装

编译deb包

```bash
make packages
```

完成后会多出 `.theos` 目录，和 `packages` 目录。动态库产物在 `./theos/_/Library/MobileSubstrate/DynamicLibraries/` 下，`deb` 产物在 `packages` 目录下。

```
$ tree -a -L 2.├── .theos│   ├── _│   ├── build_session│   ├── fakeroot│   ├── last_package│   ├── obj│   └── packages├── Makefile├── Tweak.x├── control├── myTweak.plist└── packages    └── com.yourcompany.mytweak_0.0.1-1+debug_iphoneos-arm.deb5 directories, 8 files
```

#### 使用iFile安装

1. 使用工具将 `deb` 文件拷贝到手机上，可以通过 `ssh` 或其他工具
2. 手机上使用 `iFile` 打开进行安装

#### ssh直接编译+安装

在 `Makefile` 最前面加上:

```bash
THEOS_DEVICE_IP=IP_Of_iPhone
```

然后执行 `make package install` 即可将产物直接安装到设备上。

如果需要指定端口，也可以再加上

```
THEOS_DEVICE_PORT=2222
```

#### 删除

安装的插件可以直接在 `Cydia` 中进行删除。

如果手机上安装了 `dpkg` (Debian Package)，可以在手机终端上执行

```
# 安装dpkg -i path/to/deb# 安装后重启SpringBoardkillall -HUP SpringBoard# 卸载，参数为插件的packageNamedpkg -r packageName
```

deb实际上是一个压缩包，如果使用其他压缩软件解压会丢失原有文件的权限信息，因此可以用 `dpkg-deb` 来解包

```bash
# 加压dpkg-deb -x abc.deb tmp #将abc.deb的程序文件解包到tmp文件夹dpkg-deb -e abc.deb tmp/DEBIAN #将abc.deb的安装控制/识别信息解包到DEBIAN文件夹# 打包# 假设将需要打包的文件放在tmp文件夹中，DEBIAN文件夹也要在放在这个文件夹中，然后输入命令：chmod -R 0755 tmp/DEBIAN #首先设置权限，如果没有包含脚本可以不需要设置权限dpkg-deb -b tmp 1.deb #打包成一个叫做1.deb的包
```

进入DEBIAN目录，可以看到有一个control文件，无后缀名，这个文件就是用来记录deb的安装信息。





### Frida

```
frida-ps -U
```

