---
title: Windows环境下appium的server，client端连接配置
tags: [appium,mobile testing]
keywords: 'appium,mobile testing'
categories: []
abbrlink: OvVxU2q7eB
date: 2021-10-14 23:11:10
description:
---

Appium是一个开源、跨平台的测试框架，可以用来测试原生及混合的移动端应用。Appium支持IOS、Android及FirefoxOS平台。**Appium使用WebDriver的json wire协议，来驱动Apple系统的UIAutomation库、Android系统的UIAutomator框架**。Appium对IOS系统的支持得益于Dan Cuellar’s对于IOS自动化的研究。Appium也集成了Selendroid，来支持老android版本。

Appium支持Selenium WebDriver支持的所有语言，如java、Object-C、JavaScript、Php、Python、Ruby、C#、Clojure，或者Perl语言，更可以使用Selenium WebDriver的Api。Appium支持任何一种测试框架。如果只使用Apple的UIAutomation，我们只能用javascript来编写测试用例，而且只能用Instruction来运行测试用例。同样，如果只使用Google的UIAutomation，我们就只能用java来编写测试用例。Appium实现了真正的跨平台自动化测试。

appium选择了client-server的设计模式。只要client能够发送http请求给server，那么的话client用什么语言来实现都是可以的，这就是appium及webdriver如何做到支持多语言的；

## 服务端配置

1. 在appium-desktop中下载对应的Windows安装包(**zip格式版本**)，下载地址： <https://github.com/appium/appium-desktop/releases>

2. 打开命令行切换到appium-desktop安装路径，确认是否配置正确：

```
npm install appium-doctor -g
> appium-doctor --ios  检测ios设备配置信息
> appium-doctor --android 检测android设备配置信息
```

Appium的客户端和服务端之间进行通信都必须在一个Session的上下文中进行。客户端在发起通信的时候首先会发送一个叫作“Desired Capabilities”的JSON对象给服务器。服务器收到该数据后，会创建一个session并将session的ID返回到客户端。之后客户端可以用该session的ID发送后续的命令。

Android（版本>4.3）：UIAutomator，Android 4.3之后系统自带的UI自动化测试工具
iOS：UIAutomation，iOS系统自带的UI自动化测试工具。

电脑（client）上运行自动化测试脚本，调用的是webdriver的接口，appium server接收到我们client上发送过来的命令后他会将这些命令转换为UIautomator认识的命令，然后由UIautomator来在设备上执行自动化

Appium Server的图形界面。 可以设置选项，启动/停止服务器，查看日志等...也不需要使用Node 的NPM来安装Appium，因为Node运行时与Appium Desktop捆绑在一起。


3. 安装安卓SDK（可以对手机进行连接操作，包括获取id，设备名，包名等）

　1、下载安卓SDK安装包，地址：<http://tools.android-studio.org/index.php/sdk>

　2、安装完成或者解压完成后，进入目录中打开：SDK Manager.exe

　3、我们使用真机测试，不需要安装模拟器镜像，暂时只需要安装Tools和Platform-tools，以后有需要可以继续安装

4. 获取手机的设备信息

4.1. 打开命令提示符，定位到**platform-tools**的目录下，输入：`adb devices` 命令查看连接的手机。

```
E:\AndroidSDK\platform-tools>adb devices
* daemon not running; starting now at tcp:5037
* daemon started successfully
List of devices attached
688190fa0705    device
```

4.2. 打开小米手机在设置——我的设备——全部参数中，记下“Android版本”的版本号。
4.3.  清空后台的所有程序，打开计算器，然后打开命令提示符，定位到adb所在的目录路径
，输入命令：

```
adb shell
dumpsys window | grep "mCurrentFocus"
```

记下当前激活的窗口： com.miui.calculator和com.miui.calculator.cal.CalculatorActivity
![20200903082117-2020-09-03](https://raw.githubusercontent.com/alterhu2020/StorageHub/master/img/20200903082117-2020-09-03.png)

**该步骤操作可通过sdk tool命令查找到对应的package Name和main activity，参考文档： https://blog.csdn.net/xiaosongbk/article/details/82903148**

4.4 启动appium-desktop，~~点击右上角的放大镜，在弹出的页面中输入前面3步所获取到的值~~，然后点击Start Session即可连接，如图：
最新版本的Appium需要单独下载额外的Inspector包： <https://github.com/appium/appium-inspector/tree/main>
![20200903082236-2020-09-03](https://raw.githubusercontent.com/alterhu2020/StorageHub/master/img/20200903082236-2020-09-03.png)

打开的新的session窗口的`Appium Server` 设置如下：
1. Remote Host: 127.0.0.1
2. Remote Port: 4723
3. Remote Path: /wd/hub

不用勾选“SSL”即可

打开新的session窗口后在`Desired Capabilities`中输入如下配置参数：

```
{
  "deviceName": "688190fa0705",
  "platformVersion": "10 QKQ1.19104.001",
  "platformName": "Android",
  "automationName": "UiAutomator2",
  "appPackage": "com.miui.calculator",
  "appActivity": "com.miui.calculator.cal.CalculatorActivity"
}
```

点击“**Start Session**" 进行调试运行。

## 客户端配置

比如对应的python的客户端包：

```
pip install appium-python-client
```

编写客户端代码（python），点击运行即可驱动真机进行计算

```
from appium import webdriver # 导入appium包
import time

# 连接手机app初始化的一些信息
desc = {}
desc['deviceName']='PNMxxxxxxxxxUQK'# 手机设备名称，adb devices
desc['platformVersion']='5.0.2'# 手机版本，在手机中：设置--关于手机
desc['platformName']='Android' # 手机类型，ios或android
desc['automationName']='UiAutomator2' # 手机类型，ios或android
desc['appPackage']='com.miui.calculator'#包名
desc['appActivity']='com.miui.calculator.cal.CalculatorActivity'#启动入口

driver=webdriver.Remote('http://127.0.0.1:4723/wd/hub',desc)#访问服务接口，并启动手机app

time.sleep(20)
driver.find_element_by_id('com.miui.calculator:id/btn_6_s').click()
time.sleep(2)
driver.find_element_by_id('com.miui.calculator:id/btn_plus_s').click()
driver.find_element_by_id('com.miui.calculator:id/btn_2_s').click()
driver.find_element_by_id('com.miui.calculator:id/btn_equal_s').click()

driver.quit()
```

## 真机或者模拟器配置

1. 打开USB调试

　设置——我的设备——全部参数——对着“MIUI版本”那一项点7次，会提示开发者选项打开。然后回到 设置——更多设置——开发者选项，将“**开启开发者选项**”、“**USB调试**”的开关设置为开启状态。
![20200903081628-2020-09-03](https://raw.githubusercontent.com/alterhu2020/StorageHub/master/img/20200903081628-2020-09-03.png)

2. 在Appium Inspector中点击“Start Sesssion"后记得点击允许安装对应的**appium-settings.apk**和 **appium-uiautomator2-server-v4.11.0.apk**包到真是的机器上，或者也可以自己进行安装，安装包位置位于appium的安装目录下面：

`E:\\WindowsApps\\Appium-windows-1.18.0-1\\resources\\app\\node_modules\\appium\\node_modules\\io.appium.settings\\apks\\settings_apk-debug.apk`

## 错误问题

1. An unknown server-side error occurred while processing the command. Original error: Neither ANDROID_HOME nor ANDROID_SDK_ROOT environment variable was exported. Read <https://developer.android.com/studio/command-line/variables> for more details

![20200903081012-2020-09-03](https://raw.githubusercontent.com/alterhu2020/StorageHub/master/img/20200903081012-2020-09-03.png)

解决方法： 按照提示在环境变量里面添加： `ANDROID_HOME`

2. An unknown server-side error occurred while processing the command. Original error: pkg: /data/local/tmp/appium_cache/728413456b8xxxxxxxxxxxxd6bd577ca6418.apk Failure [INSTALL_FAILED_USER_RESTRICTED]

解决方法： 请在开发者选项中打开USB安装、USB调试（安全设置）的开关。开关打开后，会安装一个**appium setting**的APP，安装完即可进行连接。

3. An unknown server-side error occurred while processing the command. Original error: Error executing adbExec. Original error: 'Command 'E:\\AndroidSDK\\platform-tools\\adb.exe -P 5037 -s 688190fa0705 install -g E:\\WindowsApps\\Appium-windows-1.18.0-1\\resources\\app\\node_modules\\appium\\node_modules\\io.appium.settings\\apks\\settings_apk-debug.apk' exited with code 1'; Stderr: 'adb: failed to install E:\WindowsApps\Appium-windows-1.18.0-1\resources\app\node_modules\appium\node_modules\io.appium.settings\apks\settings_apk-debug.apk: Failure [INSTALL_FAILED_USER_RESTRICTED: Install canceled by user]'; Code: '1'

解决方法： 打开Appium Inspector中Start Session后会在你的手机设备上安装一个appium-settings.apk的安装包，很多手机会阻止安装，你需要打开你的手机，在弹出的提示里允许进行操作安装apk。

4. An unknown server-side error occurred while processing the command. Original error: Error executing adbExec. Original error: 'Command 'E:\\AndroidSDK\\platform-tools\\adb.exe -P 5037 -s 688190fa0705 install -r --no-incremental E:\\WindowsApps\\Appium-windows-1.18.0-1\\resources\\app\\node_modules\\appium\\node_modules\\appium-uiautomator2-server\\apks\\appium-uiautomator2-server-v4.11.0.apk' exited with code 1'; Stderr: 'adb: failed to install E:\WindowsApps\Appium-windows-1.18.0-1\resources\app\node_modules\appium\node_modules\appium-uiautomator2-server\apks\appium-uiautomator2-server-v4.11.0.apk: Failure [INSTALL_FAILED_USER_RESTRICTED: Install canceled by user]'; Code: '1'

解决方法： 同上步骤，你需要在你的手机设备上允许安装一个操作安卓设备的**uiautomator2**，名称为： appium-uiautomator2-server-v4.11.0.apk
