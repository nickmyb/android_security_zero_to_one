脱壳和抓包基础
=============

## 0. 致谢

以下内容全部来源于 @**r0ysue**

![肉丝的星球-大家一起来玩](../1_environment_preparation/static/r0ysue.jpg)

[相关内容全部上传百度网盘 提取码：ljt1](https://pan.baidu.com/s/1__B7dnZ2xKBdXTNE_sklZg)

## 1. 走进加固的世界

主要还是合规需求(等级保护),另外小部分业务需求

- 相关链接
    - [中国网络安全等级保护网](http://www.djbh.net/webdev/web/HomeWebAction.do?p=init)
    - [威胁猎人 | 2018年上半年短视频行业黑灰产研究报告](https://bbs.pediy.com/thread-246419.htm)
    - [永安在线](https://bbs.pediy.com/user-807527.htm)
    - 百度盘 - 加固技术.png - 链接极为丰富

## 2. 一二代壳及其脱壳机的根本原理

- 相关链接
    - [来自高纬的对抗：定制ART解释器脱所有一二代壳](https://mp.weixin.qq.com/s/3tjY_03aLeluwXZGgl3ftw)
    - [一键加固](https://github.com/guanchao/apk_auto_enforce)
    - [在线源码](http://androidxref.com/)
    - [FART：ART环境下基于主动调用的自动化脱壳方案](https://bbs.pediy.com/thread-252630.htm)
    - [Android中实现「类方法指令抽取方式」加固方案原理解析](http://www.520monkey.com/archives/1118)
    - [拨云见日：安卓APP脱壳的本质以及如何快速发现ART下的脱壳点](https://bbs.pediy.com/thread-254555.htm)

### 2.1 一代壳的核心原理

双亲委派: 一个类加载器需要加载类，那么首先它会把这个类请求委派给父类加载器去完成，每一层都是如此。一直递归到顶层，当父加载器无法完成这个请求时，子类才会尝试去加载

- BaseDexClassLoader
    - PathClassLoader: Activity & Services
    - DexClassLoader: dex
    - InMemoryDexClassLoader: >8.0,从内存的字节流中加载

- 脱壳关键在于类加载的流程需要十分熟悉(虽然我一脸茫然),art解释器的开发能力决定脱壳能力

- 一代壳
    - 主要是对源程序的dex进行加解密,解密dex后使用DexClassLoader加载dex,PathClassLoader加载dex中的Activity之类Android组件: 脱壳关键点在于解密后的dex最后调用ClassLoader加载,所以只要关心ClassLoader即可
    - 通用方案：dex(关心dex的起始地址和长度)打开和优化的流程、及产出odex、dex2oat编译流程中的关键函数、或生成的oat文件(oat中也有完整的dex)

### 2.2 二代壳的核心原理

- 二代壳
    - 解析原始dex及其中所有方法的结构体信息,通过hook系统函数dexFindClass获取结构体信息,将需要抽取的方法的结构体置空,在类需要使用前将指令还原,更变态的还有对函数使用前后进行解加密
    - 方案：关注被抽取函数的执行流程是关键，核心是定位被抽取函数的真正恢复时机！在解释器插桩，在每条指令真正执行前设置回调

## 3. 热门脱壳机场景横评与实战

- 相关链接
    - [GDA](https://github.com/charles2gan/GDA-android-reversing-Tool)

### 3.1 三代壳的核心原理((dex)vmp & Dex2C)

- 三代壳(新手直接放弃就好了)
    - VMP: 定位解释器是关键，找到映射关系即可恢复
    - Dex2C: 基础是编译原理，进行等价语义转换，几乎无法彻底还原
    - 区别: VMP调用的JNI函数一直是同一个, Dex2C调用各种JNI
    - 方案：关注JNI接口的调用流程，是分析被VMP和Dex2C保护的函数的关键！也就是trace分析内部大致的逻辑即可！

### 3.2 热门脱壳机

#### 3.2.1 DEXDump暴力搜刮内存脱壳机

- [FRIDA-DEXDump](https://github.com/hluwa/FRIDA-DEXDump/tree/master/frida_dexdump)

搜索dex的特征文件头找到起始地址和长度

`unable to access process with pid 5233 due to system restrictions; try "sudo sysctl kernel.yama.ptrace_scope=0", or run Frida as root`,可以考虑认为是frida反调试,并可以`cat /proc/4848/status`查看tracepid是否更是双进程保护

```
grep -ril "MainActivity" *: 搜索binary时可能不够全面,因为grep在binary碰到不可见字符会中断退出,最好像FART一样提供函数列表来搜索
```

#### 3.2.2 (Frida_)FART轻松好用的被动调用脱壳机

- [hanbingle](https://bbs.pediy.com/user-632473.htm)
- [来自高纬的对抗：定制ART解释器脱所有一二代壳](https://mp.weixin.qq.com/s/3tjY_03aLeluwXZGgl3ftw)
- [FART源码解析](https://www.anquanke.com/post/id/201896)
- [hanbingle脱壳机实现](https://bbs.pediy.com/thread-254555.htm)
- [FART](https://github.com/hanbinglengyue/FART)
- [frida版的fart的介绍以及使用说明](https://bbs.pediy.com/thread-260045.htm)
    - 需要过frida的反调试,过不了考虑直接上FART/Youpk
    - spawn
        1. 下载并解压frida_fart.zip
        2. frida_fart/lib/fart* push到 /data/local/tmp
        3. /data/local/tmp/fart* 复制到 /data/app 并且 chmod 777 fart*
        4. 安装apk并给予sd卡读写权限
        5. 手机端以root启动frida-server 然后以spawn方式启动app: frida -U -f com.test.package -l frida_fart_hook.js --no-pause
        6. 等待app进入主界面后调用fart()即可 等待出现`end dump CodeItem`即可 此时可以在sd卡下看到dump的dex和函数体bin文件
    - attach
        5. 手机端以root启动frida-server 然后以spawn方式启动app: frida -U com.test.package -l frida_fart_reflection.js --no-pause

#### 3.2.3 FART/Youpk主动调用函数修复脱壳机

#### 3.2.3.1 FART

- [FART使用教程](https://mp.weixin.qq.com/s/3tjY_03aLeluwXZGgl3ftw)
- 如果使用免sdcard授权版本拖壳结果在`/data/data/com.test.package`
- 最后真正需要使用的只有FART拖壳结果中的 `xxx_ins_yyy.bin` 相关的xxx开头的几个文件,搜方法体在txt中搜,jadx打开xxx_dexfile_execute.dex,需要pull到本地的话需要先cp到sdcard
- dex中方法体如果为空需要修复
    - python需要用2.7
    - xxx_dexfile_execute.dex用来反编译查看源码,xxx_dexfile.dex用来修复
    - `python ~/Codes/github/others/FART/fart.py -d 3925368_dexfile.dex -i 3925368_ins_5923.bin >> repired.txt`
    - 但修复后是smali,并没有回填到dex,如果需要回填使用下面的修复脚本
    - [修复通过FART dump下来的dex](https://github.com/gagalin95/FART_repairdex)

#### 3.2.3.2 Youpk

- [Youpk使用教程](https://bbs.pediy.com/thread-259854.htm)
- Youpk直接回填相比之下更方便一点点

## 4. 抓包核心原理和配置

### 4.1 配置纯净抓包环境

1. 解压下载的kali虚拟机(kali-linux-2019.4-vmware-amd64.zip)
2. 修改kali虚拟机相关配置
    + 内存
        - 8G(8192MB)
    + 处理器
        - 4核：2 * 2
    + 显示器
        - 取消勾选 加速3D图形
    + 选项
        - 虚拟机名称 - CAPTURE
    + 网络适配器
        - **桥接模式**
    + 编辑 - 虚拟网络编辑器(需要使用管理员模式打开vmware)
        - 已桥接至 - (选择后面与手机连接在相同路由器中的网卡)
3. 更新源并且安装一些常用工具/ssh/proxychains/vim/.../(发挥主观能动性)
    + [直接apt install就踩坑了kali一顿操作疯狂更新GUI没了](https://askubuntu.com/questions/74523/how-can-i-install-a-package-without-installing-some-dependencies)
    + apt download htop jnettop nethogs tree tmux
    + dpkg [--force-all] -i htop_3.0.3-2_amd64.deb jnettop_0.13.0-1.1_amd64.deb nethogs_0.8.5-2+b1_amd64.deb tree_1.8.0-1+b1_amd64.deb tmux_3.1c-1_amd64.deb
4. sailfish刷机/Magisk/wifiadb/Postern(aosp_810r1_platform_tools.7z + sailfish-opm1.171019.011-factory-56d15350.zip) - 正确设置时间
5. 先: node(newest lts) + 后: pyenv(git安装)
6. [charles](https://www.charlesproxy.com/latest-release/download.do)

### 4.2 降维打击 [OSI七层模型](https://zh.wikipedia.org/wiki/OSI%E6%A8%A1%E5%9E%8B)

- [防止应用被抓包](http://www.520monkey.com/archives/1263)
- [ARM设备武器化指南·破·Kali.Nethunter.2020a.上手实操](https://www.anquanke.com/post/id/205455)
- [根据网卡判断VPN](https://www.remix.asia/blog/remix/2016/01/android.html)
- [实用FRIDA进阶：内存漫游、hook anywhere、抓包](https://www.anquanke.com/post/id/197657)

```
socks5 192.168.50.136 10808
会话层 网络层/路由层 传输层

socks5: session layer
http: presentation layer
```

VPN: 存在于网络层，会增加一项虚拟网卡(系统路由)，路由表全部都走这个网卡，通过会话层的SOCKS代理实现(连接感觉更为准确)网络层的VPN

```
255|sailfish:/ # ip route show table 0
default dev tun0  table tun0  proto static  scope link

traceroute
```

- Charles&Postern抓包配置
    1. charles配置: Proxy - Proxy Settings - Enable SOCKS proxy
    2. Postern配置: 配置代理 - 删除默认配置 - 添加代理服务器 - 服务器名称(任意) - 服务器地址(CAPTURE虚拟机IP) - 服务器端口(8889) - 代理类型(SOCKS5)
    3. Postern配置: 配置规则 - 删除默认配置 - 添加规则 - 匹配类型(匹配所有地址) - 动作(通过代理连接) - 代理/代理组(CAPTURE)
    4. charles配置: Allow
    5. charles配置(开启https抓包): Proxy - SSL Proxying Settings - Include - Add - Host:* Port:*
    6. charles配置: WIFI中配置http代理(8888) - 打开chls.pro/ssl安装证书 - 关闭http代理
    7. 手机配置: (复制charles证书到根证书) - su - mount -o remount,rw / - cd /data/misc/user/0/cacerts-added/ - cp -a * /etc/security/cacerts/ - mount -o remount,ro /
    8. 有些手机remount需要以下操作(adb root - adb disable-verity - adb reboot - adb root - adb remount) nougat需要 - mount -o rw,remount / - cp -a * /system/security/cacerts/
    9. 客户端验证服务器部分结束

## 5. HTTPS.C/S校验核心原理

- 客户端和服务器https通信
    1. 客户端通过URL请求服务器443端口
    2. 服务端返回服务端公钥服务器证书到客户端
    3. 客户端验证URL和服务端公钥服务器证书是否匹配(客户端验证服务器)
    4. 客户端使用服务端公钥对客户端公钥加密
    5. 服务端用服务端私钥解密出客户端公钥客户端证书(服务器验证客户端)
    6. 服务端用客户端公钥加密sessionkey
    7. 客户端使用客户端私钥解密出sessionkey
    8. 客户端服务端使用sessionkey对称加密传输http内容

### 5.1 服务器校验客户端实例(soul)

完整项目在 5_hook_key_store_load_and_ssl.js

需要找到证书文件及密码

```
// 针对不同的框架需要hook不同的方法来获取密码
// hook_key_store_load_and_ssl.js
// frida -H 192.168.2.10:8888 -f cn.soulapp.android -l agent/hook_key_store_load_and_ssl.js --no-pause

function hook_key_store_load() {
    var StringClass = Java.use("java.lang.String")
    var KeyStore = Java.use("java.security.KeyStore")
    KeyStore.load.overload('java.security.KeyStore$LoadStoreParameter').implementation = function (arg0) {
        console.log("KeyStore.load1:", arg0)
        // console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()))
        this.load(arg0)
    }
    KeyStore.load.overload('java.io.InputStream', '[C').implementation = function (arg0, arg1) {
        console.log("KeyStore.load2:", arg0, arg1 ? StringClass.$new(arg1) : null)
        // console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()))
        this.load(arg0, arg1)
    }

    console.log("hook_key_store_load...")
}

function hook_ssl() {
    var ClassName = "com.android.org.conscrypt.Platform"
    var Platform = Java.use(ClassName)
    var targetMethod = "checkServerTrusted"
    var len = Platform[targetMethod].overloads.length
    console.log(len)
    for(var i = 0; i < len; ++i) {
        Platform[targetMethod].overloads[i].implementation = function () {
            console.log("class:", ClassName, "target:", targetMethod, " i:", i, arguments)
            // console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()))
        }
    }
}

function main() {
    // 获取证书密码
    Java.perform(hook_key_store_load)
    // ssl pinning验证
    // Java.perform(hook_ssl)
}

setImmediate(main)

```

```
root@nickmyb:~/Downloads/soul# 7z x soul_3_65_0.apk
# 证书一般都是.p12
root@nickmyb:~/Downloads/soul# tree -NCfhl | grep -i p12

# 获取包名 cn.soulapp.android
root@nickmyb:~# frida-ps -H 192.168.2.10:8888 | grep soul

root@nickmyb:~/Codes/frida-agent-example# frida -H 192.168.2.10:8888 -f cn.soulapp.android -l agent/hook_key_store_load_and_ssl.js --no-pause
# 发现如下日志 KeyStore.load2: android.content.res.AssetManager$AssetInputStream@2e02d34 }%2R+\OSsjpP!w%X

# Proxy - SSL Proxying Settings - Client Certificates - Add - Import P12 - 填入hook到的密码(}%2R+\OSsjpP!w%X) - Host: * Port: * - OK

```

## 6. 基于HOOK的花样抓包和利用

- [SSL PINNING](https://github.com/WooyunDota/DroidSSLUnpinning)
- [某加密到牙齿的APP数据加密分析](https://bbs.pediy.com/thread-260422.htm)
- [保存Bitmap图片](https://www.jb51.net/article/158635.htm)
- Frida.registerClass
    - [1](https://www.securify.nl/blog/android-frida-hooking-disabling-flagsecure)
    - [2](https://bbs.pediy.com/thread-260032.htm)
- okhttp
    - [1](https://mp.weixin.qq.com/s?src=11&timestamp=1611629364&ver=2851&signature=yzCH7ydJMqnj4iYgKsPM1l2PT1vjd1iKOD12F2VdonR8ymcC7A2zMPb-ZKZ5DRlTTrokhUlBD*ODy0hFEU39-avbA97*aTT8m3DukrkrP*Gk2QaxwztxP0eEDoHYNmBL&new=1)
    - [2](https://mp.weixin.qq.com/s?src=11&timestamp=1611629364&ver=2851&signature=yzCH7ydJMqnj4iYgKsPM1l2PT1vjd1iKOD12F2VdonSQTldTj*Oxi6KVlAXRBuvNe20tjYfM2T9dT9ZXJeJ6jO*3CjI6Za4H6j6R9cqqe8mcMKEbJdKzvbCfDA7vJYAr&new=1)
    - [3](https://mp.weixin.qq.com/s?src=11&timestamp=1611629364&ver=2851&signature=yzCH7ydJMqnj4iYgKsPM1l2PT1vjd1iKOD12F2VdonTjvWrI4ORUDdaZ4KmGqHC1j1hFrbQf8OWs-90Zqg3k0nqd8TL07e9oomVPpc5pJC98qbcgMLwW1FdurNpjFlQe&new=1)
- 观点:
    1. 我就是系统
    2. 降维打击
    3. 离数据越近越好

Android开发角度思考: (Android展示图片) -> ImageView -> BitmapFactory -> decodeByteArray(返回值就是Bitmap) -> 保存图片 -> Runnable

完整项目在 6_fulao2.js

```
function hook_BitmapFactory() {
    console.log('start hook_BitmapFactory')

    var class_BitmapFactory_name = 'android.graphics.BitmapFactory'
    var class_BitmapFactory = Java.use(class_BitmapFactory_name)

    var class_File_name = 'java.io.File'
    var class_File = Java.use(class_File_name)

    var class_FileOutputStream_name = 'java.io.FileOutputStream'
    var class_FileOutputStream = Java.use(class_FileOutputStream_name)

    var class_CompressFormat_name = 'android.graphics.Bitmap$CompressFormat'
    var class_CompressFormat = Java.use(class_CompressFormat_name)

    var class_Thread_name = 'java.lang.Thread'
    var class_Thread = Java.use(class_Thread_name)

    class_BitmapFactory.decodeByteArray.overload('[B', 'int', 'int', 'android.graphics.BitmapFactory$Options').implementation = function(data, offset, length, opts) {
        var t = class_Thread.currentThread()
        console.log('currentThread =', t.getName())

        var ret = this.decodeByteArray(data, offset, length, opts)
        console.log('data =', data)
        console.log('offset =', offset)
        console.log('length =', length)
        console.log('opts =', opts)
        console.log('ret =', ret)

        var image_name = '/sdcard/Download/' + String(ret) + '.jpeg'
        // console.log('image_name =', image_name)
        var file = class_File.$new(image_name)
        var file_os = class_FileOutputStream.$new(file)
        // console.log('file_os.getFD() =', file_os.getFD())
        ret.compress(class_CompressFormat.JPEG.value, 100, file_os)
        file_os.flush()
        file_os.close()

        return ret
    }
}


function hook_BitmapFactory_Runnable() {
    console.log('start hook_BitmapFactory_Runnable')

    var class_BitmapFactory_name = 'android.graphics.BitmapFactory'
    var class_BitmapFactory = Java.use(class_BitmapFactory_name)

    var class_File_name = 'java.io.File'
    var class_File = Java.use(class_File_name)

    var class_FileOutputStream_name = 'java.io.FileOutputStream'
    var class_FileOutputStream = Java.use(class_FileOutputStream_name)

    var class_CompressFormat_name = 'android.graphics.Bitmap$CompressFormat'
    var class_CompressFormat = Java.use(class_CompressFormat_name)

    var class_Runnable_name = 'java.lang.Runnable'
    var class_Runnable = Java.use(class_Runnable_name)

    var class_Log_name = 'android.util.Log'
    var class_Log = Java.use(class_Log_name)

    var class_Throwable_name = 'java.lang.Throwable'
    var class_Throwable = Java.use(class_Throwable_name)

    var class_String_name = 'java.lang.String'
    var class_String = Java.use(class_String_name)

    var class_Charset_name = 'java.nio.charset.Charset'
    var class_Charset = Java.use(class_Charset_name)

    var class_Thread_name = 'java.lang.Thread'
    var class_Thread = Java.use(class_Thread_name)

    var class_DumpImage = Java.registerClass({
        name: 'com.nickmyb.Fulao2DumpImage',
        implements: [class_Runnable],
        fields: {
            bm: "android.graphics.Bitmap"
        },
        methods: {
            $init: [
                {
                    // returnType: 'void',
                    argumentTypes: ["android.graphics.Bitmap",],
                    implementation: function(bm) {
                        this.bm.value = bm
                        console.log('com.nickmyb.Fulao2DumpImage.$init(' + bm + ')')
                    }
                }
            ],
            run: [
                {
                    returnType: 'void',
                    implementation: function () {
                        try {
                            console.log('com.nickmyb.Fulao2DumpImage.run() start')
                            var t = class_Thread.currentThread()
                            console.log('currentThread =', t.getName())
                            var image_name = '/sdcard/Download/' + String(this.bm.value) + '.jpeg'
                            var image_file = class_File.$new(image_name)
                            var image_file_os = class_FileOutputStream.$new(image_file)
                            this.bm.value.compress(class_CompressFormat.JPEG.value, 100, image_file_os)
                            image_file_os.flush()
                            image_file_os.close()

                            var log_name = '/sdcard/Download/' + String(this.bm.value) + '.log'
                            var log_file = class_File.$new(log_name)
                            var log_file_os = class_FileOutputStream.$new(log_file)
                            var traceback = class_Log.getStackTraceString(class_Throwable.$new())
                            var str_traceback = class_String.$new(traceback)
                            var l = str_traceback.getBytes(class_Charset.forName("UTF-8"))
                            log_file_os.write(l)
                            log_file_os.flush()
                            log_file_os.close()
                            console.log('com.nickmyb.Fulao2DumpImage.run() end')
                        } catch (e) {
                            console.log("com.nickmyb.Fulao2DumpImage.run() raise Exception =", e)
                        }
                    }
                }
            ]
        },
    })

    class_BitmapFactory.decodeByteArray.overload('[B', 'int', 'int', 'android.graphics.BitmapFactory$Options').implementation = function(data, offset, length, opts) {
        var ret = this.decodeByteArray(data, offset, length, opts)
        console.log('data =', data)
        console.log('offset =', offset)
        console.log('length =', length)
        console.log('opts =', opts)
        console.log('ret =', ret)

        var dump_image = class_DumpImage.$new(ret)
        // dump_image.run()

        var t = class_Thread.$new(dump_image)
        t.start()
        t.join()

        return ret
    }
}


function main() {
    // Java.perform(hook_BitmapFactory)
    Java.perform(hook_BitmapFactory_Runnable)
}


setImmediate(main)
```

## 7. Java Socket逆向思路

- [实用FRIDA进阶：内存漫游、hook anywhere、抓包](https://www.anquanke.com/post/id/197657)
- [burp破解版](https://blog.csdn.net/u014549283/article/details/81248886)
- [Brida](https://github.com/federicodotta/Brida)

### 7.1 burpsuite

#### 7.1.1 burpsuite安装

- burp
    1. java -jar burploader.jar
    2. run - 复制key - next - Manual activation - 复制request - 粘贴response - next - finish
    3. 后续使用burp - java -jar burploader.jar - run
    4. Extender - BApp Store - Brida - Install
    5. pip install pyro4
    6. npm install frida-compile@9.3.0: 安装老一点的版本新版本没有-x选项
    7. Brida - python path: /root/.pyenv/shims/python3.8 - frida-compile path: /root/Applications/burpsuite/node_modules/.bin/frida-compile - Frida JS file folder: /root/Codes/frida-agent-example/agent - Load default JS files: /root/Codes/frida-agent-example/agent - Application ID: com.android.settings

#### 7.1.2 burpsuite使用

- burp
    1. Proxy - Intercept - intercept is off
    2. Proxy - Options - Proxy Listeners - All interfaces
    3. Proxy - Options - Proxy Listeners - export CA certificate - Export - Certificate in DER format - xxx.der
    4. openssl x509 -inform DER -in capture.der -out capture.pem
    5. adb push capture.pem /sdcard/Download
    6. 设置 - 安全性和位置信息 - 加密与凭据 - 从存储设备安装 - 选择pem证书 - 安装用户证书 - 拷贝到根证书
    7. Postern - Https
    8. Filter

### 7.2 Nethunter手机观察

- htop / nethogs
- netstat -tunlp

### 7.3 路由器观察

同上

### 7.4 objection add $init

```
# objection 1.8.4
vim ~/.pyenv/versions/3.8.0/lib/python3.8/site-packages/objection/agent.js

# line 9211
    r.forEach(function(r) {
->
    r.concat(["$init"]).forEach(function(r) {
```

### 7.5 我就是系统

- java.io.File
- inputstream


## 8. 制作路由器来抓包

### 8.1 制作路由器

- 制作路由器
    1. lsusb查看插入的网卡是否被支持(是否显示网卡)
    2. nm-connection-editor
    3. Ethernet -> + -> Wi-Fi -> Create -> SSID(任意), Mode(Hotspot), IPV4 Settings设置(192.168.x.1, 24, 192.168.x.1) -> Save
    4. 注意: 网卡需要供电比较稳定,虚拟机重启后可能需要删除Wi-Fi重建
    5. 配合Wireshshark抓包

### 8.2 Socket Hook

- 相关链接
    - [某聊天app的音视频通话逆向](https://bbs.pediy.com/thread-260656.htm)

hook_socket.js

```
function hook_socket() {
    var libc_so_name = 'libc.so'
    var function_recvfrom_name = 'recvfrom'
    var function_sendto_name = 'sendto'
    var function_recvfrom_address = Module.findExportByName(libc_so_name, function_recvfrom_name)
    var function_sendto_address = Module.findExportByName(libc_so_name, function_sendto_name)

    console.log('function_recvfrom_address =', function_recvfrom_address)
    console.log('function_sendto_address =', function_sendto_address)

    Interceptor.attach(function_recvfrom_address, {
        onEnter: function(args) {
            console.log('recvfrom arg1 =', hexdump(ptr(args[1])))
        },
        onLeave: function(ret) {}
    })

    Interceptor.attach(function_sendto_address, {
        onEnter: function(args) {
            console.log('sendto arg1 =', hexdump(ptr(args[1])))
        },
        onLeave: function(ret) {}
    })
}

function main() {
    hook_socket()
}

setImmediate(main)

```


## 9. [Brida](https://github.com/federicodotta/Brida)

- Brida -> Hooks and functions
- 源码很不错,但实际直接使用也太卡了
