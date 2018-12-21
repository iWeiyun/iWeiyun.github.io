---
title: Mac内核扩展开发
date: 2018-12-21 12:31:55
author: sniper
tags: mac,kernel,内核
---



因为项目需要，在Mac系统上要实现一个能监控所有的文件读写的应用，并且可以阻塞或者拒绝一些特定的文件操作。

在Window上可以通过文件系统过滤驱动来做，而且网上资料也挺多。
Mac平台上则需要通过内核扩展（KEXT）来做，相关的资料相对就少很多了，而且大部分是英文的，过程也中踩了一些坑。

就准备把流程在这篇文章里记录一下。



### 准备测试机
不管是Window还是Mac，估计Linux也一样，内核相关的开发是需要两台机器的。一台机器用于开发，一台机器专门用于测试编译出来的kext。

原因之一是方便，内核中如果出现空指针啊之类的错误的话，是会导致kernel panic，就只能重启系统了。普通的逻辑bug放到内核里也可能导致整个系统都不能正常运作，总之重启是家常便饭。这时两台机器分工能大大提高开发效率的。

原因之二是内核调试必须要使用两台机器。估计是因为内核断点下来之后，应用层的UI、调试器之类的程序也都暂停了。所以只能用另外的机器进行远程调试。



------



实际操作中通常是使用虚拟机，成本低，有快照支持，非常适合用来当测试用机。
虚拟化软件推荐直接使用VMware Fusion，最开始我使用的Virtual Box，想着是免费开源的，然而踩了几天坑之后，发现VBox对MacOS支持的并不好。

首先要创建安装系统用的iso：

```shell
hdiutil create -o /tmp/HighSierra -size 8G -layout SPUD -fs HFS+J -type SPARSE
hdiutil attach /tmp/HighSierra.sparseimage -noverify -mountpoint /Volumes/install_build
sudo /Applications/Install\ macOS\ High\ Sierra.app/Contents/Resources/createinstallmedia --volume /Volumes/install_build
hdiutil detach /Volumes/Install\ macOS\ High\ Sierra/
hdiutil convert /tmp/HighSierra.sparseimage -format UDTO -o /tmp/HighSierra.iso
```



第三行中的.app可以直接从App Store上下载，不同系统版本路径会稍有变化。
![系统安装文件下载](/images/mac-kernel/1545314322_42.png)
其他的命令是创建磁盘镜像，挂载，卸载，转换为iso格式，具体可以参考hdiutil的man文档。

------

然后就是新建虚拟机，把刚刚创建的iso装载到虚拟机光驱里，启动，安装就行了。没有什么需要特别设置的。
（VBox在安装过程中的第一次重启，不能引导正确的分区，需要进boot manager手动选择）

![网卡设置](/images/mac-kernel/1545314915_27.png)

值得一提的是网卡这边可以设成host-only，这样就可以用虚拟机的ip直接访问它。但是虚拟机不能访问外网，不过也没必要访问外网。

![文件共享](/images/mac-kernel/1545318719_59.png)

通过打开虚拟机Mac自带的文件共享来传输文件，VMware自带的文件拖放不太好用。

### 配置调试环境
首先要给虚拟机安装上内核调试套件Kernel Debug Kit，这里面包含了Debug版的内核和驱动，以及相应的符号文件，可以从苹果的[开发者中心](https://developer.apple.com/download/more/ "下载地址")下载。

![KDK下载](/images/mac-kernel/1545319704_63.png)

注意版本号要与虚拟机里装的MacOS完全一致！

KDK的安装路径是/Library/Developer/KDKs，其中有一个官方的readme文件，KDK的使用方法，Debug方法基本都包含了，是非常有用的资料。

------

在虚拟机里完成KDK的安装后，需要设置启动参数，VMware里就跟真机一样设置就好了：

```shell
sudo nvram boot-args="-v debug=0x146 kdp_match_name=en0 kcsuffix=development"
```

- -v：输出所有内核日志，启动时也会输出到黑屏幕
- debug：内核调试的关键设置，具体可以参考苹果官方文档[Building and Debugging Kernels](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/build/build.html "Building and Debugging Kernels")里的Debugging flags

![Debugging flags](/images/mac-kernel/1545321624_58.png)

- kcsuffix：指定使用的内核，这里因为只是debug我们开发的kext，而不是内核本身，使用development的内核就可以了，性能会比较好。
- kdp_match_name：指定断点时用哪个网卡进行远程调试

这里特别提一下kdp_match_name这个选项，网上的教程大部分都没有这个选项。但是我最开始用VBox来远程调试的时候，死活都连不上，卡了很久，最后在前面提到的官方readme里找到了这个选项才解决。因为当时设置了两个网卡，除了host-only，还有一个NAT连外网，系统默认用了NAT的网卡来调试，当然就没办法连上了。
（VBox对nvram的支持非常差，直接用nvram命令设置是无效的，要用命令：VBoxManage setextradata 虚拟机名 "VBoxInternal2/EfiBootArgs" "-v debug=0x146 kdp_match_name=en0 kcsuffix=development"）
（后面关闭SIP的设置没法保存也是同理，必须每次启动都操作一次）

------

接下来要做的是替换development内核，但在此之前要先关闭SIP（System Integrity Protection），因为内核是受SIP保护的，而且后面要加载未签名的KEXT也是需要在SIP关闭的情况下。

1. 重启，并按住Cmd+R进恢复模式
   （VBox因为没有没有启动前的BIOS画面？所以也只能通过boot manager进恢复模式）
2. 在恢复模式的终端里执行csrutil disable

关闭了SIP后，重启，替换一下开发内核：

```shell
sudo cp /Library/Developer/KDKs/KDK_10.14_18A391.kdk/System/Library/Kernels/kernel.development /System/Library/Kernels/kernel
sudo kextcache -invalidate /
sudo reboot
```

后两句是清除内核缓存并且重启。
到此调试环境就算是配置完成了。

### KEXT开发调试
先来测试下远程调试是否OK。
因为前面设置了0x04这个debug标记，可以通过按键触发NMI （non-maskable interrupt）来进入调试状态。因为要按的键太多了，设置个映射比较方便。
![按键映射](/images/mac-kernel/1545324557_74.png)

按完之后如果虚拟机整个卡住了，那就说明进入调试状态了。

在开发机上也装上KDK，因为需要符号和内核调试用的lldb脚本。之后运行lldb进行调试：

```shell
lldb /Library/Developer/KDKs/KDK_10.14_18A391.kdk/System/Library/Kernels/kernel.development
```

按提示导入脚本：
![导入脚本](/images/mac-kernel/1545324957_2.png)
也可以按提示设置一下自动导入，下次进来就不用再敲导入命令了。

用ip连接虚拟机进行调试：

```shell
kdp-remote 192.168.19.128
```

![kdp-remote](/images/mac-kernel/1545325071_45.png)
剩下的就跟lldb调试普通应用差不多了，例如bt命令打印堆栈等等，这里就不展开了，大家可以单独去学习。
![bt](/images/mac-kernel/1545325185_40.png)

kext执行在内核中，调试也是差不多的。当你的kext引发了panic时，会自动断下来，此时用lldb连上去就能看到堆栈了。
也可以在lldb中下断点，或使用int3主动断下来。

------

最后我们来实现最开始说的文件读写监控的功能。

Google了一阵之后，发现苹果在10.4引入的Kauth（Kernel Authorization）相关KPI（Kernel Programming Interface），正好符合我们的要求。

Kauth里的Vnode Scope，会回调所有的文件系统操作给KEXT中注册的回调函数，包括读写、执行、删除等，并且可以由KEXT决定是否拒绝访问。

具体细节可以参考[官方文档](https://developer.apple.com/library/archive/technotes/tn2127/_index.html "官方文档")，同时还提供了一个[示例代码](https://developer.apple.com/library/archive/samplecode/KauthORama/Introduction/Intro.html#//apple_ref/doc/uid/DTS10003633 "示例代码")，可以说是非常贴心了。虽然这些资料都有些年头了，但是完全没有过时。

接下来直接看代码：

```c
extern kern_return_t com_example_apple_samplecode_kext_KauthORama_start(kmod_info_t * ki, void * d);
extern kern_return_t com_example_apple_samplecode_kext_KauthORama_stop(kmod_info_t * ki, void * d);
```

这是入口函数，在kext加载或卸载时执行，可以在工程设置中指定：
![入口函数](/images/mac-kernel/1545326692_63.png)
然后是注册回调函数，以及回调的部分实现：

```c
static void InstallVnodeListener() {
    gListener = kauth_listen_scope(KAUTH_SCOPE_VNODE, VnodeScopeListener, NULL);
    if (gListener == NULL ) {
        RemoveVnodeListener();
    }
}

static int VnodeScopeListener(
    kauth_cred_t    credential,
    void *          idata,
    kauth_action_t  action,
    uintptr_t       arg0,
    uintptr_t       arg1,
    uintptr_t       arg2,
    uintptr_t       arg3    // type (int *), is errPtr, if denies, indicate the error to return to the client.
)
{

    int ret = KAUTH_RESULT_DEFER;

    atomic_fetch_add(&gActivationCount, 1);

    int err;
    vnode_t vp = (vnode_t)arg1;

    int tag = vnode_tag(vp);
    if (tag != VT_HFS && tag != VT_APFS) {
        atomic_fetch_sub(&gActivationCount, 1);
        return ret;
    }

    char * vpPath;
    err = createVnodePath(vp, &vpPath);
```

更具体的代码就先略过了，可以参考官方文档和示例。除此之外还有用sysctl控制kext的方法，这个示例代码里就有。以及跟用户层进行类socket通信的[KEXT Controls](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/NKEConceptual/control/control.html "KEXT Controls") KPI。

------

代码写好之后，先编译出来，然后要在plist里配置一下依赖库，用kextlibs命令来查询用到了那些库：
![kextlibs](/images/mac-kernel/1545327326_80.png)
填到plist的OSBundleLibraries项里：
![plist](/images/mac-kernel/1545327390_10.png)
重新编译一下就可以加载了，不加这些的话，在加载时就会出现找不到符号的错误。

------

在虚拟机上执行以下命令加载kext：

```shell
chown -R root:wheel KauthORama.kext; kextload KauthORama.kext
```

必须以root用户运行，我是先用了sudo -i进了root shell的，就不用每次都sudo了。chown的原因也是因为kextload的限制，要求kext文件的owner是root。

之后就可以通过log命令来查看kext里print出来的读写请求了：

```shell
log show --last=3m --predicate 'sender == "KauthORama"'
```

效果如下：

```shell
Filtering the log data using "sender == "KauthORama""
Skipping info and debug messages, pass --info and/or --debug to include.
Timestamp                       Thread     Type        Activity             PID    TTL  
2018-12-21 01:44:41.308911+0800 0x3722     Default     0x0                  0      0    kernel.development: (KauthORama) action=LIST_DIRECTORY|ACCESS, uid=501, vp=/Users/zouquan/Desktop/KauthTest, dvp=<null>, pid=271, pname=Finder
2018-12-21 01:44:41.308917+0800 0x3722     Default     0x0                  0      0    kernel.development: (KauthORama) action=SEARCH|ACCESS, uid=501, vp=/Users/zouquan/Desktop/KauthTest, dvp=<null>, pid=271, pname=Finder
2018-12-21 01:44:41.309565+0800 0x3706     Default     0x0                  0      0    kernel.development: (KauthORama) action=READ_ATTRIBUTES|READ_SECURITY, uid=501, vp=/Users/zouquan/Desktop/KauthTest, dvp=<null>, pid=65, pname=mds
2018-12-21 01:44:41.309676+0800 0x3706     Default     0x0                  0      0    kernel.development: (KauthORama) action=READ_ATTRIBUTES, uid=501, vp=/Users/zouquan/Desktop/KauthTest, dvp=<null>, pid=65, pname=mds
2018-12-21 01:44:41.309697+0800 0x3706     Default     0x0                  0      0    kernel.development: (KauthORama) action=LIST_DIRECTORY, uid=501, vp=/Users/zouquan/Desktop/KauthTest, dvp=<null>, pid=65, pname=mds
--------------------------------------------------------------------------------------------------------------------
Log      - Default:          5, Info:                0, Debug:             0, Error:          0, Fault:          0
Activity - Create:           0, Transition:          0, Actions:           0
```

当然也可以用控制台APP来查看，效果是一样的：
![控制台](/images/mac-kernel/1545328422_27.png)

