---
title: iOS逆向笔记
date: 2018-10-12 14:52:00
tags: [iOS]
categories: [iOS]
---

## iOS系统安全架构

1. 安全启动链
  	* 信任链：系统启动 -> boot rom -> LLB -> iBoot -> kernal
2. 系统软件授权
    * 固件 -> cpu -> itunes ==> 服务器验证
      * 低版本服务器返回验证许可，可通过保存shsh绕过验证
      * 高版本返回验证许可+随机串
3. 应用代码签名
  	* 所有可执行代码（动态库，动态资源）运行时验证签名
4. 运行时进程安全
    * 沙盒：/var/mobile/Containers/Data/Application/[GUID]
    * DEP
    * ASLR	
5. 数据加密保护
    * 硬件秘钥 + 密码秘钥 -> 类秘钥
    * 硬件秘钥 -> 文件系统秘钥
    * 类秘钥 + 文件系统秘钥 -> 文件秘钥 => **文件内容**


<!-- more -->


## 越狱
1. Cydia Substrate
    * MobileHooker
    * MobileLoader
    * safe mode
2. ssh连接ios设备
    * openssh

      * ssh root@ip
    * 通过usb连接：
      1. [usbmuxd](cgit.sukimashita.com/usbmuxd.git/snapshot/usbmuxd-1.0.8.tar.gz) -> tcpreplay.py -t 22:xx
      2. brew install libimobiledevice -> iproxy 2222 22

      		* ssh root@localhost -p xx
      		* 配置免密码登录越狱设备

```bash
ssh-keygen -t rsa -P ''
ssh-copy-id -i /Users/username/.ssh/id_rsa root@ip
或者安装sshpass自己设置密码:
brew install https://raw.githubusercontent.com/kadwanev/bigboybrew/master/Library/Formula/sshpass.rb
```

{{< admonition tip >}}
ssh key不匹配，删除.ssh/know_hosts对应可以即可
{{< /admonition >}}

3. 终端输入中文

```bash
cd /var/root
touch .inputrc
echo 'set convert-meta off' >> .inputrc
echo 'set meta-flag on' >> .inputrc
echo 'set output-meta on' >> .inputrc
echo 'set input-meta on' >> .inputrc
```
4. 插件
  1. ps：安装adv-cmds
  2. 绕过app签名：安装appsync
  3. 文件显示：[安装apple file conduit 2](https://yalujailbreak.net/apple-file-conduit-2-ios-11/)
  4. fileza

> 越狱工具 [next.tweakboxapp.com]()

## app文件结构和构建
1. 查看符号表：nm -nm
2. 依赖库、汇编等：otool -L / -tV
3. 从源代码到ipa的构建过程

## 数据存储
1. property list
  	* plutil -convert xml1 xx -o xx.plist
2. User Defaults
  	* sandbox -> Library -> Preferences -> xx.plist
3. Archive
4. Database
  	* sqlite
5. keyChain
  	* [Keychain Dumper](https://github.com/ptoomey3/Keychain-Dumper)

## Runtime

## Hook
1. Method Swizzle

	针对objective-C，利用runtime，替换Dispatch table中SEL和IMP的对应关系。
    * [IOS黑魔法－Method Swizzling](https://www.google.co.jp/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=2ahUKEwifib7x-ZneAhUJvrwKHQRNCBsQFjAAegQICRAB&url=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fff19c04b34d0&usg=AOvVaw0Gl7o95OB0WBZ73DrZ4FP8)
    * [Objective-C Method Swizzling 的最佳实践](http://blog.leichunfeng.com/blog/2015/06/14/objective-c-method-swizzling-best-practice/)
    * [如何在 Swift 中高效地使用 Method Swizzling](http://swift.gg/2016/03/29/effective-method-swizzling-with-swift/)
2. fishhook

	rebind_symbols：遍历查找懒加载表和非懒加载表的符号并替换地址。

	* [https://github.com/facebook/fishhook](https://github.com/facebook/fishhook)
  
3. Cydia Substrate

    * MSHookMessageEx -> objC
      * replace Objective-C message implementations
    * MSHookFunction -> c
      * hijack native code to your own implementation
    * [http://iphonedevwiki.net/index.php/Cydia_Substrate](http://iphonedevwiki.net/index.php/Cydia_Substrate)

## ARM汇编
push {r0}  <==>  str r0,[sp,#-4]!

pop {r0}   <==>  ldr r0,[sp],#4 

> 汇编使用intel模式：[https://github.com/vadimcn/vscode-lldb/issues/87](https://github.com/vadimcn/vscode-lldb/issues/87)
>
> 模拟器使用x86架构，32位入栈。64位rdi、rsi、rdx、rcx、r8、r9，多余参数入栈。
>
> 真机arm架构，32位r0-3，64位w0-8。

## Mach-O
1. 文件架构

  `lipo -info whatsApp`
  
  ` file whatsApp`

  * Header

    `otool -hv whatsApp`

    MachOView
    * 魔数
      * 通用格式：0xcafebabe
      * armv7：0xfeedface
      * arm64：0xfeedfacf
    * 文件类型
      * OBJECT：1
      * EXECUTE：2
      * DYLIB：6
  * Load commands -> n * Segment

    `otool -lv whatsApp`
    * (lldb) image list -e -f
    * Load commands -> LC_SEGMENT(_TEXT) -> VM address
    * 加载地址 + 段虚拟地址 = 文件加载地址
  * Data -> n * Section
  * Mach-O动态链接（延迟加载）

    * 
  * Code Signature

    * codesign -dvvvv whatsApp

## App签名
1. 查看加密标识：
  	* otool -l whatsApp | grep crypt
2. 查看电脑中证书
  	* security find-identity -v -p codesigning
3. 获取描述文件（embedded.mobileprovision）
    * 苹果官网获取
    * 新建工程真机安装后在包中获取
    * 查看：security cms -D -i embedded.mobileprovision
    * 拷贝至要签名文件中
4. 生成授权文件
    * security cms -D -i embedded.mobileprovision > entitlements_full.plist
    * /usr/libexec/PlistBuddy -x -c 'Print:Entitlements'  entitlements_full.plist > entitlements.plist
5. 重签名
  	* codesign -fs "iPhone Developer: xxx（2中证书名）" --no-strict --entitlements=xxx（4中授权文件） Payload/xxx.app
      * 如果有extension，可删可重签名
      * 如果新加动态库，需要签名
6. 压缩为ipa
  	* zip -qr target.ipa Payload

## 动态库
1. clang编译objec并签名，放入Library->load dylibrary->DynamicLibaries中
2. 导出和隐藏符号
    * static
    * export_list
    * -fvisibility=hidden
3. 函数调用顺序
    1. \_\_attribute__((constructor))
    2. main
    3. \_\_attribute__((destructor))
4. 基于c++动态库
5. 基于oc动态库
6. 注入可执行文件
    * LC_LOAD_DYLIB
    * 工具: [insert_dylib](https://github.com/Tyilo/insert_dylib)
    * 查看：otool -L

## 应用砸壳
1. 查看加壳
  * otool -l xx(mach-o) | grep cryptid
    * 从itools导出ipa
    * 利用scp导出执行文件（打开app，ps找到进程）
2. 砸壳工具
  * [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)
    * 查找Document路径:
      1. cycript -p WeChat
      2. NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,NSUserDomainMask,YES)[0] 
  * [AloneMonkey/dumpdecrypted](https://github.com/AloneMonkey/dumpdecrypted)
  * [Clutch](https://github.com/KJCracks/Clutch) 
  * [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump)

## class-dump
1. [https://github.com/nygard/class-dump](https://github.com/nygard/class-dump)

    官方工具修正后编译： 

```
// CDObjectiveC2Processor.m  224h
objc2Class.data       = value & ~3;   //error: value & ~7;
```
2. `./class-dump mach-o_file -H -o ./Headers`

{{< admonition tip >}}
通过 lipo mach-o_file -thin armv7 -output mach-o_file.armv7 瘦身提取砸壳架构
{{< /admonition >}}

## Reveal
1. cydia中添加BigBoss源安装Reveal2Loader，开启reveal使用
2. 高版本[reveal](https://www.52pojie.cn/thread-762028-1-1.html)需要将RevealServer（help->show reveal library in finder->ios library）重命名为libReveal.dylib，并和libReveal.plist一同放入ios的Library->MobileSubstrate->DynamicLibraries中

  	plist文件内容：
    ```
    { Filter = { Bundles = ( "bundleID" ); }; }
    ```
3. [MonkeyDev](https://github.com/AloneMonkey/MonkeyDev)

## cycript
1. [cycript](http://www.cycript.org/)

## 静态分析
1. 工具
    * hopper
    * IDA
2. arm汇编，结构体！

## 动态调试
1. 处理debugserver
    * 下载/Developer/usr/bin/debugserver
    * 授权文件entitlements.plist内容：
      * com.apple.springboard.debugapplications 
      * run-unsigned-code
      * get-task-allow
      * task_for_pid-allow
    * 签名
    * 上传至/usr/bin/debugserver
2. lldb + debugserver
  * 连接
    1. 映射端口： python tcpreply.py -t 22:2222 1234:1234
    2. 手机：debugserver *:1234 -a WhatsApp
    3. pc： llvm --> process connect connect://localhost:1234

  * 调试指令
    * image list -o -f
    * b -[xx xxx]
    * br s -a 0xaaaaaa
    * po $r0
    * p/x $r0
    * x/s $r1
    * register read 
    * memory read
    * br list /  br com add 1.1  : DONE
3. xcode调试
  1. 新建与app同名工程
  2. [运行脚本](https://github.com/AloneMonkey/iOSREBook/blob/master/chapter-5/5.2%20%E5%8A%A8%E6%80%81%E8%B0%83%E8%AF%95/Snapchat/pack.sh)
  3. run
  4. 安装[chisel](https://github.com/facebook/chisel)

  or

  使用 **MokeyDev**

 {{< admonition tip >}}
  **定位button的action事件**

   * 传统:
     1. [btn allTargets]    //xxx
     2. [btn allControlEvents]    //yyy
     3. [btn actionsForTarget:xxx forControlEvent:yyy]
  * 使用chisel:
  	1. pviews
  	2. pactions btn
    {{< /admonition >}}

## Theos
1. tweak
  1. 执行$THEOS/bin/nic.pl创建工程
  2. 编写hook代码

    ```
    %hook ViewController
    
    -(void)clickme:(id)sender{
    	NSLog(@"Click me hooked !!!");
    	%log;
    	%orig;  //执行原方法
    }
    
    %end
    ```
  3. 编译
    * make
    * make package
    * make install
2. iOSOpenDev（13年没更新了）
3. [MonkeyDev](https://github.com/AloneMonkey/MonkeyDev)

> **查看Log输出**
> 
> 安装libimobiledevice工具(需要[usbmuxd](https://github.com/flutter/flutter/issues/22595)):
> `brew install --HEAD libimobiledevice`
> 
> 使用如下命令查看输出Log:
> `idevicesyslog | grep 'xxx'`
> 
> 或者使用自带的 console.app 程序查看。

## Oplayer lite 去广告
1. 使用frida-ios-dump砸壳
2. 用class-dump导出头文件
3. 用reveal定位广告view
4. 用xcode lldb调试找到广告的addSubview:调用位置
5. 使用hooper静态分析逻辑
6. 用Theos写插件改变逻辑去广告

## SnapChat 消息

{{< admonition tip >}}
* 使用cycript或者reveal找到view对应的viewcontroller
* 通过theos的logify.pl hook一个类所有方法找到具体调用
* 通过lldb打印具体调用的执行堆栈 -- bt
* tweak找不到类可以通过声明@class xxx解决
{{< /admonition >}}

## 迁移到非越狱设备

## Frida

## 数据加密
* 字符串加密
* 数据存储加密
* 网络数据加密

## 反调试与反注入
* 反调试
  * ptrace

    ```c
    ptrace(PT_DENY_ATTACH, 0, 0, 0);
    ```

    ```c
    void *handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);
       ptrace_ptr_t ptrace_ptr = (ptrace_ptr_t)dlsym(handle, "ptrace");
       ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
    ```

  * sysctl

    使用sysctl获取当前进程，标志位p_flag对比P_TRACED

    ```c
    BOOL isDebuggerPresent(){
        int name[4];
        
        struct kinfo_proc info;
        size_t info_size = sizeof(info);
        
        info.kp_proc.p_flag = 0;
        
        name[0] = CTL_KERN;
        name[1] = KERN_PROC;
        name[2] = KERN_PROC_PID;
        name[3] = getpid();
        
        if(sysctl(name, 4, &info, &info_size, NULL, 0) == -1){
            NSLog(@"sysctl error...");
            return NO;
        }
        
        return ((info.kp_proc.p_flag & P_TRACED) != 0);
    }
    ```

  * syscall

    ```c
    syscall(26, 31 , 0, 0, 0);
    ```
* 反反调试

  通过[fish hook](https://github.com/AloneMonkey/iOSREBook/blob/master/chapter-8/8.3%20%E5%8A%A8%E6%80%81%E4%BF%9D%E6%8A%A4/AntiAntiDebug/AntiAntiDebug.m)

  * hook ptrace
  * hook dlsym
  * hook syscall
  * hook sysctl		

* 反注入
  * 编译加入标志位

    ```
    -Wl,-sectcreate,__RESTRICT,__restrict,/dev/null
    ```

    原理，增加__RESTRICT这个section

    dylib的注入一般是通过DYLD_INSERT_LIBRARIES这个环境变量来实现的，在[dyld源码](http://www.opensource.apple.com/source/dyld/dyld-210.2.3/src/dyld.cpp)中：

    ```
    case restrictedBySegment:
            dyld::log("main executable (%s) has __RESTRICT/__restrict section\n", sExecPath);
            break;
    ```

    故存在该分段即可忽略DYLD_开头的环境变量，从而防止dylib注入。
  * 检测加载动态库路径中是否有DynamicLibraries（测试没用）

    ```
    BOOL isInjected0(){
        int count = _dyld_image_count();
        if(count > 0){
            for(int i=0; i<count; i++){
                const char* dyld = _dyld_get_image_name(i);
                if(strstr(dyld, "DynamicLibraries")){
                    return YES;
                }
            }
            return NO;
        }
        return NO;
    }
    ```

  * 检测是否存在环境变量DYLD_INSERT_LIBRARIES（测试有用）

    ```
    BOOL isInjected1(){
        char* env = getenv("DYLD_INSERT_LIBRARIES");
        if(env)
            return YES;
        return NO;
    }
    ```
* 反反注入

  修改mach-o文件中__RESTRICT字段中restrict字符串为其他字符串

## 混淆
* 类名和方法名混淆
  * 宏定义替换
    * ios-class-guard
  * 直接替换MachO文件

    1. 修改 __objc_classname
    2. 修改 __objc_methname
* OLLVM
  ​	